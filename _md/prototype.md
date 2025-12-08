# Our Prototype

## What it Does
Our prototype automaticly transfers files from ANY usb device to the cloud
all you need to do is:
1 plug in a drive
2 wait
3 un plug the drive
and than your files are in whichever cloud storage provider you chose during set up

fetures:
- no human input
- under $30
- adds a notes file with drive structure and file tree
- works with over 70 storage providers
- works with any storage device if you have an adapter
what it does behind the senes:
1 uses udev rules to detect a drive
2 if conected to internet will transfer to cloud, if not connected it will put it in local storage until reconect
3 add a .notes.txt with drive's file structure and notes
4 when conected to internet it will unload it's local storage into the cloud

## What you Need to Make on
you can find everything you need to make it and instructions [Here](savethepast.kervian.com/setup)
