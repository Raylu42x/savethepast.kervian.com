# How to Setup our Prototype

You need a Raspberry Pi zero 2 w and a computer for setup. 

Will get updated for more troubleshooting

## Flash an sd card with RPi os
Download RPi imager from [Raspberry Pi Imager Site](https://www.raspberrypi.com/software/) and open it. Plug your sd card in and flash it with Raspberry Pi os 64 lite. Set it up with ssh and wifi.

## Setup rclone
install rclone:
```bash
sudo apt update
sudo apt install rclone -y
```
configure it:
```bash
rclone config
```
select new remote and follow instructions for your cloud service. when done use ctrl+c to exit

## Create Directories
Conect to the pi with ssh or with a moniter and paste:
```bash
mkdir -p ~/autocopy/devices
mkdir -p ~/autocopy/logs
```

## Create the worker script
```bash
nano ~/autocopy/import_worker.sh
```
and paste this inside:
```bash
#!/bin/bash
# import_worker.sh
# Run as user pi. Mounts partitions for a device, copies files, uploads to OneDrive, unmounts.

set -eu

# Config
BASE_DIR="$HOME/autocopy"
DEVICES_DIR="$BASE_DIR/devices"
LOG_DIR="$BASE_DIR/logs"
RCLONE_REMOTE="onedrive"    # change here if your remote has different name
DASHBOARD_URL="http://localhost:8080"  # if you have a dashboard; harmless if not

# Device is single argument: the kernel name passed by udev (e.g. sda or sdb)
DEVICE_KERNEL="$1"
if [[ -z "$DEVICE_KERNEL" ]]; then
  echo "No device specified. Usage: $0 sda"
  exit 1
fi

TIMESTAMP=$(date +"%Y%m%d-%H%M%S")
LOGFILE="$LOG_DIR/${DEVICE_KERNEL}_$TIMESTAMP.log"
exec > >(tee -a "$LOGFILE") 2>&1

echo "=== START import for device $DEVICE_KERNEL at $(date) ==="

# Lock so concurrent events for same device don't race
LOCKDIR="/tmp/autocopy_lock_${DEVICE_KERNEL}"
if ! mkdir "$LOCKDIR" 2>/dev/null; then
  echo "Another job is running for $DEVICE_KERNEL — exiting."
  exit 0
fi
trap 'rm -rf "$LOCKDIR"; echo "Lock removed";' EXIT

# Resolve full device (e.g. sda); we want all partitions on that device
DEV_PREFIX="/dev/$DEVICE_KERNEL"
echo "Device kernel: $DEVICE_KERNEL"

# Find partitions for this device (e.g. sda1, sda2). Use lsblk.
PARTS=$(lsblk -ln -o NAME "/dev/$DEVICE_KERNEL" | tail -n +2 || true)
# If no partitions listed, try the device itself (some devices have no partitions)
if [[ -z "$PARTS" ]]; then
  PARTS="$DEVICE_KERNEL"
fi

echo "Partitions: $PARTS"

# Determine device label for naming: prefer first partition label, else kernel name
FIRST_PART=""
for p in $PARTS; do
  FIRST_PART="$p"
  break
done

LABEL=$(lsblk -no LABEL "/dev/$FIRST_PART" 2>/dev/null || true)
if [[ -z "$LABEL" || "$LABEL" == " " ]]; then
  LABEL="$DEVICE_KERNEL"
fi
# sanitize label (remove spaces and slashes)
SANITIZED_LABEL=$(echo "$LABEL" | tr ' /' '__')

# Determine the final local target name; we must ensure uniqueness if remote exists
TARGET_BASE="$DEVICES_DIR/$SANITIZED_LABEL"

# Function to check remote existence and pick unique name (C3)
pick_remote_name() {
  local base="$1"
  # check remote folder existence
  if rclone lsd "${RCLONE_REMOTE}:/autocopy/${base}" >/dev/null 2>&1; then
    # choose next suffix
    local i=2
    while rclone lsd "${RCLONE_REMOTE}:/autocopy/${base}-${i}" >/dev/null 2>&1; do
      i=$((i+1))
    done
    echo "${base}-${i}"
  else
    echo "$base"
  fi
}

REMOTE_NAME=$(pick_remote_name "$SANITIZED_LABEL")
echo "Local label: $SANITIZED_LABEL  -> remote folder: $REMOTE_NAME"

# Create target local folder (made by pi user)
mkdir -p "$TARGET_BASE"

# Copy rules: exclude common junk/system files and folders.
RSYNC_EXCLUDES=(
  --exclude='System Volume Information/'
  --exclude='FOUND.*'
  --exclude='$RECYCLE.BIN/'
  --exclude='Recycler/'
  --exclude='Thumbs.db'
  --exclude='desktop.ini'
  --exclude='.DS_Store'
  --exclude='.Trashes'
  --exclude='/.Spotlight-V100/'
  --exclude='/.fseventsd/'
  --exclude='__MACOSX/'
  --exclude='lost+found/'
  --exclude='pagefile.sys'
  --exclude='hiberfil.sys'
  --exclude='*.tmp'
)

# iterate partitions
for part in $PARTS; do
  DEVPATH="/dev/$part"
  echo "Processing partition $DEVPATH"

  # Check if it's already mounted. If so, use existing mountpoint
  MOUNT_POINT=$(lsblk -no MOUNTPOINT "$DEVPATH" 2>/dev/null || true)
  if [[ -z "$MOUNT_POINT" ]]; then
    echo "Mounting $DEVPATH..."
    MOUNT_OUTPUT=$(udisksctl mount -b "$DEVPATH" 2>&1) || {
      echo "Mount failed for $DEVPATH: $MOUNT_OUTPUT"
      continue
    }
    MOUNT_POINT=$(echo "$MOUNT_OUTPUT" | grep -o "/media/[^ ]*" || true)
    if [[ -z "$MOUNT_POINT" ]]; then
      echo "Could not parse mount point for $DEVPATH. Raw output:"
      echo "$MOUNT_OUTPUT"
      continue
    fi
  else
    echo "$DEVPATH already mounted at $MOUNT_POINT"
  fi

  echo "Copying from $MOUNT_POINT to $TARGET_BASE (preserve structure)..."
  # Use rsync preserving structure; trailing slashes to copy contents into target dir
  rsync -a --info=progress2 "${RSYNC_EXCLUDES[@]}" "$MOUNT_POINT"/ "$TARGET_BASE"/ || {
    echo "rsync returned non-zero for $DEVPATH (continuing)"
  }

  # After copy of this partition we'll unmount it (we mount per partition)
  echo "Unmounting $DEVPATH..."
  udisksctl unmount -b "$DEVPATH" >/dev/null || echo "Warning: could not unmount $DEVPATH"
  udisksctl power-off -b "$DEVPATH" >/dev/null || true
done

# Ensure .notes.txt exists with simple tree (do not overwrite existing)
NOTES_FILE="$TARGET_BASE/.notes.txt"
if [[ ! -f "$NOTES_FILE" ]]; then
  echo "Creating notes file at $NOTES_FILE"
  echo "Imported from device: $SANITIZED_LABEL" > "$NOTES_FILE"
  echo "Imported at: $(date)" >> "$NOTES_FILE"
  echo "" >> "$NOTES_FILE"
  echo "Directory listing:" >> "$NOTES_FILE"
  # simple tree: list dirs and files with indentation
  (cd "$TARGET_BASE" && find . -print | sed 's|^\./||' >> "$NOTES_FILE")
fi

# Upload to OneDrive (use picked remote name). Run as pi, rclone must be configured for pi.
REMOTE_DEST="${RCLONE_REMOTE}:/autocopy/${REMOTE_NAME}"
echo "Uploading $TARGET_BASE -> $REMOTE_DEST"
# use conservative retries and a bandwidth cap is optional; feel free to adjust
rclone copy "$TARGET_BASE" "$REMOTE_DEST" --transfers=4 --checkers=8 --retries 3 --low-level-retries 3 --stats-one-line --stats 10s || {
  echo "rclone copy returned non-zero (upload may be incomplete). Check rclone logs."
}

# Dashboard notification (non-fatal)
if command -v curl >/dev/null 2>&1; then
  curl -s "$DASHBOARD_URL/start/$SANITIZED_LABEL" >/dev/null 2>&1 || true
fi

echo "Import complete for $SANITIZED_LABEL at $(date)"
echo "Log file: $LOGFILE"
```
ctrl o enter ctrl x to save and than make it executable:
```bash
chmod +x ~/autocopy/import_worker.sh
```
## Create the root-wrapper udev script
Make and edit the wrapper script:
```bash
sudo nano /usr/local/bin/auto_import_wrapper.sh
```
and paste this:
```bash
#!/bin/bash
# wrapper called by udev; runs worker as user pi
DEVICE_KERNEL="$1"
if [[ -z "$DEVICE_KERNEL" ]]; then
  exit 0
fi

# Delay slightly to let kernel settle & partitions appear
sleep 1

# Run worker as pi (no password) – worker will manage locking and logging
sudo -u pi /home/pi/autocopy/import_worker.sh "$DEVICE_KERNEL" &
```
save than run:
```bash
sudo chmod 755 /usr/local/bin/auto_import_wrapper.sh
sudo chown root:root /usr/local/bin/auto_import_wrapper.sh
```
## Install the udev rule to trigger on any block device
create and edit the file with:
```bash
sudo nano /etc/udev/rules.d/99-autocopy.rules
```
than paste:
```bash
# Trigger for any new block device (sdX) to handle partitions/whole-disk devices
KERNEL=="sd[b-z]*", SUBSYSTEM=="block", ACTION=="add", RUN+="/usr/local/bin/auto_import_wrapper.sh %k"
```
reload with:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## Test manully
unplug any drives, plug in your drive, wait a second than run:
```bash
tail -F ~/autocopy/logs/*.log
```
