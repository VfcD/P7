# Arch Linux : Decrypt system drive with USB Key and Keyfile

https://wiki.archlinux.de/title/dm-crypt#System_per_USB-Stick_entschl.C3.BCsseln

Scenario:

    arch linux, 1 hdd for root, 1 hdd for home, UEFI and GRUB.

USB Stick with 2 Partitions

	/dev/sdb1	ext4		1MB	small partition for keyfile
	/dev/sdb2	ext4 | fat	rest	remaining space for data

	udev rule ensures that this stick is always named /dev/usbkey /dev/usbkey1 and /dev/usbkey2

1. Prepare USB Srick: create partitions, make filesystems, create labels

  	  lsblk 
	--> assume stick is /dev/sdb
	
	commands for **fdisk /dev/sdb**:

        o
        n
        p
        Enter (default 1)
        Enter
        +1M
        n
        p
        Enter (default 2)
        Enter
        Enter
        w

        mkfs.ext4 /dev/sdb1
        mkfs.ext4 /dev/sdb2

Label the USB stick depends on which filesystem you want:
check: https://wiki.ubuntuusers.de/Labels/

	e2label /dev/sdb1 USBKEY
	e2label /dev/sdb2 USBDATA

2. Create udev rule for the usb stick
https://wiki.archlinux.de/title/Einbindung_von_USB-Ger%C3%A4ten#Udev-Regel_erstellen

Show serial number of usb stick:

	udevadm info -a -p `udevadm info -q path -n /dev/sdb` | grep ATTRS{serial}

check product name:

	udevadm info -a -p `udevadm info -q path -n /dev/sdb` | grep ATTRS{product}
	
Create udev rule file in **/etc/udev/rules.d/**:

	touch /etc/udev/rules.d/50-myusb.rules

Use for ATTRS{serial} the determined serial number from the above steps.

Add to **/etc/udev/rules.d/50-myusb.rules** (SYMLINK is the devicename):
	
	SUBSYSTEMS=="usb", ATTRS{serial}=="14AB0000000096", KERNEL=="sd*", SYMLINK+="usbkey%n"

Reload udev rules:

	udevadm control --reload-rules

Now your USB stick is known as /dev/usbkey, /dev/usbkey1, /dev/usbkey2
You can check this with:

	ls /dev/usbkey*

3. Write Keyfile to /dev/sdb1 and add the keyfile to a channel for luks decrypt

        mkdir /mnt/usbkey
        mount /dev/usbkey1	/mnt/usbkey

        dd if=/dev/urandom of=/mnt/usbkey/archkey bs=512 count=4

	Enable keyfile decryption for your system drive /dev/sda2:

        cryptsetup luksAddKey /dev/sda2 /mnt/usbkey/archkey 
	
	Add the udev rule to **/etc/mkinitcpio.conf**:

        FILES="/etc/udev/rules.d/50-myusb.rules"
	
	Check HOOKS="..." contains **block** before **encrypt**.

	Create new:
	
        mkinitcpio -p linux

4. Adjust GRUB to use the keyfile on the USB stick

	Edit **/etc/default/grub**:

		vim /etc/default/grub

	Append to GRUB_CMDLINE_LINUX="...":

		cryptkey=/dev/<partition2>:<fstype>:<path>

	E.g.: (notice that You have to use your specified name in udev rule. **NOT /dev/sdb1 !**)
	
		GRUB_CMDLINE_LINUX=" ... cryptkey=/dev/usbkey1:ext4:archkey"		
