# RPI Touch Driver

User-mode driver for the touch controller used in several 7" HDMI capacitive touch displays.

The driver supports the USB touchscreen controller identified as:

* **USB Vendor ID:** `0x0eef`
* **USB Product ID:** `0x0005`
* **Manufacturer:** `RPI_TOUCH`
* **Product:** `By ZH851`
* **USB HID:** 1.10
* **Display resolution:** 800x480

## Tested Hardware

This driver was originally developed for 7" capacitive touch HDMI displays commonly sold for Raspberry Pi and similar single-board computers.

Examples:

* 7" Capacitive Touch Screen HDMI TFT LCD
* Displays using the `RPI_TOUCH / By ZH851` USB touch controller

The USB controller can be identified with:

```bash
lsusb -d 0eef:0005
```

Expected output:

```text
ID 0eef:0005 D-WAV Scientific Co., Ltd By ZH851
```

> Note: The USB VID `0eef` is assigned to D-WAV Scientific Co., Ltd, although the display/controller may have been manufactured or sold by another company.

## Tested Operating Systems

The driver has been tested successfully on:

| Operating system | Version     | Result    |
| ---------------- | ----------- | --------- |
| Ubuntu           | 24.04.4 LTS | ✅ Working |

The original driver was developed in 2015 for Raspberry Pi systems. There is nothing Raspberry Pi-specific about the userspace driver itself; it requires a Linux kernel with `uinput` support.

## The Problem

When the touchscreen USB cable is connected, Linux detects the device correctly:

```text
usb ...: New USB device found, idVendor=0eef, idProduct=0005
usb ...: Product: By ZH851
usb ...: Manufacturer: RPI_TOUCH
```

However, the standard Linux `hid-generic` driver does not expose the touchscreen as a normal input device.

Instead, it creates a `hidraw` device:

```text
/dev/hidrawX
```

but no corresponding:

```text
/dev/input/eventX
```

As a result, desktop environments and applications cannot use the touchscreen normally.

### Example

Before starting this driver, the device may appear as:

```text
hid-generic ... hiddev0,hidraw3
```

but it does not appear in:

```bash
cat /proc/bus/input/devices
```

as a touchscreen input device.

## Why This Device Is Unusual

The controller has several unusual characteristics:

* `idVendor=0x0eef` is assigned to D-WAV Scientific Co., Ltd.
* The HID report descriptor is incorrect.
* Touch coordinates are transferred as 16-bit big-endian values.
* The device sends a proprietary 25-byte touch report.
* Some original Raspberry Pi distributions reportedly included modified kernel code to support the first touch.

## Touch Protocol

Touch events consist of 25 bytes.

Example:

```text
aa 01 03 1b 01 d2 bb 03 01 68 02 cc 00 5d 01 ef
01 5f 01 fe 00 fb 02 37 cc
```

The packet format is:

| Offset | Description                        |
| -----: | ---------------------------------- |
|      0 | Start byte (`aa`)                  |
|      1 | Any touch (`0=off`, `1=on`)        |
|    2-3 | First touch X                      |
|    4-5 | First touch Y                      |
|      6 | Multi-touch start (`bb`)           |
|      7 | Bitmask for all touches (bits 0-4) |
|    8-9 | Second touch X                     |
|  10-11 | Second touch Y                     |
|  12-13 | Third touch X                      |
|  14-15 | Third touch Y                      |
|  16-17 | Fourth touch X                     |
|  18-19 | Fourth touch Y                     |
|  20-21 | Fifth touch X                      |
|  22-23 | Fifth touch Y                      |
|     24 | End byte (`cc` or `00`)            |

Coordinates are transmitted as 16-bit big-endian values.

## How It Works

This is a userspace driver.

It reads the raw HID reports from the touchscreen and injects them back into the Linux input subsystem using `uinput`.

```text
USB Touchscreen
      |
      v
0eef:0005
      |
      v
/dev/hidrawX
      |
      v
rpi_touch_driver
      |
      v
Linux uinput
      |
      v
/dev/input/eventX
      |
      v
Desktop / Applications
```

After the driver is started, the virtual device should appear as:

```text
RPI_TOUCH_uinput
```

For example:

```text
Handlers=mouse1 event13
```

## Requirements

* Linux
* `uinput` kernel support
* USB touchscreen with VID `0eef` / PID `0005`
* Build tools
* `libudev` development files

On Debian/Ubuntu systems:

```bash
sudo apt install build-essential libudev-dev
```

Make sure the `uinput` module is available:

```bash
sudo modprobe uinput
```

## Building

Clone the repository and build:

```bash
git clone <repository-url>
cd rpi_touch_driver

make
```

This creates:

```text
rpi_touch_driver
```

## Testing Without Installation

The driver can be tested directly from the source directory:

```bash
sudo ./rpi_touch_driver
```

With the touchscreen connected, check:

```bash
ls -l /dev/input/
```

A new input device should appear, for example:

```text
/dev/input/event13
```

Check the device:

```bash
cat /proc/bus/input/devices
```

You should see:

```text
N: Name="RPI_TOUCH_uinput"
```

## Testing Touch Events

Install `evtest`:

```bash
sudo apt install evtest
```

Run:

```bash
sudo evtest
```

Select the `RPI_TOUCH_uinput` device.

The device should report events such as:

```text
EV_ABS
ABS_X
ABS_Y
BTN_TOUCH
ABS_MT_POSITION_X
ABS_MT_POSITION_Y
ABS_MT_TRACKING_ID
```

For an 800x480 display, the expected coordinate ranges are:

```text
ABS_X: 0..800
ABS_Y: 0..480
```

## Installation

Install the driver:

```bash
sudo make install
```

The executable is installed as:

```text
/usr/local/bin/rpi_touch_driver
```

## systemd

On systems using systemd:

```bash
sudo make systemd-install
```

The repository contains:

```text
rpi-touch-driver.service
```

which starts:

```text
/usr/local/bin/rpi_touch_driver
```

You can check the service with:

```bash
systemctl status rpi-touch-driver
```

To enable it at boot:

```bash
sudo systemctl enable rpi-touch-driver
```

To start it immediately:

```bash
sudo systemctl start rpi-touch-driver
```

Check the generated input device:

```bash
cat /proc/bus/input/devices
```

## X11 Calibration

If calibration is required under X11, install:

```text
99-rpi-touch.conf
```

to:

```text
/etc/X11/xorg.conf.d/99-rpi-touch.conf
```

The default configuration is intended for an 800x480 display:

```text
Option "Resolution" "800 0 480 0"
```

The calibration option is disabled by default:

```text
#Option "Calibration" "1988 56 143 1936"
```

If calibration is necessary, uncomment the option and replace the calibration constants with values obtained from your calibration procedure.

> Calibration configuration may depend on the display orientation and the X11 setup.

## Troubleshooting

### USB device is detected but touchscreen does not work

Check:

```bash
lsusb -d 0eef:0005
```

Then:

```bash
ls -l /dev/hidraw*
```

If the touchscreen creates a `/dev/hidrawX` device but does not create a corresponding `/dev/input/eventX`, try:

```bash
sudo ./rpi_touch_driver
```

Then check:

```bash
cat /proc/bus/input/devices
```

You should see:

```text
RPI_TOUCH_uinput
```

### Check raw HID data

The device can also be inspected directly:

```bash
sudo xxd -c 25 /dev/hidrawX
```

Touching the screen should produce 25-byte reports beginning with `aa` and normally ending with `cc`.

Example:

```text
aa 01 ... bb ... cc
```

### No `/dev/input/eventX`

Make sure `uinput` is available:

```bash
sudo modprobe uinput
```

Then restart the driver.

## Known Working Configuration

Tested on:

```text
Hardware:
    7" HDMI capacitive touchscreen
    800x480

USB:
    VID: 0x0eef
    PID: 0x0005
    Manufacturer: RPI_TOUCH
    Product: By ZH851

Operating system:
    Ubuntu 24.04.4 LTS

Kernel:
    Linux kernel with uinput support

Result:
    USB device detected
    /dev/hidrawX created
    rpi_touch_driver successfully creates RPI_TOUCH_uinput
    /dev/input/eventX created
    Touch events verified with evtest
```

## License

Copyright (c) 2015 Bjarne Steinsbo.

See `LICENSE` for the license terms.

Code and inspiration from:

* `thiemonge.org/getting-started-with-uinput`
* CyanogenMod userspace touchscreen driver for Cypress CTMA395

## Original References

Examples of displays using this controller:

* Waveshare 7" HDMI LCD
* Eleduino 7" HDMI touch display

This repository is based on the original work by Bjarne Steinsbo and adds documentation and testing information for modern Linux systems.
