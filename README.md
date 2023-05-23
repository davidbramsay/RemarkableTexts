# Remarkable and Receipt Printer Texting Interface for OS X

![how texting works](https://github.com/dramsay9/RemarkableTexts/blob/main/pic1.png?raw=true)

![an example of what's printed.  May your food arrive in a more timely manner than mine.](https://github.com/dramsay9/RemarkableTexts/blob/main/pic2.png?raw=true)

This is a project by David Ramsay at the MIT Media Lab that allows you to create a secondary workspace for communicating near your main workstation running OS X.  This handles text messaging-- incoming and outgoing texts are printed to the receipt printer with nice formatting (it can support JPEGs as well).  Additionally, incoming texts are stored on a remarkable tablet, and any handwritten text that you write on those PDFs is automatically pulled off the device, OCRed to text, and sent as text messages to the intended recipient.  

The result is an interface that feels a bit like writing letters and monitoring a ticker tape.  It changes the cadence, the barriers, and the intentionality of asynchronous communication.  It also preserves more focused, walled digital spaces that are only designated for working without communication and interruption. 

## Overview

To get this working requires a thermal printer (i.e. the Adafruit Mini Thermal Printer ADA597), a USB to TTL Serial Cable (FTDI FT232), a 5-9V >1.5A Power supply, a remarkable tablet, and (if you want to create a physically separate workspace for communicating) a long USB cable connected to a powered USB hub.  This code has been tested on OS X Big Sur and Monterey. 

Once you've added addresses to your address book in /lib, the script will automatically pull the latest texts from these contacts and generate PDFs to be pushed to the Remarkable Tablet with their names and most recent texts.  These queued PDFs can be found in the local 'TO_REMARKABLE' folder.  Whenever the script boots up, it will pull the most recent text from all of your contacts and any other number that has a PDF pushed to your remarkable, and make sure they are up-to-date with the most recent text you've received.  PDFs in the TO_REMARKABLE folder are queued for pushing to the device.  

The script then iterates through a loop every 15 seconds. On each iteration, it first attempts to create an SSH connection with the Remarkable tablet. If successful, PDFs in the TO_REMARKABLE folder will be pushed to the device, and their contents, their file structure on the Remarkable Tablet, and their most recent modification time on the tablet are saved in a persistent datastore.  

Each iteration, the script will look for updates to those PDFs based on the modification time.  If a modification has been made and no document is currently open for editting, it will pull the modified PDFs off the Remarkable and put them in the 'FROM_REMARKABLE' folder.  These PDFs have handwritten responses on them that need to be sent out to contacts as a text.

Each iteration, the script will check for files in the 'FROM_REMARKABLE' folder, crop them as images to send out via text, OCR them using the google vision API, and attempt to send them via iMessages using applescript.  If this fails, they simply remain in the 'FROM_REMARKABLE' folder until the next iteration.

Finally, every iteration, the chat.db file which updates iMessages is checked for modifications.  If there are new incoming or outgoing texts, these texts get added to the print queue.  When the thermal printer is attached, it will print the new prints in order.  If the message contains no text, it will attempt to print the attachment that goes with it, by marrying the number of attachments in detected in the texts with the most recent files added to the 'Attachments' folder tree buried in Application Support.  If it fails to match (i.e. the number of new files in the Attachments folder and the number of attachments sent are not equal), it simply does not try to print the attachment.  It currently works with JPEGs and thumbnails of files.

## Setup 

For Monterey, the AppleScript to send images via text is different; check lib/textControl and use SEND_IMAGE_LAPTOP instead of SEND_IMAGE.

You should set up your address book in 'lib', mimicking the 'address.pyEXAMPLE' file, and delete that example.

The thermal printer, if setup correctly, will talk over serial.  You need to pass the right serial port to the lib/printer class.  Right now, it defaults to the address my printer registers under ( def __init__(self, serialport='/dev/tty.usbserial-1422110').  You can pass this in from the run.py script when initializing the printer using the serialport variable.  

Run.py is the main entry point for this code and can/should be daemonized.

You need to set up your Remarkable for ssh using an rsa key.  If you have existing id_rsa and id_rsa.pub in ~/.ssh, you can use: `ssh-copy-id root@10.11.99.1`, otherwise need to generate with `ssh-keygen`.  The default address for the remarkable is 10.11.99.1; you should be able to `ssh root@10.11.99.1` without entering a password once you've set this up.  The initial ssh password to get into the remarkable and set up ssh can be found in Settings>Help>Copyrights.  There are lots of good examples of how to get this working online, check remarkable reverse engineering wiki.

NOTE: the remarkable API included here uses OpenSSH's ControlMaster and ControlPath to manage the SSH connections, which gave me a little trouble initially before I realized how the connection was persisting and added a timeout for unexpected disconnects.  Poke around and read about -M and -S flags before editing the code.

You also need to set up the google vision API for this software to work; install google CLI using

```
curl https://sdk.cloud.google.com | bash
gcloud init
gcloud projects create dramsayocrtext #this name must be unique to your project, of all projects ever created in gcloud
gcloud auth login
gcloud config set project dramsayocrtext
gcloud auth application-default login
gcloud auth application-default set-quota-project dramsayocrtext
```

Then enable the API in the google cloud console, and enable billing in the google cloud console ($1.50/1000 images).

If you're doing this for a second time on a second computer, gcloud init will simply allow you to select the project on `gcloud init`, and you can skip to the `gcloud auth application-default login` command.

There are some additional dependencies that can be finicky-- poppler (brew) and pillow and opencv for pdf editing.  Requirements.txt should work but your mileage may vary.
