# Technical Documentation: Interrupts, Hardware, and Protocols

## Interrupts

### SH4A (DP5, DP6)

| Code    | Function                  | Verified |
|---------|---------------------------|----------|
| `0x40`  | TLB Miss (Read)           | Yes      |
| `0x60`  | TLB Miss (Write)          | Yes      |
| `0x80`  | Initial Page Write        | Yes      |
| `0xA0`  | TLB Prot. Violation (Read)| Yes      |
| `0xC0`  | TLB Prot. Violation (Write)| Yes      |
| `0xE0`  | Address Error (Read)      | Yes      |
| `0x100` | Address Error (Write)     | Yes      |
| `0x400` | TMU0 TUNI0                | Yes      |
| `0x420` | TMU0 TUNI1                | Yes      |
| `0x440` | TMU0 TUNI2                | Yes      |
| `0x560` | ADC_ADI                   | Yes      |
| `0x600` | IRQ0                      | Yes      |
| `0x620` | IRQ1                      | Yes      |
| `0x640` | IRQ2                      | Yes      |
| `0x660` | IRQ3 (SD Media Change??)  | No       |
| `0x900` | SCIFA3                    | No*      |
| `0x9E0` | CEU1                      | No*      |
| `0xA20` | USB0                      | No*      |
| `0xAA0` | RTC_PRI                   | Yes      |
| `0xAC0` | RTC_CUI                   | Yes      |
| `0xB20` | DMAC1B_DEI5               | No*      |
| `0xBE0` | Keypad (KEYSC)            | Yes      |
| `0xC20` | SCIF_SCIF1                | No*      |
| `0xC40` | SCIF_SCIF2                | No*      |
| `0xCC0` | SPU_SPUI0                 | No*      |
| `0xCE0` | SPU_SPUI1                 | No*      |
| `0xD00` | SCIFA4                    | No*      |

_* Interrupt function not verified. This entry is based on the SH7724, which is the chip most closely resembling the chip used by Casio._

### SH3 (DP3)

| Code    | Function                  | Verified |
|---------|---------------------------|----------|
| `0x40`  | TLB Miss (Read)           | Yes      |
| `0x60`  | TLB Miss (Write)          | Yes      |
| `0x80`  | Initial Page Write        | Yes      |
| `0xA0`  | TLB Prot. Violation (Read)| Yes      |
| `0xC0`  | TLB Prot. Violation (Write)| Yes      |
| `0xE0`  | Address Error (Read)      | Yes      |
| `0x100` | Address Error (Write)     | Yes      |
| `0x1C0` | NMI                       | Yes      |
| `0x400` | TMU TUNI0                 | Yes      |
| `0x420` | TMU TUNI1                 | Yes      |
| `0x440` | TMU TUNI2                 | Yes      |
| `0x4A0` | RTC PRI                   | Yes      |
| `0x4C0` | RTC CUI                   | Yes      |
| `0x560` | WDT ITI                   | Yes      |
| `0x600` | IRQ0                      | Yes      |
| `0x620` | IRQ1                      | Yes      |
| `0x640` | IRQ2                      | Yes      |
| `0x660` | IRQ3 (SD Media Change??)  | No       |
| `0x700` | -                         | -        |
| `0x980` | ADC ADI                   | Yes      |
| `0xA20` | -                         | -        |
| `0xE80` | -                         | -        |
| `0xEA0` | -                         | -        |
| `0xF00` | -                         | -        |

---

## Hardware Interfaces

### Keypad (SH4A - DP5, DP6)

The SH4A has a built-in Keyscan IO module used by Ex-word dictionaries. It uses a matrix starting at address `0xA44B0000` to store the currently pressed keys.
```
Bit 0	 Bit 1	 Bit 2	 Bit 3	 Bit 4	 Bit 5	 Bit 6	 Bit 7
Byte 0		 Func 1	 Func 2	 Func 3	 Func 4	 Func 5	 Func 7	 Func 8
Byte 1	 Power							
Byte 2		 Q	 W	 A	 Shift	 Z	 History	 Page up
Byte 3		 Func 6						
Byte 4		 R	 T	 F	 C	 V	 Back	 B
Byte 5		 E	 S	 D	 X	 Zoom	 Jump	 Page down
Byte 6		 O	 K	 M	 Y	 G	 Up	 Left
Byte 7		 U	 I	 J	 N	 H	 Enter	 Down
Byte 8								
Byte 9		 P	 L	 Symbol	 Backspace	 Audio		 Right
```

### LCDC (DP5+ SH4A)

The DP5+ Series use a 538x320 color TFT with 16 bit pixels in 5:6:5 format.

VRAM starts at address 0xAC200000 in external memory and is copied to the LCD display by using DMA transfers using the DMAC (channel 3) address of 0xFE008050.

An external LCDC is used and according to its device identification appears to be a R61523.
Access to the LCDC is done via the memory address 0xB4000000. In order to send a command to the LCDC you first need to clear bit 4 on the 8 bit PORT R register at 0xA405013C and write the the command index to 0xB4000000, afterwards reset bit 4 and read or write the command's parameters (depending on the command sent) from/to 0xB4000000.

Below is some C code that implements a VRAM copy function as well as a function to set the hardware window on the LCDC. It is based on the code found in the disassembled B99 NOR image.

```c
volatile unsigned char  * PORTR  = (unsigned char*)  0xA405013C; // port R IO data register. 8-bits wide
volatile unsigned short * LCDC   = (unsigned short*) 0xB4000000;

volatile unsigned long  * SAR_3  = (unsigned long*)  0xFE008050;
volatile unsigned long  * DAR_3  = (unsigned long*)  0xFE008054;
volatile unsigned long  * TCR_3  = (unsigned long*)  0xFE008058;
volatile unsigned long  * CHCR_3 = (unsigned long*)  0xFE00805C;
volatile unsigned short * DMAOR  = (unsigned short*) 0xFE008060;

void lcdc_write_word(unsigned short val)
{
    *LCDC = val;
    asm("synco");
}

void lcdc_reg_select(unsigned char index)
{
    *PORTR = *PORTR & 0xef; // clears DCX pin on LCDC
    asm("synco");
    *LCDC = (unsigned short)index & 0xff;
    asm("synco");
    *PORTR = *PORTR | 0x10; // resets DCX pin on LCDC
    asm("synco");
}

void lcdc_set_window(short x, short y, short width, short height)
{
    width = (x + width) - 1;
    y += 0x28;
    height = (y + height) - 1;

    lcdc_reg_select(0x2a);
    lcdc_write_word((x >> 8) & 0xFF);
    lcdc_write_word(x & 0xFF);
    lcdc_write_word((width >> 8) & 0xFF);
    lcdc_write_word(width & 0xFF);

    lcdc_reg_select(0x2b);
    lcdc_write_word((y >> 8) & 0xFF);
    lcdc_write_word(y & 0xFF);
    lcdc_write_word((height >> 8) & 0xFF);
    lcdc_write_word(height & 0xFF);
}

void vram_copy(short width, short height)
{
    unsigned long transfer_size;
    transfer_size = ((width*height) >> 4); //((width*height*2) / 32) or ((width*height) / 16)
    lcdc_reg_select(0x2c);
    *TCR_3 = transfer_size;
    *SAR_3 = 0xac200000 & 0x1fffffff;
    *DAR_3 = 0xb4000000 & 0x1fffffff;
    *CHCR_3 = *CHCR_3 & 0xfffffffe;
    *DMAOR = *DMAOR & 0xfffe;
    *CHCR_3 = 0x40101401;
    *DMAOR = *DMAOR | 0x1;
    while ((*CHCR_3 & 0x2) == 0);
}
```



## Add-ons

Add-ons for exword dictionaries support running custom code via the addin=���� option in the infotgt.htm file. Using this it is possible to write custom applications to run on exword dictionaries.


### Custom Apps

These are third party add-ins for exword devices. They can be installed using the exword program included with libexword, just install them as a normal add-in dictionary.

NOR Dumper - Dumps the NOR image for an exword device
Memory Dumper - Dumps the contents the RAM and on-chip memory
Copy nand - Copy all files in the nand to tf card

## CASIO Ex-Word protocol documentation

The protocol used by Casio EX-Word electronic dictionaries is a slightly modified version OBEX. This custom version prefixes each request with a sequence number. The response packet is then preceded by a one byte packet containing the sequence number of the request.

example of a request/response exchange

request:   <seq> <opcode> <packet length> <headers...>
response:  <seq>
           <response code> <packet length> <headers...>

Commands

Connect

This command must be the first one you send and is used to make a
connection to the obex server on the dictionary.

The request/response for the connect string is defined the same as
in standard obex except that in the request the version byte is used
to specify operating mode, there are also an additional three
bytes with the values 0x40, 0x00 and a third byte specifying the
region.

Valid regions:
   Japanese = 0x20
   Korean   = 0x40
   Italian  = 0x48
   Chinese  = 0x60
   Indian   = 0x68
   German   = 0x80
   Spanish  = 0xa0
   French   = 0xc0
   Russian  = 0xe0

Modes:
   TextLoader = <region>
   Library    = <region> - 0xf
   CDLoader   = 0xf0
Disconnect

This command sends a disconnect to the dictionary and shuts down the usb
device. It uses the standard obex disconnect opcode with no additional
headers.
Setpath

Changes the current path on the dictionary.

Each pathname needs to begin with either /_INTERNAL_00 or /_SD_00 which
specify which storage device we are accessing either internal memory or
the currently inserted SD card.

Request:
  Uses the 0x85 (setpath) opcode followed by a name header containing the
  Unicode pathname.
Response:
  Contains no additional headers.
Capacity

Retrieves capacity of currently selected storage device.

Request:
  Uses the 0x83 (get) opcode followed by a name header containing the
  Unicode string "_Cap".
Response:
  The response returns a length header and end of body header.
  The length header contains the length of data sent in the end of body
  header and should contain the value 8. The end of body header contains
  two 4 byte numbers in network byte order with the first representing the
  total capacity and the second one represents amount of space used.
Model

Get model information.

Request:
  Uses the 0x83 (get) opcode followed by a name header containing the
  Unicode string "_Model".
Response:
  The response returns a length header and end of body header.
  The length header contains the length of data sent in the end of body
  header. The end of body header contains two null terminated strings
  containing the model information. So far the first one seems to always
  be the same for all models tested.
List

Return directory information for the currently selected path.

Request:
  Uses the 0x83 (get) opcode followed by a name header containing the
  Unicode string "_List".
Response:
  The response returns a length header and end of body header.
  The length header contains the length of data sent in the end of body
  header. The end of body header contains the directory information as
  an array of the following structure.

  struct directory_entry {
      uint16_t size;  //size of structure
      uint8_t  flags; //file = 0, directory = 1, unicode/longname = 2
      uint8_t  name[]; //name of file/directory
  }
Remove

Removes a file from currently selected path.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_Remove". Following the name header are a length and
  end of body header. The length gives the length of data in the end of
  body header and the end of body header contains a null terminated string
  of the filename to remove.
Response:
  Contains no additional headers.

On DataPlus 5+ devices the name of the file is a utf16 string when in
Text mode.
SdFormat

Formats currently inserted SD Card

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_SdFormat". Following the name header are a length and
  end of body header. The length gives the length of data in the end of
  body header and the end of body header contains only a null character.
Response:
  Contains no additional headers.
Send file

Sends a file to the dictionary.

Request:
  Uses the 0x82 (put) opcode followed by a name, length, and body/end of body
  headers. The name header is a Unicode string containing the name of the
  file being sent, the length header should contain the total size of the
  file and the body/end of body headers contain the actual file data.

  If the file is greater then max packet size it will be split across multiple
  put requests only the first one will include the name and length headers.
Response:
  Contains no additional headers.
Get file

Sends a file to the dictionary.

Request:
  Uses the 0x83 (get) opcode followed by a name header. The name header is
  a Unicode string containing the name of the file being retrieved.
Response:
  The response returns a length header and body/end of body headers.
  The length header contains the length of data sent in the body/end of
  body headers. The body/end of body headers contain the file data
  retrieved.

  If the file is large enough(more then about 4k), it will be split across
  multiple response packets. In this case only the first response contains
  the length header.
AuthChallenge

Sends an authentication request to the dictionary.

This command must be issued in Library mode before many commands suchs as
delete, upload, capacity, etc will work. Somewhat interestingly the List
command does not require authentication and will if issued cause the other
commands requring authentication to work as if you had issued a successful
AuthChallenge command.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_AuthChallenge". Following the name header are a length
  and end of body header. The length should always be 20 and the end of
  body header should contain the the 20 byte authentication key.
Response:
  Contains no additional headers.
AuthInfo

Resets dictionary authentication info stored in authinfo.inf.

When run this command will also delete all dictionaries currently
installed on device.

Request:
  Uses the 0x83 (get) opcode followed by a name header containing the
  Unicode string "_AuthInfo". Following the name header is a 40 byte
  non-standard header (0x70) contaning data used to generate a new
  authkey.
Response:
  The response returns a length header that and end of body header.
  The length header will always be 20 and the end of body header will
  contain the new 20 byte authkey.
UserId

Sets userid of dictionary.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_UserId". Following the name header are a length header
  and end of body header. The length should always be 17 and the end of body
  header contains the new username, this should be zero padded to a length
  of 17.
Response:
  Contains no additional headers.
Unlock

Unlock device for adding/removing add-ons.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_Unlock". Following the name header are a length and
  end of body header. The length should always be 1 and the end of body
  always contains a single null byte.
Response:
  Contains no additional headers.
Lock

Lock device after removing/adding an add-on.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_Lock". Following the name header are a length and
  end of body header. The length should always be 1 and the end of body
  always contains a single null byte.
Response:
  Contains no additional headers.
CName

Specifies name of dictionary.

This comand is used to set the name of the add-on dictionary you wish to
access. It shold be sent immediately after then Unlock comamnd.

Request:
  Uses the 0x82 (put) opcode followed by a name header containing the
  Unicode string "_CName". Following the name header are a length and
  end of body header. The length header contains the length of the
  end of body header and the end of body contains two null terminated
  sjis encoded strings. The first should be the add-on's five character
  ID. The second is the name of the add-on dictionary.
Response:
  Contains no additional headers.
CryptKey

Generates a new CryptKey.

The key returned is not used to directly encrypt the dictionary. Instead it
is used to generate a secondary key that does the actual encryption.

Request:
  Uses the 0x83 (get) opcode followed by a name header containing the
  Unicode string "_CryptKey". Following the name header is a non-standard
  header (0x71) containing a 28 byte block used to generate the encryption
  key.

  The algorithim used to generate the 16 byte key looks like the following:
       b[0] b[1] b[2]  b[3]  b[4]  b[5]  b[6]  b[7]  b[8]  b[9]  b[10] b[11]
    +    0    0  b[16] b[17] b[18] b[19] b[20] b[21] b[22] b[23]    0     0

  The last 4 bytes are b[24] - b[27], however only the first 12 are returned
  in the response header.
Response:
  The response returns a length header that and end of body header.
  The length header will always be 12 and the end of body header will
  contain tthe first 12 bytes of the generated key.

## SDHI Hardware

Both the SH3 and SH4a CPUs use the same SDHI module. In fact this SDHI module appears to be used by ALL SuperH CPUs that support SDHI. Unfortunately the SDHI documentation on Renesas datasheets requires an NDA so there is no official access to the register layout of the SuperH SDHI module, however basic register layout and usage can be found under the drivers/mmc/host/tmio_* files in the linux kernel. These files implement the host driver that is used in SuperH chips.

Below is a list of register information for the SuperH SDHI module


Register Mappings

Base Address
SH3	 A4550000
SH4a	 A4CF0000
Registers
CTL_SD_CMD	 00
CTL_ARG_REG	 04
CTL_STOP_INTERNAL_ACTION	 08
CTL_XFER_BLK_COUNT	 0a
CTL_RESPONSE	 0c
CTL_STATUS	 1c
CTL_STATUS2	 1e
CTL_IRQ_MASK	 20
CTL_SD_CARD_CLK_CTL	 24
CTL_SD_XFER_LEN	 26
CTL_SD_MEM_CARD_OPT	 28
CTL_TRANSACTION_CTL	 34
CTL_DMA_ENABLE	 d8
CTL_RESET_SD	 e0


sys_close (0x38)

Closes an open file.

Prototype

int sys_close(int fd);
Parameters

fd: Valid open file descripter
Returns

Returns 0 on success


sys_create (0x34)

Create a new file or directory.

Prototype

int sys_create(char *path, int flags);
Parameters

path: Full pathname of file/directory to create
flags: Value of 1 will create a file, while a value 5 creates a directory.
Returns

0 - Success
-1 - Path does not exist
-2 - Invalid or no device name specified
-11 - Read only filesystem
-13 - File system entry already exists
-33 - No card in SD card slot

sys_create (0x34)

Create a new file or directory.

Prototype

int sys_create(char *path, int flags);
Parameters

path: Full pathname of file/directory to create
flags: Value of 1 will create a file, while a value 5 creates a directory.
Returns

0 - Success
-1 - Path does not exist
-2 - Invalid or no device name specified
-11 - Read only filesystem
-13 - File system entry already exists
-33 - No card in SD card slot

sys_delete (0x35)

Delete a file or directory.

Prototype

int sys_delete(char *path);
Parameters

path: Full pathname of file/directory to delete
Returns

0 on success, negative value on error

sys_freediskspace (0x33)

Retreive the available space of the specified storage device.

Prototype

int sys_freediskspace(char *device, unsigned long *free);
Parameters

device: Device name (eg. drv0, crd0)
free: Free space on device
Returns

Returns 0 on success

sys_get_filesize (0x3a)

Gets the current size of an open file.

Prototype

int sys_get_filesize(int fd);
Parameters

fd: Valid file desripter
Returns

Returns file size in bytes

sys_open (0x37)

Open an existing file.

The path specified must point to an existing file that was previously created with sys_create.

Prototype

int sys_open(char *path, int flags);
Parameters

path: Full pathname of file to open
flags: open mode of file (Read, Write, or Read/Write)
Returns

Returns file descriptor on success, on error returns negative error code.

sys_read (0x3c)

Read data from an open file.

Prototype

int sys_read(int fd, char *buffer, int size);
Parameters

fd: Valid file desripter
buffer: Buffer to read data into
size: Size of buffer
Returns

Returns the number of bytes that are actually read from the file

sys_sdformat (0x31)

Formats the currently inserted SD card.

Prototype

int sys_sdformat();
Returns

0 on success, negative on error

sys_seek (0x3b)

Seek to a position within a file.

This function is similiar in behaviour to te standard lseek function, except that on success it will always return 0 and it won't seek past the end of a file.

This function only performs a seek for read operations, if you want to write at a specific offset use sys_write2.

Prototype

int sys_seek(int fd, int offset, int whence);
Parameters

fd: Valid file desripter
offset: Offset to seek to
whence: Determines how the offset param is interpreted
Returns

Returns 0 on success

sys_strcmp (0x68)

Performs a lexical comparison on two strings. This behaves the same as the standard C function.

Prototype

int sys_strcmp(char *s1, char *s2);
Parameters

s1: string 1
s2: string 2
Returns

Returns 0 if s1 == s2, -1 if s1 < s2, and 1 if s1 > s2

sys_strdiff (0x66)

Locates the first differing character between two string.

Prototype

unsigned int sys_strdiff(char *s1, char *s2);
Parameters

s1: string 1
s2: string 2
Returns

Returns index of first character that differs between s1 and s2.

sys_strlen (0x67)

Finds the length of a string. This function is the same as the standard C version.

Prototype

unsigned int sys_strlen(char *s1);
Parameters

s1: string 1
Returns

Returns the length of s1 excluding the terminating NULL character.

sys_strrcmp (0x69)

Performs a lexical comparison on two strings, however unlike sys_strcmp it compares the strings from right to left instead of left to right.

Prototype

int sys_strrcmp(char *s1, char *s2);
Parameters

s1: string 1
s2: string 2
Returns

Returns 0 if s1 == s2, -1 if 1 < s2, and 1 if s1 > s2

sys_strtolower (0x16a)

Converts a string to lowercase

Prototype

void sys_strtolower(char *str);
Parameters

str: string to be converted

sys_totaldiskspace (0x32)

Retreive the total space of the specified storage device.

Prototype

int sys_totaldiskspace(char *device, unsigned long *total);
Parameters

device: Device name (eg. drv0, crd0)
total: Total space on device
Returns

Returns 0 on success

sys_write (0x3d)

Write data to an open file.

 This function will always append the data to the end of the file.

Prototype

int sys_write(int fd, char *buffer, int size);
Parameters

fd: Valid file desripter
buffer: Data to be written
size: Size of buffer
Returns

Returns the number of bytes actually written to the file

sys_write2 (0x3e)

Writes data to an open file.

This function differs from sys_write in that it takes a fourth offset that tells the system call where in the file to write the data at.

The offset can not be larger than the size of the file.

Prototype

int sys_write2(int fd, char* buffer, int size, int offset);
Parameters

fd: Valid file desripter
buffer: Data to be written
size: Size of buffer
offset: offset to write data at
Returns

Returns the number of bytes actually written to the file.

## EXWord Syscalls


The syscall interface works by storing the address of the syscall entrypoint at 0x74000000, in order to execute a syscall you first need to retreive the entrypoint address and then load r0 with the correct syscall number.

Registers r4-r7 are used to hold the parameters passed into the syscall and r0 will contain the return value after execution.

Example of making a call to sys_sdformat

_sys_sdformat:
  mov.l @(0xc, pc), r0
  mov.l @(0xe, pc), r1
  mov.l @r1, r1
  jmp @r1
  nop
.align 2
.long 0x31
.long 0x74000000

String syscalls

sys_strdiff - Finds the first differing character in string
sys_strlen - Determines the length of a string
sys_strcmp - Compares two strings
sys_strrcmp - Compares two strings (right to left)
sys_strtolower - Converts string to lowercase

File syscalls

sys_sdformat - Formats inserted SD card
sys_totaldiskspace - Gets total space for storage device
sys_freediskspace - Gets the free space for storage device
sys_create - Create file or directory
sys_delete - Delete file or directory
sys_open - Open a file
sys_close - Close open file descriptor
sys_get_filesize - Get size of a file in bytes
sys_seek - Seeks to offset in a file
sys_read - Read data from file
sys_write - Write data to file
sys_write2 - Write data to file