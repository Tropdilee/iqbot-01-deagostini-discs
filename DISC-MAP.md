# I-Qbot:01 CD-ROM 1–4: disc map and firmware analysis

I-Qbot:01 is the Dutch edition of De Agostini's **I-Droid01**, designed by RoboTech srl (Pisa). Nothing was flashed or written to any device while making this map. Everything here comes from reading the disc images.

## How the robot is built

Only the **Brain & Vision** module runs Linux. Every other board is a small Freescale microcontroller running bare-metal firmware. All of them hang off one I2C bus.

| Board | I2C addr | CPU | Firmware on the discs | Controls |
|---|---|---|---|---|
| Brain & Vision | 40 | Freescale i.MX1 (MC9328MXL, ARM920T), 16 MB SDRAM, 16 MB NOR flash | Linux 2.4.26 + `IDROID` | Webcam, Bluetooth (owns the BT module once installed), user programs, the I2C master |
| Head | 41 | 68HC08 (flash @ 0x0860) | `HEAD10IT`, `HEAD11JP` | Pan/tilt motors, eye/ear LEDs, touch sensor, sound direction ("SoundFollower") |
| Base | 42 | HCS08 (flash @ 0xC000) | `BASE10IT`, `BASE11IT`, `BASE12JP` | Wheel motors (PID, encoders), base up/down, 3 ultrasonic sensors |
| Voice / Motherboard | 43 | 68HC08 (MC68HC908AP64) | `MTB-10IT`, `MTB-11IT` (none for the voice board) | LCD, buttons, battery, voice; Bluetooth host when there is no Brain & Vision |
| Arms | 44 | 68HC08 | `ARMS10IT` | Left/right arm motors, lights, temperature, GPIO/analog inputs, tools |
| Hand ("Pliers") | 45 | HCS08 | `HAND10IT` (CD-4 only) | Gripper open/close with force setting |
| URC (IR remote) | 46 | HCS08 | `URC-10IT`, `URC-11IT` (CD-4 only) | IR learning and sending |

- **Bluetooth module:** National Semiconductor LMX9830, a serial-port module. The Brain drives it on `/dev/ttyS1` at 9600 or 921600 baud.
- **Camera:** PixelPlus PO2030 CMOS sensor. The robot's menu shows it as "SP2030". It is connected to the i.MX1's camera interface, and the Linux driver is `id01cam.o`, which provides `/dev/video0`.
- **Suffixes:** in names like `HEAD10IT`, `10` means version 1.0. `IT`/`JP` is probably the edition (Italy/Japan). That last part is a guess.

## What's on each disc

### CD-1 (2007): first build stage, PC/phone control

There is no Linux and no firmware on this disc.
- `software/pc-setup.exe`, `mobile-setup.exe`: PC Control and Mobile Control apps. Both drive the robot over Bluetooth through the motherboard.
- `components/Idroidcomm.exe`: installer for the Bluetooth communication component.
- `software/linux (unsupported)/`: Linux build of PC Control, plus the phone app (J2ME `.jar`).
- `pdf/`: chip datasheets, a useful hardware reference:
  - MC9328MXL (i.MX1)
  - MC68HC908AP64 and addendum
  - MC9S08GB60
  - LMX9830 (Bluetooth)
  - K4S283233E (SDRAM)
  - S29GL-M (NOR flash)
  - Bluetooth guide and compatible phones list
- `video/*.swf`: 18 assembly videos (Flash).

### CD-2 (2007): Brain & Vision install, Visual C-like editor

- **`software/brainVision/`: first Brain & Vision firmware release**
  - `zImage`: Linux kernel
  - `cramfs.img`: root filesystem
  - `jffs2.img`
  - `blob.uue`, `blob-bootloader.txt`: bootloader
  - `brain_config.exe`, `brain_update.exe`, `brain_debug.exe`: flashing and terminal tools
  - `brain_config.log`: a real flashing session, very useful
- **`software/Source/`: GPL source code**
  - `linux-2.4.26-idroid`
  - `blob-2.0.5-pre2-idroid`
  - `busybox-0.60.0-idroid`
  - `tinycc-idroid`
- **`software/Linux_Unsupported/`**
  - `brain_linux.tar.gz`: Linux versions of the brain tools, plus a minicom profile and a README
  - PC Control, Mobile Control and Visual C-like for Linux
- **`visual_setup.exe`:** Visual C-like editor for Windows.
- **Example programs:** `esempi Visual C-like/` (2 examples).
- **Manuals:** `pdf/`.
- **`video/`:** 15 videos.

### CD-3 (2008): updates, Java library

- `software/BrainVision/`: second firmware release (new kernel build and new `IDROID`).
- `java-library.zip`, `java-binaries.zip`, `java-eclipse-projects.zip`, `java-demo.zip`: Java Bluetooth protocol library, with full source.
- `doc/`: Javadoc for that library.
- `pdf/i2c-protocol.html`, `pdf/Bluetooth-Protocol.html`: protocol specifications in Dutch.
- `pdf/Menu_Display_BV.pdf`: map of the robot's LCD menu.
- Newer PC Control (1.4.3), Mobile Control (1.7.4) and Visual C-like (1.1.1). The kernel source is included again.

### CD-4 (2008): newest firmware, hand and remote control

This is a Mac-made hybrid disc. The long filenames only exist in its UDF part, so it was extracted from that.
- `software/BrainVision/`: third and newest firmware release. It adds the `HAND10IT` and `URC-10/11IT` firmware.
- `java-labs.zip`: Java examples, for example `ArmsControl` and `TestArms`.
- `Programmeervoorbeelden/`: 10 C-like example programs.
- Newer PC Control (1.5.5), Mobile Control (1.8.16) and Visual C-like (1.2.1).
- Updated protocol docs and Javadoc.

## Where things are

Paths starting with `/` are inside the root filesystem image `cramfs.img`. It starts with 512 bytes of padding and has the standard little-endian cramfs magic at offset 512.

### Linux system image (Brain & Vision)

- **Kernel**
  - Version: Linux `2.4.26-vrs1-mx1ads`, gcc 3.3.2
  - CD-2: build #113 from 9 Nov 2006
  - CD-3/4: build #4 from 21 Mar 2007
  - Command line: `console=ttyMX0,115200n8 root=/dev/mtdblock2`
- **Root filesystem:** `cramfs.img` (read-only, about 3.9 MB). It contains:
  - glibc 2.3.2 and BusyBox
  - TinyCC (`tcc`), so the robot compiles user C programs itself
  - libjpeg and kernel modules
- **`jffs2.img`:** the same root filesystem as a writable JFFS2 image. None of the flashing tools use it.
- **Bootloader:** `blob.uue`, which is blob 2.0.5-pre2 "for I-Droid01 Brain&Vision", 32,632 bytes when decoded. `blob-bootloader.txt` is the i.MX1 serial-bootstrap script that initializes flash and SDRAM and loads blob into RAM.
- **Boot order**
  1. `rcS`
  2. `rc.rofilesys` (mount the user disk)
  3. `rc.modules`: `mx1ads-pwm`, `i2c-dbmx1`, `id01cam`, `rd`, `usbdmonitor`
  4. `rc.local`
  5. `/idroid/IDROID start`. This is skipped if `/mnt/disk/.nostart` exists. If `/mnt/disk/.start` exists, that script runs instead.

**Flash layout** (16 MB NOR at 0x10000000). It comes from `drivers/mtd/maps/mx1-amd-flash.c` in the kernel source and from CD-2's `brain_config.log`:

| mtd | Name | Offset | Size | Contents |
|---|---|---|---|---|
| 0 | bootloader | 0x000000 | 128 KB | blob |
| 1 | kernel | 0x020000 | 896 KB | zImage |
| 2 | rootdisk | 0x100000 | 6 MB | cramfs.img (root) |
| 3 | userdisk | 0x700000 | 9 MB | JFFS2 `/mnt/disk`: programs, pictures, IR codes, `idroid.ini` |

### Microcontroller firmware (motors, arms, head, hand, remote)

Location: `/idroid/firmware/*.s19` inside `cramfs.img` (Motorola S-records). All checksums are valid, and each file's embedded ID string matches its filename. They were built with CodeWarrior and SofTec. There is **no source** for these on the discs.

The robot's own `IDROID` program flashes them over I2C:
1. Write to register 170 (`BOOT_REG`) to make the board enter its bootloader.
2. Program the new firmware.
3. Verify it.

The command is `IDROID update|backup|verify mtb|head|arms|base <file>`. It refuses to run on low battery.

### Control code

- **`/idroid/IDROID`** (ARM binary, stripped, no source): the main control program. It handles:
  - all motor, arm, head and hand commands over `/dev/i2c-0`
  - Bluetooth: the LMX9830 on `/dev/ttyS1`, with video streaming, file transfer and register access
  - the webcam: `/dev/video0`, color tracking and JPEG pictures
  - voice, IR, the menu and user programs

  It also works as a command-line tool, for example `IDROID arm left up`, `base move`, `head pan`, `bt addr`, `cmos get`, `ver`.
- **User C API:** headers in `/idroid/include/robot/*.h` (`arms.h`, `base.h`, `head.h`, `hand.h`, `brain.h`, and others) and the runtime `/lib/libc-like.so`. `/idroid/compile_up` builds programs on the robot with `tcc`.
- **Webcam driver, with source:** `drivers/media/video/id01cam.c` inside `software/Source/linux-2.4.26-idroid.tar.bz2` (CD-2, CD-3 and CD-4). It was written by RoboTech and is based on MontaVista's `mx1ads-cam.c`.
- **I2C driver:** `i2c-dbmx1.o`. The PWM driver is `mx1ads-pwm.o`.
- **Bluetooth protocol, PC/phone side, with source:** `src/communication/` inside `software/java-library.zip` (CD-3 and CD-4). It covers chunk protocol, internal register access, video streaming and file transfer.
- **Protocol specs** (Dutch, on CD-3 and CD-4):
  - `pdf/i2c-protocol.html`: register map of every board
  - `pdf/bluetooth-protocol.html`: Bluetooth packet types

### USB connection

- **How it appears:** with the robot running, the Brain & Vision USB port shows up as USB ID `19f8:1001` ("RoboTech srl I-Droid 01"). It is a 3 MB FAT drive labeled **"ID-01 Pics"**.
- **What's on it:** this is not the flash. `/etc/hotplug/usbd.agent` builds a RAM disk on connect and copies `/mnt/disk/pictures/*.jpg` onto it.
- **Deleting photos:** when the cable is unplugged, the robot deletes every picture that is missing from the drive. Deleting a photo on the PC therefore deletes it on the robot.
- **Old USB chip:** the port is USB 1.1 (full speed). On some modern USB 3 ports it fails to enumerate (`error -71`). Plugging it in again, or going through a USB 2.0 hub, helps.

## Firmware versions on the discs

| Item | CD-2 | CD-3 | CD-4 |
|---|---|---|---|
| zImage | build #113 (2006-11-09) | build #4 (2007-03-21) | same as CD-3 |
| cramfs.img MD5 | `1128215a…` | `096e0c7f…` | `ee86eaba…` |
| IDROID MD5 | `58010724…` | `dd9d8762…` | `56a012d7…` (newest) |
| MCU firmware | ARMS, BASE×3, HEAD×2, MTB×2 | same | + HAND10IT, URC-10IT, URC-11IT |
| blob | same on all | | |

Apart from `IDROID` and the extra CD-4 firmware files, the three root filesystems are identical.

## How flashing works (reference only, nothing was done)

- **Full install:** `brain_config.exe` (or `brain_linux/brain_config` on Linux).
  1. Put a jumper on the B&V board and connect a serial cable.
  2. The i.MX1 boots from UART at 38400 baud and receives `blob-bootloader.txt` and blob.
  3. Blob is written to flash.
  4. Remove the jumper.
  5. Run `usbddisk kernel` + `flash kernel`, then `usbddisk ramdisk` + `flash ramdisk`. In each case the image is copied to the board, which appears as a USB drive.
- **Update only:** `brain_update.exe` does just the kernel and ramdisk steps through blob. No jumper is needed.
- **Microcontroller firmware:** the `IDROID update` command, as described above.
- **Before ever flashing**
  1. Read the installed versions in the robot's **Diagnostic** menu (for example `HEAD10IT`, `MTB-10IT`, `BRN-xx`).
  2. Back up first: blob has a `dump` command, and IDROID has a `backup` command.
  3. Make sure the battery is full.

## Not found on the discs

- Source code for `IDROID`, `libc-like.so`, and the microcontroller firmware.
- Firmware for the voice board.
- Firmware for the Bluetooth module itself. The LMX9830 has its own ROM.
