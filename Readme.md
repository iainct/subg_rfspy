# Pre-Requisites for Building subg_rfspy

[sdcc] (http://sdcc.sourceforge.net/) package is required for this build.

    sudo apt-get install sdcc

# UART on RileyLink

Perform the build. The output file will be stored at output/uart1_alt2_RILEYLINK_US/uart1_alt2_RILEYLINK_US.hex

    make -f Makefile.uart1_alt2

Perform the install:

    make -f Makefile.uart1_alt2 install

UART/SPI pins exposed on cc1110 debug header:

    PIN - SPI_alt2 / UART1_alt2
    P1.4 - CT / SSN
    P1.5 - RT / SCK
    P1.6 - TX / MOSI
    P1.7 - RX / MISO

# Radio Frequency Selection

This code defaults to building firmware that works with US-based pumps. If your
pump model number ends with 'WW' then you need a 'WorldWide' firmware:

    make -f Makefile.uart1_alt2 RADIO_LOCALE=WW

# UART on the WirelessThings ERF stick

Perform the build. The output file will be stored at output/uart0_alt1_SRF_ERF_US/uart0_alt1_SRF_ERF_US.hex

    make -f Makefile.uart0_alt1 BOARD_TYPE=SRF_ERF

Installation is bit more complicated, as you will need to attach connectors to
the SRF pins manually (you can't use the wiring points on the board itself).

This [XRF blog post](http://paulswasteland.blogspot.co.uk/2015/01/building-your-own-firmware-for-ciseco.html)
maps cc-debugger connector to the XRF pin names.

However, the XRF pins are not the same as the SRF pin locations. For that, you
can use the [SRF OpenMicros Data](http://openmicros.org/index.php/articles/88-ciseco-product-documentation/259-srf-technical-data)
to map things to the correct ERF locations.

- SRF Pin 5 - DDATA (also known as DD)
- SRF Pin 6 - DCLOCK (also known as DC)
- SRF Pin 9 - 3.3v (also known as VDD)
- SRF Pin 10 - Ground (also known as GND)
- SRF Pin 15 - Reset

To install the firmware:

    make -f Makefile.uart0_alt1 BOARD_TYPE=SRF_ERF install

# USB on support on TI cc1111 USB stick (CC1111EMK868-915) AKA "Don's Dongle"

    make -f Makefile.usb_ep0 install

Shows up as a serial device on linux.

# CCTL Support

If you have [CCTL](https://github.com/oskarpearson/cctl/tree/24mhz_clock_and_erf_stick_hack)
on your device stick, you can re-program the firmware without requiring the cc-debugger.
To compile firmware that's compatible with CCTL, set the CODE_LOC and CODE_LOC_NAME parameters:

    make -f Makefile.uart0_alt1 BOARD_TYPE=SRF_ERF CODE_LOC=0x400 CODE_LOC_NAME=CCTL

Then, compile the cctl writer program:

    cd /where/you/want/the/cctl/code/to/live
    git clone https://github.com/oskarpearson/cctl.git
    cd cctl
    git checkout 24mhz_clock_and_erf_stick_hack
    cd cctl-prog
    make clean all

Then connect the ERF stick over the serial port (normally /dev/ttyUSB0), and write the firmware:

    ./cctl-prog -d /dev/ttyUSB0 -f /path/to/subg_rfspy/output/uart0_alt1_SRF_ERF_WW_CCTL/uart0_alt1_SRF_ERF_WW_CCTL.hex
    Waiting 10s for bootloader, reset board now

Reset the board by disconnecting the +ve lead, and you should then see:

    ....Bootloader detected
    Erasing, programming and verifying page 1
    ...
    Erasing page 31
    Programming complete


# Yardstick One

**NOTE** Currently the amplifier on the Yardstick One is not yet supported. This
enables standard Yardstick one support only!

The Yardstick (and other RfCat devices) use CCBootloader, which has a
bootloader size of 0x1400.

For USA pumps:

    make -f Makefile.usb_ep0 BOARD_TYPE=YARDSTICK_ONE CODE_LOC=0x1400 CODE_LOC_NAME=CCBOOT

For WorldWide pumps:

    make -f Makefile.usb_ep0 BOARD_TYPE=YARDSTICK_ONE CODE_LOC=0x1400 CODE_LOC_NAME=CCBOOT RADIO_LOCALE=WW

Once you have a compiled firmware, follow the instructions in the section 'CC-Bootloader Devices'

# CC-Bootloader Devices

If you have a cc-bootloader device like the Yardstick One or other RfCat device, you can
overwrite the firmware via serial port, without the CC-Debugger. However, you need
the 'rfcat' code to force the device into serial-port mode.

First, download this file somewhere. You may also need to install the python USB
and serial port libraries:

    $ cd /tmp
    $ git clone FIXME rfcat-tmp
    $ sudo easy_install -Z pyusb pyserial

Now install udev rules, so that you can communicate with the device:

    $ cd rfcat-tmp
    $ sudo cp etc/udev/rules.d/20-rfcat.rules /etc/udev/rules.d
    $ sudo udevadm control --reload-rules

If everything has gone well, once you plug in the Yardstick One, you will see a file called /dev/RFCAT1 or similar

    $ ls -la /dev/RFCAT1
    lrwxrwxrwx 1 root root 15 Jan 28 08:14 /dev/RFCAT1 -> bus/usb/002/101


## Write the firmware

Once you've done the above steps, you can actually write the firmware to the
device:

1. Make sure the USB device is not plugged in.

2. Enable the bootloader by running rfcat:

    $ cd /tmp/rfcat-tmp
    $ ./rfcat --bootloader --force

3. You should see a message saying: "No Dongle Found.  Please insert a RFCAT dongle.""

4. Plug in the Dongle. If you receive this message, you can ignore it:

    'NoneType' object has no attribute 'excepthook', you can ignore it.

5. If all has gone well, you should see a new device, called /dev/RFCAT_BL_C,
  /dev/RFCAT_BL_D, or /dev/RFCAT_BL_YS1

    $ ls -al /dev/RF*
    lrwxrwxrwx 1 root root 7 Jan 28 18:32 /dev/RFCAT_BL_YS1 -> ttyACM0

6. Write the firmware

    $ /tmp/rfcat-tmp/rfcat_bootloader /dev/RFCAT_BL_C erase_all
    $ /tmp/rfcat-tmp/rfcat_bootloader /dev/RFCAT_BL_C download bins/RfCatChronosCCBootloader.hex
    $ /tmp/rfcat-tmp/rfcat_bootloader /dev/RFCAT_BL_C verify bins/RfCatChronosCCBootloader.hex
    $ /tmp/rfcat-tmp/rfcat_bootloader /dev/RFCAT_BL_C run
