# Printing from Windows over network to a POS58 USB Thermal Receipt Printer using Linux/Raspberry Pi

Based on the demo script in this repo, I have set up a solution to print to the POS-58 from network.
The main reason was that I kept having issues with the printer connected to Windows PC via USB, it would keep losing functionality, print would get stuck in a queue and I would have to fully uninstall and reinstall drivers every time I wanted to use it.
Therefore, a solution was needed which would keep the printer permanently available, grabbing an old unused RasPi to use as a printserver seemed the cleanest.
I have plans to explore using a Pi Zero for this and building it into the printer body itself, but am vary of performance after testing on a gen 1 B. Perhaps a Zero 2W will work.

This has been tested on a Raspberry Pi 1 B, and a Raspberry Pi 2 B+


# Don't use Zijian Linux Driver

The driver is not necessary, as per the direct print demo.

# Linux and CUPS print server Configuration

The configuration guide is written on a basis of Raspberry Pi using a recent release of Raspbian. 
Where possible, I tried to use command line interface, but while CUPS *can* be set up fully over command line, it's much more comfortable in a GUI, so there is a level of expectation to use a web browser.
(if you wish to use VNC in place of raspi connect, make sure to enable it in raspi-config:

```
sudo raspi-config
```

Note that next to no browser will work on a Model 1 B anymore, so in such case, I used lynx to activate remote access to CUPS UI and connected from a PC.

## Initial setup for direct printing.

This mirrors the steps from the demo, and indeed the goal at the end of this part is to be able to print by the demo script.

Add your user to the Linux group "lp" (line printer), otherwise you will get a user 
permission error when trying to print.
the "lpadmin" (printer admin) group will be necessary to set up printer in CUPS for sharing.

```
sudo usermod -a -G lp <myusername>
sudo usermod -a -G lpadmin <myusername>
```

Add a udev rule to allow all users to use a USB device that matches this vendor ID and product ID.
Without this rul, you will get a permissions error.

Example: in `/etc/udev/rules.d` create a file ending in .rules, such as `33-receipt-printer.rules` with the contents:

```
# Set permissions to let anyone use the thermal receipt printer
SUBSYSTEM=="usb", ATTR{idVendor}=="0416", ATTR{idProduct}=="5011", MODE="0666"
```

Then, make sure to run following commands to force udev to reload the rules

```
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## Python setup

Install pyusb to your Python (suggest using virtualenv):

You will need to install Python3 for a modern version of Python (unless already installed). 
pip will complain about installing pyusb in an externally managed system; you can force it to do it.

```
pip install pyusb --break-system-packages
```

Do not worry about breaking system packages. It will want to set up a virtual environment, which makes sense in some cases, but this is just installing functionality to a single-purpose server.

## libusb setup

Lastly, install libusb 1.0 for usb device support.

```
sudo apt install libusb-1.0-0-dev
```
And that's it. You can test the printer:
```
python usb_receipt_printer_demo.py
```

## CUPS setup

First quick step is to enable CUPS (which will already be installed on near any end-user linux) to share printers over the network:

```
cupsctl --share-printers --remote-any
```

Now is the time to connect to the GUI to be able to run a browser. Preinstall Chromium or Firefox will do.
Go to `http://localhost:631/` to open your CUPS UI. Once on click Printers, you will see if any printers are installed.

The POS-58 is not yet installed, so go to Administration instead. This page will be password protected - use your unix username and password e.g. the one you use to log into the OS (Raspberry Pi OS).
If you get a Forbidden page, check that you have added yourself to the "lpadmin" group.

In the Administration, clink on Add Printer. Your printer will almost certainly be marked as "Unknown" under Local Printers. 
Select the "Unknown" printer and click Continue. Give the printer a name and a description, for example "POS-58-Remote". Check "Share This Printer" and click Continue.

Now you will have to select a Make and Model. Of course the actual Make and Model will not be available. You can set up to be a Generic Text-Only printer, or a Raw Queue. For performance and simplicity reasons, I recommend Raw Queue.
Finally click on Add Printer. Now you should see your printer. In theory it is shared, but I have found that using IPP does not work from Windows as the printer does not accept the format sent (even if set up as Generic IPP Everywhere printer, Generic PDF printer etc.).

If you do want to try, you can add it as an IPP device (in Windows, using The printer I want isn't listed option, then selecting Add a printer using an IP address or hostname. Select Device type "IPP Device".)
Hostname is `ipp://<hostname of your raspi>:631/printers/<name of your printer>` e.g. `ipp://printer-pi:631/printers/pos-58-remote`

## Printer sharing over Samba

I have had good results using Samba to share the printer. Advantage being that Windows will use the local driver to send the print job to the queue.
Raspbian does not ship with Samba by default, so start by installing it:

```
sudo apt install samba
sudo systemctl enable smbd nmbd
```

Next, you will need to allow printer sharing and set the protocol used to the newest SMB3 (older protocols will be ignored by current Windows machines).
The configuration lives in `/etc/samba/smb.conf`

What you need to set up:

1. find section `[global]` and put under it:
```
   printing = CUPS
   load printers = yes
   printcap name = cups
   min protocol = SMB3
```

2. find sections `[printers]` and `[print$]` and ***in each of them*** change following lines accordingly:
```
   browseable = yes
   guest ok = yes
```

3. Restart Samba server for changes to apply
```
sudo systemctl restart smbd.service nmbd.service
```
You can also use the reload config command
```
sudo smbcontrol all reload-config
```


Now you can add your printer to Windows as a Samba printer. Again, use The printer I want isn't listed option (unless it *is* auto-discovered and listed), then use Select a shared printer by name
THe name is `\\<hostname of your raspi>\<name of your printer>` e.g. `\\printer-pi\POS-58-Remote`

You will have to pick a printer driver, pick the POS-58 driver. If you haven't previously installed it, do it now.

Note: If you are on Windows 11 22H2, [you might run into issues](https://winaero.com/windows-11-22h2-users-suffer-from-printing-issues/).
What worked for me was the fix from [the Samba Wiki](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Print_Server#Known_issues) (the two reg add commands, to be ran in Windows Powershell)
