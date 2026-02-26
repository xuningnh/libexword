# Ex-word Format Description

This document is a collection of notes on the various file formats used in ex-word add-on dictionaries.

## Common Files

### BMP Files
Standard Bitmap image files. They have nothing special about them and can be edited in any program capable of manipulating BMPs.

### guide*.htm
These are normal HTML files used to display the guide/help information for an add-on dictionary.

### diction.htm
A small HTML file containing the title of the add-on dictionary.

**`diction.htm` template:**
```html
<html>
<head>
<title>Add-on Dictionary Title Here</title>
<meta name="soft" content="OFF">
</head>
<body>
</body>
</html>
```

### info*.htm
Configuration files that specify various options for the dictionary. Despite the extension, these are not actual HTML files.

### fileinfo.cji
This is a simple text file that lists the CJT and CJD files used by the add-on dictionary.

## Font and Data Files

### CJF Files
These are simple bitmap font files that appear to have a 256-byte header. They can contain one or more fonts within a single file. They contain a `0x40` byte main header, followed by one or more `0xC0` byte headers describing each font. Following the headers is the actual bitmap data for the fonts.

#### CJF Header (`0x40` bytes)
-   `0x00-0x02`: `CJF` (magic value)
-   `0x03`: `type` (0x81 = single font, 0xC1 = multiple fonts)
-   `0x04-0x07`: unknown (so far only seen 0x40)
-   `0x08-0x09`: unknown
-   `0x0A-0x1F`: Filler (0xFF)
-   `0x20-0x3F`: font title - unused bytes padded with 0x20 (SJIS encoding used)

#### Font header (`0xC0` bytes)
-   `0x00-0x03`: unknown (always seems to be 0x00 in single font files, and 0xFFFFFFFF in multi font files)
-   `0x04-0x07`: next font header offset (0xFFFFFFFF if no more headers)
-   `0x08-0x09`: start unicode code point
-   `0x0A-0x0B`: zero padding
-   `0x0C-0X0D`: end unicode code point
-   `0x0E-0X0F`: zero padding
-   `0x10-0x17`: unknown
-   `0x18-0x19`: height
-   `0x1A-0x1B`: width
-   `0x1C-0x1D`: bytes per glyph
-   `0x1E-0x1F`: flags (bitwise: 0x00 = normal, 0x01 = bold, 0x02 = italic)
-   `0x20-0x27`: Filler (0xFF)
-   `0x28-0x33`: unknown
-   `0x34-0x37`: offset where font data starts
-   `0x38-0xBF`: Filler (0xFF)

### CJD Files
These files are listed in `fileinfo.cji` as `<filename>,<filesize>,<unknown number>`.

These files generally seem to be the data files used by the add-on, though the format used in the file may depend on the APL version being used by the add-on.

### CJT Files
These files are listed in `fileinfo.cji` as `<filename>,0,<bytes per entry>,<number of entries>`.

These files are usually lists of numbers containing `<number of entries>` that are of length `<bytes per entry>`. These are typically lists of indexes and offsets.

-   If `<bytes per entry>` is -1, the file in question seems to be a string table with each string separated by a `0xFF` character.
-   The second field always seems to be zero.

## Dictionary Data Structure

### String tables
These tables are defined by a set of three files: an index, offset, and string file. For example, from the Kenkyusha dictionary, we have:
-   `sjstr.cjt` (string file)
-   `sjoff.cjt` (offset file)
-   `sjidx.cjt` (index file)

These string tables are not used for display purposes but rather for matching strings during a search. The string file contains a series of non-null terminated strings separated by `0xFF` characters. The offsets file contains a list of offsets for each string in the strings file. The index file is likely used to map each string to an entry in the dictionary.

### comp.cjd
This appears to be the main dictionary file that contains all the actual entries. The format seems to have a null-terminated string followed by entry data, repeated for each entry. The null-terminated string is likely used for lookup rather than display.

The entry data itself does not seem to contain any valid SJIS or ASCII strings, so it is believed to be using some form of text compression/encoding. Modifying byte values in the entry data affects the text display for that entry. The `dbadd.cjt` file seems to be a list of offsets to each entry in the `comp.cjd` file.

### wct.cjd, head.cjd, tree.cjd
All dictionaries have these three files, which are never listed in `fileinfo.cji`. It is believed these files are used in the text encoding/compression process.