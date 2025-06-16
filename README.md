# IO-Modem
Connect to dialup modems over VoIP using SIP, no modem hardware required.

https://www.aon.com/cyber-solutions/aon_cyber_labs/introducing-IO-Modem-a-software-sip-modem/

## Changes in this fork

 - Increased data rates up to full 56k (tested with Cisco 2951 with PVDM2 digital modems and clock synced to GPS using [icE1usb](https://osmocom.org/projects/e1-t1-adapter/wiki/IcE1usb) at the other end, direct SIP between IO-Modem and Cisco)
 - Highly improved connection stability (connections lasting days instead of minutes)
 - Audio output of modem tones using the PJSIP audio output
 - Anonymous calls without credentials
 - RTP- and SIP-ports randomized, allowing multiple instances on the same system
 - `ATX3` (no dialtone required) is the default
 - Running as non-root directly supported (and encouraged!)
 - Various bug fixes

## Building
You'll need Linux and a working 32-bit development environment (gcc -m32 needs to work, Debian-based systems can install: libc6-dev-i386 gcc-multilib), along with PJSIP's dependencies (OpenSSL).  Then run 'make' from the top-level directory.

## How it Works
Traditional “controller-based” modems generally used a microcontroller and a DSP to handle all aspects of modem communication on the device itself.  Later, so-called “Winmodems” were introduced that allowed for field-programmable DSPs and moved the controller and other functionality into software running on the host PC.  This was followed by “pure software” modems that moved DSP functionality to the host as well.  The physical hardware of these softmodems was only used to connect to the phone network, and all processing was done in software. 

IO-Modem replaces a softmodem’s physical hardware with a SIP stack.  Instead of passing audio to and from the software DSP over an analog phone line, audio travels via the RTP (or SRTP) media streams of a SIP VoIP call.   

## Usage
The repository contains two applications: 

slmodemd – A stripped down and patched version of Debian’s sl-modem-daemon package.  All kernel driver code has been replaced with socket-based communication, allowing external applications to manage audio streams. 

IO-Modem – External application that interfaces with slmodemd to manage SIP calls and their associated audio streams.

socat.sh - Script that invokes Socat, connecting 2 modems together and transferring data between them via TCP relay (see [Testing](#testing)).

After they have been built, you can configure SIP account information in the SIP_LOGIN environment variable for calls over a SIP proxy:

    # export SIP_LOGIN=username:password@sip.example.com
Next, run slmodemd, passing the path to IO-Modem in the -e option.  Use -d<level> for debug logging. 

    # ./slmodemd/slmodemd -d9 -e ./IO-Modem
    SmartLink Soft Modem: version 2.9.11 Oct 28 2021 16:51:30 
    symbolic link `/dev/ttySL0' -> `/dev/pts/3' created. 
    modem `slamr0' created. TTY is `/dev/pts/3' 
    Use `/dev/ttySL0' as modem device, Ctrl+C for termination.

In another terminal, connect to the newly created serial device at 115200 bps: 

    # screen /dev/ttySL0 115200

You can now interact with this terminal (almost) as you would with a normal modem using standard AT commands.  A similar modem’s manual provides a more complete list. 

To successfully connect, you might need to manually select a modulation and data rate.  In our testing, V.32bis (14.4kbps) and below might be more reliable, though V.34 (up to 33.6kbps) and V.90 (up to 56k) connections are usually successful.  For example, the following command selects V.32bis with a data rate of 4800 – 9600 bps.  Refer to [the manual](./doc/ST56ATCommands.pdf) for further details. 

    at+ms=132,0,4800,9600 
    OK

Finally, dial the number of the target system.  Below shows a connection to the NIST atomic clock: 

    atd303-494-4774 
    CONNECT 9600 
    National Institute of Standards and Technology 
    Telephone Time Service, Generator 1b 
    Enter the question mark character for HELP 
                            D  L 
     MJD  YR MO DA HH MM SS ST S UT1 msADV         <OTM> 
    59515 21-10-28 21:40:18 11 0 -.1 045.0 UTC(NIST) * 
    59515 21-10-28 21:40:19 11 0 -.1 045.0 UTC(NIST) * 
    59515 21-10-28 21:40:20 11 0 -.1 045.0 UTC(NIST) * 
    59515 21-10-28 21:40:21 11 0 -.1 045.0 UTC(NIST) * 
    59515 21-10-28 21:40:22 11 0 -.1 045.0 UTC(NIST) * 
    59515 21-10-28 21:40:23 11 0 -.1 045.0 UTC(NIST) *

If you want to initiate a direct call to a SIP endpoint without credentials, use `ATDTendpoint@sip.domain`:

    ATDT1234@192.168.0.1
    CONNECT 37333
    Login:

## Testing
Install multipurpose relay Socat and Minicom.

Run slmodemd from 2 terminals, passing the path to socat.sh in the -e option and specifying different modem devices.

    # ./slmodemd/slmodemd -d2 -e ./socat.sh /dev/slamr0

    # ./slmodemd/slmodemd -d2 -e ./socat.sh /dev/slamr1

In 2 other terminals, connect to the newly created serial devices:

    # minicom -D /dev/ttySL0

    # minicom -D /dev/ttySL1

To successfully connect, you might need to manually select a modulation and data rate: 

    at+ms=132,1,,14400 
    OK

Put one modem in answering mode:

    ata

Finally, dial the number of the second system. 2130706433 is a decimal number of localhost IP address 127.0.0.1. If you run second modem on another machine, convert its IP address to a decimal number and dial.

    atd2130706433

Now modems are connected and can interact with each other:

    CONNECT 9600 

To stop data transmission, first escape from on-line mode (+++), then hang up:

    +++
    ath

## Known Issues / Future Work
- Additional logging/error handling is needed 
- The serial interface could be replaced with stdio or a socket, and common AT configuration options could be exposed as command line options 
- There is currently no support for receiving calls in IO-Modem.


Copyright 2021 Aon plc
