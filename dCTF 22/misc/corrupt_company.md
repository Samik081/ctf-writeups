## Corrupt Company (200 points)

### Description
A whistleblower from Best Company Ever inc. sent us this corrupt file that may be containing incriminating data. Help him find the proof of company's evil plans!

### JPG File 
###### you can download it and try yourself :)
![](nothing.jpg)

### File analysis
We are given `nothing.jpg` file. Typically, in this case, I would start with `binwalk`:

```console
$ binwalk nothing.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
30            0x1E            TIFF image data, big-endian, offset of first image directory: 8
28768         0x7060          Zip archive data, at least v2.0 to extract, name: secret/
28805         0x7085          Zip archive data, at least v2.0 to extract, compressed size: 50122, uncompressed size: 53337, name: secret/Business plan.docx
79022         0x134AE         Zip archive data, at least v2.0 to extract, compressed size: 77, uncompressed size: 627, name: secret/.h/not_important.txt
```
So, there's a `.docx` file and **apparently not important** `not_important.txt` archive file.

Let's extract the latter and analyze it:

### Extraction of the flag content

```console
$ dd bs=1 skip=79022 if=nothing.jpg of=flag.zip

134+0 records in
134+0 records out
134 bytes copied, 0.0350347 s, 3.8 kB/s

$ zipdetails flag.zip
No Central Directory records found
```

Looks like the ZIP archive is broken.

### Fixing the ZIP archive


Let's try to repair it using `zip -FF` and try to get the flag:
```console
$ zip -FF flag.zip --out repaired_flag.zip

Fix archive (-FF) - salvage what can
        zip warning: Missing end (EOCDR) signature - either this archive
                     is not readable or the end is damaged
Is this a single-disk archive?  (y/n): y
  Assuming single-disk archive
Scanning for entries...
 copying: secret/.h/not_important.txt  (77 bytes)
 
$ unzip repaired_flag.zip
Archive:  repaired_flag.zip
secret/.h/not_important.txt:  ucsize 627 <> csize 77 for STORED entry
         continuing with "compressed" size value
 extracting: secret/.h/not_important.txt   bad CRC 08d3bdbd  (should be fd4cb455)

$ cat secret/.h/not_important.txt

,VHIUy%
E%y%z\CV]c_eXoX_noR)&y
`
```

You can already notice `bad CRC` error and that we get some binary data instead of plain text flag. Let's analyze the repaired zip:

```console

$ zipdetails repaired_flag.zip

0000 LOCAL HEADER #1       04034B50
0004 Extract Zip Spec      14 '2.0'
0005 Extract OS            00 'MS-DOS'
0006 General Purpose Flag  0000
0008 Compression Method    0000 'Stored'
000A Last Mod Time         548C6643 'Tue Apr 12 12:50:06 2022'
000E CRC                   FD4CB455
0012 Compressed Length     0000004D
0016 Uncompressed Length   00000273
001A Filename Length       001B
001C Extra Length          0000
001E Filename              'secret/.h/not_important.txt'
0039 PAYLOAD               ...,VH..IU..y.%.....E%.y%z.\C.*....\.V].
                           c._eX.oX._n.oR......).....&y.....`...

0086 CENTRAL HEADER #1     02014B50
008A Created Zip Spec      00 '0.0'
008B Created OS            00 'MS-DOS'
008C Extract Zip Spec      14 '2.0'
008D Extract OS            00 'MS-DOS'
008E General Purpose Flag  0000
0090 Compression Method    0000 'Stored'
0092 Last Mod Time         548C6643 'Tue Apr 12 12:50:06 2022'
0096 CRC                   FD4CB455
009A Compressed Length     0000004D
009E Uncompressed Length   00000273
00A2 Filename Length       001B
00A4 Extra Length          0000
00A6 Comment Length        0000
00A8 Disk Start            0000
00AA Int File Attributes   0000
     [Bit 0]               0 'Binary Data'
00AC Ext File Attributes   00000000
00B0 Local Header Offset   00000000
00B4 Filename              'secret/.h/not_important.txt'

00CF END CENTRAL HEADER    06054B50
00D3 Number of this disk   0000
00D5 Central Dir Disk no   0000
00D7 Entries in this disk  0001
00D9 Total Entries         0001
00DB Size of Central Dir   00000049
00DF Offset to Central Dir 00000086
00E3 Comment Length        0000
```

At this point, we should notice following lines:

```console
0000 LOCAL HEADER #1       04034B50
0008 Compression Method    0000 'Stored'
0012 Compressed Length     0000004D
0016 Uncompressed Length   00000273
...
0086 CENTRAL HEADER #1     02014B50
0090 Compression Method    0000 'Stored'
009A Compressed Length     0000004D
009E Uncompressed Length   00000273
```

Apparently, data is compressed - we could not read the file, got bad CRC and the Compressed and Uncompressed Length values differ. 

However, the compression method is set to `0x00` which means, there's no compression. This looks like the broken part, so let's try to fix it. I will assume the strongest compression first (`8`).

To do this, we are going to change the compression byte for the **Local file header** from `0x00` to `0x08` using some hex editor or bash command:
```console
$ printf '\x08' | dd of=repaired_flag.zip bs=1 seek=$((0x08)) count=1 conv=notrunc

1+0 records in
1+0 records out
1 byte copied, 0.000863 s, 1.2 kB/s
```

Let's try to unzip and get the flag now:
```console
$ unzip repaired_flag.zip && cat secret/.h/not_important.txt
Archive:  repaired_flag.zip
  inflating: secret/.h/not_important.txt
This file is not important.

( ... lots of whitespaces ... )

dctf{pl5_z1p_1t_w3_4r3_4_g00d_c0mp4ny}
```




