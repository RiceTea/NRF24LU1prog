NRF24LU1prog
============

An NRF24LU1+ SPI programmer for Arduino compatible boards.

Program bricked, bootloader-less or code protected [NRF24LU1+][1] via SPI.  
Works with 16kB and 32kB variants of the chip.


---
## BACKGROUND

I wanted some NRF24LU1+ boards, but instead of building my own ones, I bought
a couple of them from those famous far east eBay traders. Less than 10$, including
shipping, seemed quite reasonable.

All of those boards came with proprietary FW, without bootloader and SPI access
locked. As I didn't manage to find any readily available solution to reprogram them,
I began to develop my own emergency programmer...

---
The original NRF24LU1+ chips come with a preprogrammed USB bootloader, residing in
the topmost 2kB of the flash memory of the "MainBlock" and is enabled by default.  
As soon as you reprogram the chip, you should make sure to not overwrite that area
and provide a mechanism to enter the bootloader again, e.g. by either restoring
the original reset vector address or implementing your own code, capable of jumping
to the bootloader's address.

Even with that missing, one could still reprogram the chip via the SPI interface, but
for good reasons, it is possible to disable the flash memory SPI access. This can be
done by writing a 0x00 to address 0x23 of the "InfoPage" memory, a special configuration
memory. There's even a handy SPI command for this: RDISMB.

Virtually all of those far east traders' NRF24LU1+ boards come with that memory lock.


The only way to reset that, is to erase the complete flash memory ("MainBlock").  
After that, but only until the next reset, PROG pin or power cycle, you can write and read
the "MainBlock" flash memory.

To completely reset that access restriction, one needs to set the value of "InfoPage's"
memory address 0x23 to 0xff, which can only be done by erasing the complete "InfoPage".

    WARNING:
    Erasing the "InfoPage" will brick the NRF24LU1+,
    at least, without creating a backup of it.

If SPI memory access is acitve, one cannot read or backup the "InfoPage", hence the complete
sequence to regain access to the NRF24LU1+ is:

  1. enable flash write (SPI command "WREN")
  2. erase complete "MainBlock" (SPI command "ERASE ALL")
  3. upload a FW that can dump the "InfoPage" contents
  4. erase the complete "InfoPage" ("ERASE ALL" with INFEN set in SFR "FSR")
  5. reprogram the "InfoPage", but with addresses 0x22/0x23 set to 0xff
  6. reprogram the bootloader

Notice that you cannot just flash the 32kB bootloader into the 16kB version of the chip.


---
## NEWS

### STATUS:

  BOARD, all main functionalities implemented:

    [X] serial comm (PC)
    [X] serial buffer dump
    [X] serial buffer write
    [X] SPI comm (NRF)
    [X] NRF24 mem read
    [X] NRF24 mem prog
    [ ] NRF24 mem verify
    [X] NRF24 erase
    [X] NRF24 FSR read
    [X] NRF24 flash operation wait
    [X] automatic pin selections for different boards
    [ ] ...

  PC, :
    
    [ ] HEX file read
    [ ] BIN file read
    [ ] serial comm
    [ ] check connection to board (all behave differently)
    [ ] ...

  NRF, pending:

    [ ] InfoPage dumper firmware


### TODO:

  - function for "while( nrfReadFSR() & NRF_FSR_RDYN )..."
  - verify memory (compare with buffer)
  - erase page (maybe...)
  - ...

---
## REQUIREMENTS

  - a 3.3V "Arduino-compatible" board with SPI functionality (see below)
  - [PlatformIO][2], [CLI][3] or [IDE][4]
  - [Python][5] interpreter
  - a handful of wires


### BOARDS

#### BUILT-IN

  NRF24LU1prog should [tm] run on a lot of 3.3V Arduino-compatible boards.  
  Currently implemented are:

  - [ChipKIT Max32][20]
  - [Teensy LC][21]

#### ADDING YOUR OWN BOARD

  ...


---
## HW-SETUP

### CONNECTIONS

  To program an NRF24LU1+ via [SPI][6], at least 5 pins are required:

    - MOSI
    - MISO
    - SCLK
    - CS
    - PROG

  In very rare cases, it might be necessary to additionally wire the RESET pin, e.g.
  when the NRF24 is powered externally and can't be turned on or off.
  But that is usually [tm] not required.

  The NRF24LU1 has an integrated voltage regulator, so it can either be used with
  a 5V or a 3V3 power supply.  
  Notice: EITHER, not BOTH:
  
    5V   VBUS  *highly recommended*
    3V3  VDD

  NOTE 1:
  
    The logic pins, including the SPI connections require 3.3V!

  NOTE 2: 

    Usually [tm], MOSI (NRF) is connected to MOSI (board) and MISO to MISO.
    If your board has different names for the SPI connection, use your brain, e.g.:
    
       board | NRF
      -------+------
        DOUT | MOSI
         DIN | MISO

  ...
  
  The pins and connections can be specified in 'config.h'.  
  Notice that there are multiple sections and make sure you are editing the
  pins related to your board.

    // config.h
    
    #define PIN_PROG     77
    #define PIN_RESET    76
    #define PIN_CS       75
    

#### DEFAULT PINS FOR CHIPKIT MAX32

    NRF24 | CHIPKIT MAX32
    ------+----------------------
     MISO | J13-1  (DIN)
     MOSI | J13-4  (DOUT)
     SCLK | J13-3  (SS)
          |
       CS | 75     (std IO)
     PROG | 77     (std IO)
    RESET | 76     (std IO)
          |
     VBUS | JP6, 5V0 (top right (when USB left))
      GND | J13-6, or see labels

  NOTE:
  
    Jumpers JP3 and JP4 set to MASTER position.


#### DEFAULT PINS FOR TEENSY LC

    NRF24 | TEENSY LC
    ------+----------------------
     MISO | MISO0  (12)
     MOSI | MOSI0  (11)
     SCLK | SCK0   (13)
          |
       CS | CS0    (10)
     PROG | 9      (std IO)
    RESET | 8      (std IO)
          |
     VBUS | Vin 5V (near the USB)
      GND | GND    (near the USB)

  NOTE:
  
    CS0 is still manually toggled in the code


---
## COMPILE & INSTALL

### PLATFORMIO

  Beside the many ways and platforms, described in the the official [installation instruction][8],
  the easiest way to install PlatformIO is doing that via [PIP][10]:

    pip install -U platformio

  Pip is usually [tm] available via your system's package manager.  
  E.g. for Ubuntu, it can be installed via
  
    sudo apt-get install python-pip 


### BOARD

  Inside the 'board' directory, type
  
    platformio run
  
  to compile or
  
    platformio run --target upload
  
  to compile and upload the firmware.
  
  You might wish to edit the targets (boards) in the 'platformio.ini' file:

    [env:mega_pic32]
    platform = microchippic32
    board = mega_pic32
    framework = arduino
    build_flags = -DBOARD_MAX32
    
    ;[env:teensylc]
    ;platform = teensy
    ;board = teensylc
    ;framework = arduino
    ;build_flags = -DBOARD_TEENSYLC

  Just comment out the stuff you don't need with a leading semicolon.  
  Depending on your selection, PlatformIO will automatically install the proper toolchains, required
  to compile the software for the board you selected.


### PC

  ...


### NRF

  ...


---
## USAGE

  NOTE 1:
  
    Most of those already programmed NRF24LU1s (e.g. those from eBay :-)
    reset the RDISMB/IP bits upon a power, a reset or a PROG pin cycle.
    
    ########################################################################
    You can ONLY read back the NRF24's freshly updated memory, directly
    after programming it and before a reset, PROG pin change or power cycle.
    ########################################################################
    
    The only way to solve that is to write 0x00 to address 0x23 of the InfoPage,
    which is (currently) not supported

  NOTE 2:

    Programming or erasing the InfoPage is (currently) disabled.
    This will brick your NRF24LU1 if the InfoPage cannot be read due to an
    already issued RDISIP command. That is usually the case if you got your
    chips from far east eBay sellers...

  NOTE 3:
    
    Some boards, e.g. the ChipKIT Max32, perform a reset after
    the serial port on the PC was opened and spend some seconds
    in the bootloader first.

  NOTE 4:
  
    Some boards, e.g. the Teensy LC don't come with an external chip for USB
    communication (e.g. FTDI). The communication can only be established after the
    "boot procedure" (lol, I know :'-) is over.


### TERMINAL MODE

#### COMMANDS

  ...

  For the ease of it, all commands can be send/read via a serial UART/VCP terminal.  
  Serial settings are "8N1":

    - 115200 bits/s
    - 8 data bits
    - 1 stop bit

  Implemented, so far:

    CMD DATA DESCRIPTION   
     ?       show help
     0       set NRF memory page address (n*512) to zero
     +       increase NRF memory page address; max is 63
     -       decrease NRF memory page address
     =       show NRF memory page address in decimal
     i       initialize NRF via RESET and PROG pin cycle
     S       show flash memory status (in human readable form)
     s       show flash memory status (FSR register value in HEX)
     W       enable  write or erase of flash memory
     w       disable write or erase of flash memory
     E       erase all; requires W (enable flash memory write/erase)
     r       read memory of current selected NRF memory page and store in buffer
     P       program NRF memory from buffer
     d       dump memory buffer to console (in HEX)
     b  34   write to memory buffer, followed by 34 HEX digits (1B addr + 16B data)


##### (?) HELP

  Print out an overview of commands that are implemented, e.g.:

    USAGE:
      ?  show help menu (this)
      +  increase mem page pointer
      -  decrease mem page pointer
      0  set mem page pointer to zero
      =  show mem page pointer value (in decimal)
      i  RESET and PROG pin cycle
      S  show flash status (human readable)
      s  show flash status (FSR register value in HEX)
      W  enable flash write
      w  disable flash write
      E  erase all (requires W cmd to enable flash write
      r  read memory page from chip
      d  dump mem page in HEX
      P  program NRF memory page from buffer (requires W cmd first)
      b  write to buffer, followed by 36 HEX digits (2B addr + 16B data)

##### (i) INIT; RESET and PROGADJUST NRF24 MEMORY POINTER

  ...

##### (0/+/-/=) ADJUST/SHOW NRF24 MEMORY POINTER

  Adjust or print the value of the NRF's memory page pointer.
  It's value can be 0..63.
  
  Notice that the 16kB variant of the chip can only use 0..31, but that
  has to be controlled by the PC software.

  '+' increases the memory page pointer by one  
  '-' decreases the memory page pointer by one  
  '0' sets the memory page pointer to 0 (zero)  
  '=' prints the current value in decimal (0..63)


##### (s/S) SHOW FLASH MEMORY STATUS

  Command 's' returns the content of the FSR register as a two digits, ASCII HEX value, followed by
  a carriage return and a line feed character, e.g.:

    06\r\n

  The captal 'S' outputs the same information in a human readable format, e.g:

    FSR REGISTER: 0x02
      DBG    HW DBG      OFF
      STP    PRG START   0x0000
      WEN    FLASH WR/ER DISABLED
      RDYN   FLASH READY READY
      INFEN  INFOPAGE    DISABLED
      RDISMB SPI RDLOCK  LOCK OFF
      RDISIP SPI IPLOCK  LOCK ON
      RES    -
    
  ...
  
##### (w/W) ENABLE/DISABLE WRITE/ERASE FLASH MEMORY

  'w' disables write/erase flash memory operations and  
  'W' (a capital 'w') enabled them.


##### (E) ERASE ALL

  'E' (a capital 'e') erases the complete flash memeory.  
  The InfoPage will not be erased or changed in any way...
  
  Notice that this command requires the 'W' command first.


##### (r) READ MEMORY FROM NRF24

  ...

##### (P) PROGRAM NRF MEMORY

  ...
  
##### (d) DUMP MEMORY BUFFER TO CONSOLE

  ...

##### (b) WRITE TO MEMORY BUFFER

  ...
  


### PROGRAMMING VIA PC APPLICATION

  ...


---
## LICENSE

  [CC BY 4.0, Attribution 4.0 International][7]

  You are free to:

  Share — copy and redistribute the material in any medium or format  
  Adapt — remix, transform, and build upon the material for any purpose, even commercially.
  
  The licensor cannot revoke these freedoms as long as you follow the license terms.


---
Have fun  
FMMT666(ASkr)  


[1]:  https://www.nordicsemi.com/eng/Products/2.4GHz-RF/nRF24LU1P
[2]:  http://platformio.org/
[3]:  http://platformio.org/get-started/cli
[4]:  http://platformio.org/platformio-ide
[5]:  https://www.python.org/
[6]:  https://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus
[7]:  https://creativecommons.org/licenses/by/4.0/
[8]:  http://docs.platformio.org/en/stable/installation.html
[9]:  https://en.wikipedia.org/wiki/WTFPL
[10]: https://en.wikipedia.org/wiki/Pip_(package_manager)

[20]: https://reference.digilentinc.com/chipkit_max32/refmanual
[21]: https://www.pjrc.com/teensy/teensyLC.html
