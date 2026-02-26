[English](nand-and-nor.md) | [日本語](nand-and-nor.ja.md) | [简体中文](nand-and-nor.zh-Hans.md)

# NAND and NOR Flash Memory

This document describes the file system formats used on NAND and NOR flash. The main difference is that NOR does not use block addressing, so all descriptions of blocks in this document can be understood as byte addresses in NOR memory.

## NAND Flash Update Image

A NAND flash update image contains two headers and a read-only file system that holds most of the data files for the built-in dictionaries. A small FAT16 file system also resides on the actual NAND flash chip, where files and add-ons transferred from a computer are stored, but this is not part of the NAND update image.

`nand_build` is a NAND building tool that can create an update image for NAND flashing based on a list of files and a NAND layout description. The current version defines layouts for DP4, DP5, and DP6 devices.

### Layout

#### Header 1

Two copies of `hdr1` are actually stored in the update image. The first is at offset `0`, while the second can be found at offset `(hdr1.hdr1_2_blk * hdr1.blocksize)`.

```c
struct hdr1 {
    char signature[12];    // CASIOPVOS???
    uint16_t magic;        // 0x55AA
    uint16_t blocksize ;   // Bytes per block (always 0x200)
    uint32_t flash_nblks;  //
    uint32_t update_nblks; //
    uint32_t dword18;      // Always 0
    uint32_t hdr1_2_blk;   // Block number of the second copy of hdr1
    uint32_t dir_blk;      // Starting block of the directory
    uint32_t dir_nblks;    // Block length of the directory
    uint32_t hdr2_blk;     // Block of the second header
    uint32_t hdr1_2_cpy;   // Block number of the second copy of hdr1?? Always same as hdr1_2_blk
    uint32_t dword30;      // Always 0xffffffff
    uint32_t dword34;      // Always 0xffffffff
    uint32_t dword38;      //
    uint32_t dword3C;      //
    uint32_t dword40;      // Always 0
    uint32_t dword44;      //
    char model[4];         // 4-character model string
    uint32_t dword4C;      //
    uint32_t img_len;      // Block length of the OS update image file
    uint32_t dword54;      //
    uint32_t dword58;      //
    uint32_t dword5C;      //
    uint32_t dword60;      //
    uint32_t dword64;      //
    char ext_model[8];     // Extended model string - DP5+ only, filled with 0xff before DP5
    char filler70[398];    //
    uint16_t checksum;     // 16-bit header checksum
};
```

#### Header 2

The NAND checksum (`datachksum`) is calculated by summing all bytes from after `hdr2` to the end of the file. Using this, custom or modified NAND images can be created for flashing. The `nand_chksums.py` tool can be used to check the NAND checksum.

```c
struct hdr2 {
    char signature[12];    // CASIOPVOS???
    uint32_t dwordC;       // Always 0
    uint32_t dword10;      // Always 0
    uint32_t dword14;      // Always 0xffffffff
    char model[4];         // Model string
    char datetime[8];      // Date time (BCD encoded)
    uint16_t version;      // Version number (BCD encoded)
    char padding[2];       // Padding, should be zero
    uint32_t dword28;      // Always 0
    uint32_t datachksum;   // NAND checksum - sum of all bytes after hdr2
    char ext_model[8];     // Extended model string - DP5+ only, filled with 0xff before DP5
    char filler2C[454];    //
    uint16_t checksum;     // 16-bit header checksum
};
```

### NAND Folder Directory Structure

The following is a list of folders found in the NAND and their presumed functions. Most folders, like `cz???` (Chinese models), are dictionary folders.

-   `ttslib`: Text-to-speech library from Fujitsu (FUJITSU LIMITED 1994). Used for speech synthesis dictionaries, in 11.025 kHz 16-bit format.
-   `txtv`: Unknown.
-   `voice`: Dictionary voice files.
-   `henka`: "henka" (変化) means change or transformation in Japanese. If this folder is removed, the "fuzzy search" in the built-in dictionary will show "Canceled".
-   `fukuE`: "fuku" (副) may refer to adverb or secondary function. Used for "jump search" (third button from the top left) for English words. If removed, jumping on an English word will go to "My Library".
-   `fukuJ`: Used for "jump search" for Chinese or Japanese words.
-   `font`: Stores `.cjf` font files, which can be dumped.
-   `czMDE`: Found in Chinese models. Possibly "Multi-Dictionary English Search" from the function menu.
-   `czMDJ`: Found in Chinese models. Possibly "Multi-Dictionary Japanese Search" from the function menu.
-   `libr`: Unknown.
-   `menu_cn`: Chinese menu resources. If removed, the menu display will be blank, but functions will still work.
-   `menu_jp`: Japanese menu resources.
-   `menu_en`: English menu resources.
-   `sys`: System folder?
-   `demo`: This folder is empty on Chinese models.

## TJFS File System

The file system used on the NAND ROM (presumably "TJFS") is quite simple, as it is read-only and does not support modification once flashed to the ROM. Since the file system is read-only, files are stored on contiguous blocks, and there is no file fragmentation. The root of the file system can only contain directories, and nested directories are not allowed.

### Master Directory Table

This table contains a list of all files and directories in the file system. Each entry is 32 bytes long, listing directories first, followed by all files contained within each directory. All values are stored in big-endian format.

```c
struct DirEntry {
    char filename[16];
    /*
     * Null-terminated string containing the name of the directory or file.
     */
    short dir_id;
    /*
     * For files, this value is always 0xffff.
     * For directories, this is a unique ID used to associate files with a directory.
     */
    short dir_ptr;
    /*
     * For directories, this value is always 0xffff.
     * For files, this value points to the dir_id of its containing directory.
     */
    unsigned long location;
    /*
     * If the entry is a directory, this field is a 0-based index into the master directory table,
     * indicating the start of the file list for that directory.
     * If the entry is a file, this field indicates the starting block number of the file (block size is 512 bytes).
     */
    unsigned long size;
    /*
     * The size of the file in bytes. For directories, this value is always 0.
     */
    unsigned long flags;
    /*
     * The flags field for the entry. Valid values are currently unknown.
     */
};
```

## Test Menu

Casio electronic dictionaries have a hidden engineering mode (Test Menu).

**How to enter:**
1.  Power off the device.
2.  Press and hold **Exit + Page Up + Power** for about five seconds until a window with "Model" appears on the screen.
3.  Release the keys, then press the **Right key twice**, and then press the **Enter key**.

### Menu Option Descriptions

-   **TJFS Physical Format**: Requires a password. Appears to perform a quick format of the TJFS file system. After execution, the device gets stuck on the "Please wait..." screen upon startup.
-   **Force Strong Format**: Erases the NAND. After entering the password, there are three options:
    -   `NACE1`: Erases from `0000` to `0FFF`. The process is slow, block by block. May display a bad block count.
    -   `NACE2`: Displays "CAPA NG" on some devices.
    -   `NACE3`: Appears not to work correctly.
    After execution, "Error Code 01" may be displayed on startup. Strangely, the device seems to boot normally even after formatting; the specific function of this feature is unknown.
-   **DATA IN**: Requires a password. Writes to the NAND from a `DATAOUT.BIN` file on the TF card. If an erase error occurs (possibly due to bad blocks), the process stops and may cause an error code on startup.
-   **DATA OUT**: Exports data from the NAND to the TF card, creating a `DATAOUT.BIN` file. The exported data may contain random bit-flip errors due to ECC (Error Correction Code) failing to fix them. Data exported multiple times may not be consistent.
    -   If multiple `DATAOUT` files are generated (e.g., `DATAOUT1.BIN`, `DATAOUT2.BIN`), they can be merged using `copy /b DATAOUT1.BIN+DATAOUT2.BIN DATAOUT.BIN` (Windows) or `cat DATAOUT1.BIN DATAOUT2.BIN > DATAOUT.BIN` (Linux/macOS).
    -   Files can be extracted from `DATAOUT.BIN` using `test_menu_dump.py`.

## OS UPDATE

The following is an example procedure for a successful OS update using generated NOR and NAND files for an E-A800.

1.  **Format TF Card**: Format a TF card (2GB recommended) in the dictionary's Test Menu.
2.  **Trim NOR File**:
    -   Use a NOR file with the password removed, e.g., `CY123OSB.BIN`.
    -   Use a hex editor (like `hexdump`) to find the boundary of valid data at the end of the file and use a tool like `truncate` to trim the excess `FF` padding bytes.
        ```
        hexdump -C NOR.BIN | tail -n 10 to find the end of the file. You get the following result.

        016ecdb0  55 55 55 55 55 55 55 55  55 50 ff ff ff ff ff ff  |UUUUUUUUUP......|
        016ecdc0  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
        The decimal for offset 016ecdc0 is 24038848. Use truncate -s 24038848 NOR.BIN to trim the file.
        ```
    
    -   Rename the trimmed file to `CY123OSB.BIN` and place it on the TF card. The updater will choose the file that is latest alphabetically (e.g., `CY123OSE.BIN` would be chosen over `CY123OSB.BIN`).
    -   Use `nor_parse.py` to unpack `UPDADN3.BIN` (or `UPDADN2.BIN`) from the NOR image and also place it on the TF card.
3.  **Generate NAND Image**:
    -   Build the NAND flashing image using `nand_build.py`:
        ```sh
        python nand_build.py --layout dp5 --model C123 --ext-model CY123 files_folder CY123D0B.BIN
        ```
    -   It is recommended to copy the files in `files_folder` from the dictionary using the `copy_nand` add-on to avoid errors introduced by `DATAOUT`.
    -   Place the generated NAND image (e.g., `CY123D0B.BIN`) on the TF card as well.
4.  **Start Update**:
    -   Insert the TF card containing `UPDADN*.BIN`, the NOR image, and the NAND image into the dictionary.
    -   Enter the Test Menu to perform the update. You can update only the NOR, only the NAND, or both.