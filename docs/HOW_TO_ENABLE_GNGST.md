# Enable `$GNGST` Sentences

Why this sentence is useful? It reports error statistics directly from the GNSS receiver, so we don't have to compute
position errors by ourselves. Position error is generally CEP, measured in meters. Larger values are worse. It is a
circle within which 50% of fixes land. The error is the circle's radius. A larger circle is worse.

## How to enable?

Included in this repo is [HOW_TO_ENABLE_GNGST.pdf](HOW_TO_ENABLE_GNGST.pdf) which is a page from the
[Application-Note-AN0037.pdf](Application-Note-AN0037.pdf) which describes the binary command to send. Expect an `ACK`
to confirm command success. See the logs further below on how it was sent. Continue reading for information on program
setup and quirks of root access.

## How to send this command?

I used a custom [CH340C Userspace Program](https://github.com/AvirukBasak/ch341-reader).

## CH340C Userspace Program Setup

```
git clone https://github.com/AvirukBasak/ch341-reader
cd ch341-reader
pip install pyusb # the only dependency
sudo ./shell.py   # needs root to directly access usb device
```

## GNSS Module Overview

The GNSS module has 3 layers in general.

1. The Antenna      - The large ceramic patch antenna (possibly).
2. SkyTraQ GNSS MCU - processes the GNSS signals, produces NMEA sentences
3. CH340C UART-USB  - Takes NMEA from the MCU and exposes a USB interface for a PC

The SkyTraQ MCU has its own protocol, which is listed in [Application-Note-AN0037.pdf](Application-Note-AN0037.pdf).
One can send these commands over USB (or direct UART if using a companion ESP32/MCU). If done over USB, these go via
the CH340C chip. A laptop or Android phone doesn't talk UART. So, the CH340C chip provides a bridge. It also has a
config protocol (which is irrelevant for end usecases).

## Why is Root necessary?

Generally we rely on a TTY node like `/dev/ttyUSB0` on Linux. Desktop linux ships with a `ch341` driver so when we
connect a device with CH340C over USB, it shows up as `/dev/ttyUSB*`. This hides the USB protcol and exposes a file
we can read or write to. Read acts like UART RX, write like TX. Android doesn't have this driver (even on custom ROMs).
Compiling a driver (ofc, rooted Android) is difficult because of the exact kernel version header files needed (which
IS available on desktop Linux). An app could work, and many support reading from external GNSS (e.g.GPS Connector on
Play Store). But none goes a level deeper and configures the SkyTraQ behind.

Our userspace program (shell.py) takes control over USB (exclusive lock). Root is needed for this purpose. Closing this
program releases the device and immediately makes it available. PyUSB (libusb wrapper for Python) is used for this. The
CH340C implementation itself is ported into Python directly from the Linux kernel's source.

## Issue with `su` and `tsu` on Termux (rooted Android)

On Android, there are 3 ways to run as root, `su -c command`, `sudo command` and `tsu command`. Tsu is NOT sudo, even
though tsu claims to be sudo for compatibility. To install sudo, use `pkg install sudo`. Don't do `pkg install tsu`.
That installs tsu and makes sudo an alias to itself. Tsu does not work on new versions of magisk root. Sudo does.

While su works initially, on Ctrl+C, it returns to the shell instead of stopping the program or letting the program
handle the signal. If Ctrl+C is not received by the program, python will not raise `KeyboardInterrupt`, and the program
will keep printing messages and never release the lock even though user is returned to the shell. The only way to exit
that is to kill the terminal (close Termux). I have not yet understood why this happens.

## Demo log from the program

We enable `$GNGST` on the module. Note that it is set as ephemeral in this demo. However, the SkyTraQ chip likely has
a FLASH to persistently store configs. This can be done using the same command but setting the last byte to 1 (it is 0
in this demo: 64 02 ... 01 **00**). Hence, for persistance, the bytes become:

```
64_02_01_01_03_01_01_01_01_00_00_00_00_01_01
```

The underscores are ignored (they are for readability). See further below for explanation of how the program is used.
This log does not have actual fix data because it was recorded immediately after the receiver was powered on. 

```
>> open
[INFO] ch341.py: CH341 device opened (version 0x31, baud 115200, bulk IN max packet size 32)
[INFO] shell.py: Opened VID=0x1a86 PID=0x7523 baud=115200
[INFO] shell.py: No carrier (DCD not asserted)

1a86:7523> read
[INFO] shell.py: Streaming - press Ctrl-C to exit
0000.00000,E,0,00,0.0,0.0,M,0.0,$GPGGA,000124.000,0000.00000,N,0$GPGGA,000125.000,0000.00000,N,00000.00000,E,0,00,0.0,0.0,M,0.0,M,,0000*6B
$GPGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,1*2D
$GLGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,2*32
$GAGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,3*3E
$GBGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,4*3A
$GIGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,6*33
$GNGLL,0000.00000,N,00000.00000,E,000125.000,V,N*59
$GNRMC,000125.000,V,0000.00000,N,00000.00000,E,000.0,000.0,060180,,,N,V*1B
$GNVTG,000.0,T,,M,000.0,N,000.0,K,N*1C
$GNZDA,000125.000,06,01,1980,00,00*49
$GPGGA,000126.000,0000.00000,N,00000.00000,E,0,00,0.0,0.0,M,0.0,M,,0000*68
$GPGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,1*2D
$GLGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,2*32
$GAGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,3*3E
$GBGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,4*3A
$GIGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,6*33
$GNGLL,0000.00000,N,00000.00000,E,000126.000,V,N*5A
$GNRMC,000126.000,V,0000.00000,N,00000.00000,E,000.0,000.0,060180,,,N,V*18
$GNVTG,000.0,T,,M,000.0,N,000.0,K,N*1C
$GNZDA,000126.000,06,01,1980,00,00*4A
^C
[INFO] shell.py: Interrupted

1a86:7523> write 64_02_01_01_03_01_01_01_01_00_00_00_00_01_00 hex sktrq-px
[INFO] shell.py: Streaming - press Ctrl-C to exit
0000.00000,E,0,00,0.0,0.0,M,0.0,M,,0000*69
$GPGSA,A,1,,,,,,,,,,���d�
$GPGGA,000002.000,0000.00000,N,00000.00000,E,0,00,0.0,0.0,M,0.0,M,,0000*6F
$GPGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,1*2D
$GLGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,2*32
$GAGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,3*3E
$GBGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,4*3A
$GIGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,6*33
$GNGLL,0000.00000,N,00000.00000,E,000002.000,V,N*5D
$GNRMC,000002.000,V,0000.00000,N,00000.00000,E,000.0,000.0,060180,,,N,V*1F
$GNVTG,000.0,T,,M,000.0,N,000.0,K,N*1C
$GNZDA,000002.000,06,01,1980,00,00*4D
$GNGST,000002.000,,,,,,,*55
$GPGGA,000003.000,0000.00000,N,00000.00000,E,0,00,0.0,0.0,M,0.0,M,,0000*6E
$GPGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,1*2D
$GLGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,2*32
$GAGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,3*3E
$GBGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,4*3A
$GIGSA,A,1,,,,,,,,,,,,,0.0,0.0,0.0,6*33
$GNGLL,0000.00000,N,00000.00000,E,000003.000,V,N*5C
$GNRMC,000003.000,V,0000.00000,N,00000.00000,E,000.0,000.0,060180,,,N,V*1E
$GNVTG,000.0,T,,M,000.0,N,000.0,K,N*1C
$GNZDA,000003.000,06,01,1980,00,00*4C
$GNGST,000003.000,,,,,,,*54
^C
[INFO] shell.py: Interrupted
[INFO] shell.py: ACK payload: 64 02

1a86:7523> close
[INFO] ch341.py: CH341 device closed
[INFO] shell.py: Device closed

>> ^C
```

### Important Bits from The Log Above

- Anything starting with `[INFO]` is from our program. Anything starting with a `$` is from the module. Any `>` prompt
  is to give a command to our program.
- First we run `open` after running `sudo ./shell.py`. This gives a `[pid]:[vid]>` prompt.
- Then we run `read`. Press Ctrl+C to stop reading and return to the `[pid]:[vid]>` prompt.
- Next we write a command using `write 64_02_01_01_03_01_01_01_01_00_00_00_00_01_00 hex sktrq-px`.
- This sends raw bytes `64 02 01 ...` to the SkyTraQ chip. The command details are described in
  [HOW_TO_ENABLE_GNGST.pdf](HOW_TO_ENABLE_GNGST.pdf).
- The `write`, `hex` and `sktrq-px` are keywords for the program itself, not the chip. `hex` means raw bytes passed as
  hex (the other option `txt` is not relevant to our purpose).
- The `sktrq-px` tells the `write` to enclose the payload in the start bytes, end bytes, add checksum and add message
  length fields. So, `64 02 01 ...` is the command and `write` turns it into the actual payload.
- Note that on write, we automatically start a read loop. ACK / NACK / failure information will show up after you press
  Ctrl+C when read loop is running.

### Observe
On initial read, there is no `$GNGST`. After the `write` it appears. There is a FLASH on the SkyTraQ chip which can
store configs persistently across power downs. In the demo, we did not write the command to the onboard FLASH. 

Also observe the line `[INFO] shell.py: ACK payload: 64 02`. Our command started with `64 02`. The ACK contains the
same, indicating a success.

