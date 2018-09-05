# Walkthrough for flashing, configuring Heads on an x230 (+ ME Neutering)

|Last updated: December 2016| 
|-----------------------|
| *There have been updates to Heads and Heads documentation since this guide was created. Please first review Heads documentation:* http://osresearch.net |

# Required Hardware
* Raspberry Pi 3 Model B
  * micro-USB cable for powering the Raspberry Pi
  * USB keyboard
  * USB mouse
  * HDMI cable + monitor
* female-to-female jumper wires
* Pomona 8-SOIC chip clip
* Thinkpad x230
  * power cable
* (optional) ethernet cable 
* (optional) device with live ethernet port

# Table of Contents

1. [Build Heads](#build-heads)
1. [Install Qubes OS on to-be-flashed x230](#install-qubes-os-on-to-be-flashed-x230)
1. [Set-up Raspberry Pi](#set-up-raspberry-pi)
1. [Install Flashrom](#install-flashrom)
1. [Connect Pomona clip to Raspberry Pi](#connect-pomona-clip-to-raspberry-pi)
1. [Prepare x230 and expose BIOS chip](#prepare-x230-and-expose-bios-chip)
1. [Connect Pomona clip to MX25L3206E chip](#connect-pomona-clip-to-mx25l3206e-chip)
1. [Set up appropriate power for Raspberry Pi and x230 board for chip](#set-up-appropriate-power-for-raspberry-pi-and-x230-board-for-chip)
1. [Backup existing BIOS](#backup-existing-bios)
1. [Flash Heads on an x230](#flash-heads-on-an-x230)
1. [Neutering ME](#neutering-me)
1. [Add Xen to boot](#add-xen-to-boot)
1. [Configuring the TPM](#configuring-the-tpm)
1. [TODO](#todo)
1. Appendix: [Running Qubes](#appendix-running-qubes)

# Build Heads

in dedicated Fedora 23 `heads` qube:

1. increase size of qube to `5120 MB`
2. `sudo dnf install @development-tools bison clang flex m4 zlib zlib-devel perl-Digest perl-Digest-MD5 patch uuid-devel elfutils-libelf-devel`
3. [Download Heads](https://github.com/osresearch/heads)
  - unzip if necessary
4. Read [Running Qubes section](#appendix-running-qubes), as you will probably need to change some aspects of `heads/initrd/bin/start-xen` prior to building Heads in the next step
5. build Heads
  - run `make` in heads folder
6. `x230.rom` created 

If your build runs into errors of the file being too large, edit the makefile following [this comment](https://github.com/osresearch/heads/issues/33#issuecomment-249731810)
 - comment out the line `initrd_bins += initrd/bin/dmsetup` so: `#initrd_bins += initrd/bin/dmsetup`

# Install Qubes OS on to-be-flashed x230

Follow the [official instructions](https://www.qubes-os.org/doc/installation-guide/) for installing Qubes on the to-be-flash x230. Installing Qubes after you have flashed the x230 is [more difficult currently](https://github.com/osresearch/heads/issues/27).

# Set up Raspberry Pi

1. install [Raspbian](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)
2. Enable SPI by Start Menu >  Preferences > Raspberry Pi Configuration, select Interfaces, then select Enabled for SPI. or through command line with `sudo raspi-config`, enable SPI under Advanced and then spidev will be enabled.

# install Flashrom

* following: https://libreboot.org/docs/install/rpi_setup.html

3. `sudo apt-get update`
4. `sudo apt-get install libusb-1.0`
4. download [latest stable version of Flashrom](https://download.flashrom.org/releases/flashrom-0.9.9.tar.bz2) and [its associated signature](https://download.flashrom.org/releases/flashrom-0.9.9.tar.bz2.asc)
5. download coreboot/flashrom developer keys: `gpg --recv-keys 0x45D34CCF6785FC01 0x918F0230023DF00B`
6. verify flashrom `gpg --verify flashrom-0.9.9.tar.bz2.asc flashrom-0.9.9.tar.bz2`
7. extract `tar -xf flashrom-0.9.9.tar.bz2`
6. `cd flashrom`
7. `make`

turn off raspberry pi

# Connect Pomona clip to Raspberry Pi

Looking at the Raspberry Pi with USB ports facing you/down, the expansion headers are on your upper right of the board. Counting the header numbers goes Left-to-Right, then Up-to-Down. So:

|left|right
|---|---
| 1 | 2
| 3 | 4
| 5 | 6
etc.

Following the [Raspberry Pi SPI header instructions](https://www.flashrom.org/RaspberryPi#Connecting_the_flash_chip), connect the jumper wires to the following headers: 19, 21, 23, 24, 25.

Now looking at the Pomona 8-SOIC clip, these wires will go:

| left | right
|---|---
| 19 | 25
| 23 | empty
| empty | 21
| empty (or 17 [if powering from RPi](https://github.com/mfc/flashing-docs/blob/master/walkthrough%20for%20flashing%20heads%20on%20an%20x230.md#set-up-appropriate-power-for-raspberry-pi-and-x230-board-for-chip)) | 24

# Prepare x230 and expose BIOS chip

Before disassembling the x230, we can go into the BIOS to enable a few options we need. We can do this after disasssmblying the x230 as well.

In the x230, go into BIOS (enter on boot, then F1) and go to:
* Network > enable Wake on LAN (Ethernet LAN Option ROM can stay disabled)
* (optional) USB > Always on USB Charge in Off Mode (to power RPi -- this did not work for me but you can try)

Now unplug the x230, remove the battery, and disassemble the thinkpad:

![](https://blog.noq2.net/content/images/corebooting-x230-0001.jpg)
* from https://blog.noq2.net/corebooting-thinkpad-x230.html

![](https://farm9.static.flickr.com/8584/28653973936_9cffb2a34e.jpg)
* from: https://trmm.net/Installing_Heads

![](https://farm8.static.flickr.com/7667/28608155181_fa4a2bfe45.jpg)
* from: https://trmm.net/Installing_Heads

Flip up the retainer and pull blue tab.

Inspect the type of flashchip on the motherboard, the x230 should have a Marconix MX25L3206E (where BIOS is) and MX25L6406E (where ME is).

we are interested in the MX25L3206E, which is "above" the other chip, closer to the screen. It should look like the following:

![](https://i.imgsafe.org/3d5e8d7d20.png)

![](https://i.imgsafe.org/3d5f36f6d9.png)

* from [page 7](http://www.zlgmcu.com/mxic/pdf/NOR_Flash_c/MX25L3206E_DS_EN.pdf) 

# Connect Pomona clip to MX25L3206E chip

Using a magnified glass or the aid of your cellphone "torch/flashlight", confirm the dot on the chip is away from the screen closer to you on your x230. This means ground (GND) is the upper-right connection if you are facing the x230, so the diagram above should be rotated 180 degrees to accurately represent your perspective of the chip).

Attach the Pomona clip to the MX25L3206E chip, with GND to the upper-right.

| left | right
|---|---
| SI/SIO0  | GND
| SCLK | empty
| empty | SO/SIO1
| empty (or VCC [if powering from RPi](https://github.com/mfc/flashing-docs/blob/master/walkthrough%20for%20flashing%20heads%20on%20an%20x230.md#set-up-appropriate-power-for-raspberry-pi-and-x230-board-for-chip)) | CS#

which is:

| left | right
|---|---
| 19 | 25
| 23 | empty
| empty | 21
| empty (or 17 [if powering from RPi](https://github.com/mfc/flashing-docs/blob/master/walkthrough%20for%20flashing%20heads%20on%20an%20x230.md#set-up-appropriate-power-for-raspberry-pi-and-x230-board-for-chip)) | 24

# Set up appropriate power for Raspberry Pi and x230 board for chip

## Power the RPi

While it is [recommended](https://www.bios-mods.com/forum/Thread-Thinkpad-X230-Can-t-read-BIOS-CHIP-using-Raspberry-PI-and-8pin-CLIP?pid=104417#pid104417) to power the Raspberry Pi through to-be-flashed x230, I was unable to do so (the Raspberry Pi did not turn on). I was able to power the Raspberry Pi by connecting it to a wall outlet via the micro-USB cable. You should test out powering the RPi with the Pomona Clip detached from the x230, so that you are just booting the RPi, and you can see whether or not it is receiving adequate power. If on boot there is a yellow lightning bolt in the upper right corner that means that your RPi is not receiving enough power -- try a different (shorter) micro-USB cable, try a wall outlet rather than plugging it into a computer.

## Powering the x230 board 

### If you want to do ME Neutering

If you are interested in [ME Neutering](#neutering-me), you need to power the board through the RPi, the Wake-on-LAN strategy described below will not work.

Ensure that the x230 is off, battery removed, and AC power is **unplugged**. Attach the additional jumper wire between the Pomona Clip and RPi, attach the Clip to the appropriate chip and then power on the RPi.

### If you don't want to do ME Neutering

You should have already gone into the x230 BIOS to enable Wake on LAN. With the x230 battery removed, plug in AC power and connect the x230 to a live ethernet connection via the ethernet cable. "live" just means that the computer (or wall socket) it is connected to is on. The green RJ45 (ethernet) light should turn on, signaling that the board is powered. The computer itself should remain off (no BIOS screen, etc).

Wake-on-LAN is the preferred method to power the board that the chip is on, the alternative is powering the board from the Raspberry Pi itself, this is [not recommended by experts](https://www.coreboot.org/pipermail/coreboot/2016-December/082592.html). However, this Wake-on-LAN strategy relies on ME functionality that will be removed by the [ME Neutering](#neutering-me).

from:
* https://www.bios-mods.com/forum/Thread-REQUEST-Lenovo-Thinkpad-X230-Whitelist-removal?pid=91134#pid91134
* https://www.bios-mods.com/forum/Thread-REQUEST-Lenovo-Thinkpad-X230-Whitelist-removal?pid=91787#pid91787

# Backup existing BIOS

Check that it can be read successfully. If you cannot read the chip and receive an error similar to "no EEPROM Detected" or "0x0 Chip detected" then you may want to ensure you Raspberry Pi has adequate power (the flashing yellow lightning bolt in the upper-right means it is not being provided sufficient power). You can try switching the two pins 19 and 21.

`./flashrom -c "MX25L3206E/MX25L3208E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r backup1.rom`

Run the flashrom command again to make a second dump:

`./flashrom -c "MX25L3206E/MX25L3208E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r backup2.rom`

Now run diff to see if they match:

`diff backup1.rom backup2.rom`

If you receive no output, they match. Alternatively, you can take a `sha512sum`, `sha1sum`, or `md5sum` of the files to confirm they match:

`sha512sum backup*.rom`

You may try and flash Heads now.

# Flash Heads on an x230

`./flashrom -c "MX25L3206E/MX25L3208E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -w /path/to/x230.rom`

When you get the message `Verifying flash... Verified` you have succeeded!

# Neutering ME

Download the [ME Cleaner python script](https://github.com/corna/me_cleaner).

attach Pomona clip to MX25L6406E, which is further from the screen and closer to you.

1. `./flashrom -c "MX25L6406E/MX25L6408E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r me-backup1.rom`
2. `./flashrom -c "MX25L6406E/MX25L6408E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r me-backup2.rom`

Now run diff to see if they match:

`diff me-backup1.rom me-backup2.rom`

If you receive no output, they match. Alternatively, you can take a `sha512sum`, `sha1sum`, or `md5sum` of the files to confirm they match:

`sha512sum me-backup*.rom`

clean a copy

1. `python ./me-cleaner.py me-backup1.rom`
1. rename `me-backup1.rom` to `me-cleaned.rom` for sanity

flash to board

`./flashrom -c "MX25L6406E/MX25L6408E" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -w /path/to/me-cleaned.rom`

* http://hardenedlinux.org/firmware/2016/11/17/neutralize_ME_firmware_on_sandybridge_and_ivybridge.html
* https://github.com/corna/me_cleaner/wiki/How-does-it-work%3F

# Add Xen to boot

https://github.com/osresearch/heads/issues/84#issuecomment-274623405

1. `make xen.intermediate`
2. extract `xen-4.6.3.gz` which should get you a file `xen`
2. copy `xen` to USB and **note the filesystem of the USB** (if NTFS reformat to ext4, Heads can't mount NTFS)
3. plug USB into Heads machine
4. `mkdir /tmp/usb`
5. `mount /dev/sdbX /tmp/usb` change "X" to the number of your USB partition (most likely `/dev/sdb1`)
6. `mount -o rw /dev/sda1 /boot` 
6. `cp /tmp/usb/xen /boot`
8. `umount /tmp/usb`
9. `umount /boot`

# Configuring the TPM

see (https://trmm.net/Installing_Heads#Configuring_the_TPM)

1. `tpm physicalpresence -s`
2. `tpm physicalenable`
3. `tpm physicalsetdeactivated -c`
3. `tpm forceclear` (if you get an error `TPM deactivated`, try [rebooting to fix the state](https://github.com/osresearch/heads/wiki/Installing-Heads#taking-ownership))
4. `tpm physicalenable`
5. `tpm takeclear -pwdo OWNER_PASSWORD` (should be `takeown`, [related bug](https://github.com/osresearch/heads/issues/117)
6. `/bin/sealtotp.sh`
7. scan QR code
8. test with `unsealtotp.sh`

If you are finding the codes do not match, confirm the right time on your machine:

1. `date -u` should give you UTC time, `date` should give you your local time. If it doesn't:
2. `export TZ=TIMEZONE`

[here is a list of timezones](https://uical.uic.edu/ocas/ocwc/american/help/timezone.htm), I had success with the `UCT#` format. also: https://unix.stackexchange.com/questions/71860/correct-use-of-tz-date-and-hwclock

# TODO

1. https://github.com/osresearch/heads/wiki/Installing-Heads#installing-extra-software
1. https://github.com/osresearch/heads/wiki/Installing-Heads#read-only-root
1. https://github.com/osresearch/heads/wiki/Installing-Heads#hashing-the--partition-and-setting-up-dm-verity
1. https://github.com/osresearch/heads/wiki/Installing-Heads#signing-boot

# Appendix: Running Qubes

Things that may need to be modified in the Makefile prior to build (unless you want to edit `/bin/start-xen` every boot...):

* https://trmm.net/Installing_Heads#Adding_your_PGP_key
  * [comment it out of Makefile for now](https://github.com/osresearch/heads/issues/119)
*	change: `--module "${KERNEL} root=/dev/mapper/qubes_dom0-root rhgb" \` to `--module "${KERNEL} root=/dev/mapper/qubes_dom0-root rhgb" \` (see: [ticket](https://github.com/osresearch/heads/issues/110))
* `XEN=/boot/xen` (see [ticket](https://github.com/osresearch/heads/issues/84#issuecomment-274623405))
