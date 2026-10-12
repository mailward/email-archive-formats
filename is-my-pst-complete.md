---
title: Is this PST complete? Two header fields that tell you
description: A PST records its own size and the location of its folder index in the first 514 bytes. A 25-line Python script compares them with the file on disk to tell an incomplete copy from a damaged one.
---

# Is this PST complete? Two header fields that tell you

A PST or OST that will not open is often not damaged but **incomplete** — a download, sync or copy that stopped before the end. The file's own header says how big it should be and where its folder index lives, so you can check that before reaching for a repair tool. Notes from writing the reader in [Mailward](https://pst.aivismonitor.com) (disclosure: I build it); the offsets are from Microsoft's [MS-PST] specification, and every output below was produced on 12 October 2026 from the 14 MB `enron.pst` that ships in the open-source pst-extractor package (`example/testdata/`), and from copies of it cut short.

## The fields

All in the 514-byte header, little-endian:

| What | Unicode PST/OST (`wVer` 23 and up) | ANSI PST (`wVer` 14, 15) |
|---|---|---|
| Signature | `!BDN` at bytes 0–3 | same |
| Format version `wVer` | byte 10 | byte 10 |
| `ibFileEof` — the file's size when it was last written | 8 bytes at 184 | 4 bytes at 168 |
| Offset of the node B-tree root page (`BREFNBT.ib`) | 8 bytes at 224 | 4 bytes at 188 |
| Offset of the block B-tree root page (`BREFBBT.ib`) | 8 bytes at 240 | 4 bytes at 196 |

The 4 KB-page OST written by Outlook 2013 and later (`wVer` 36) uses the Unicode offsets with 4,096-byte B-tree pages instead of 512. Every folder and message is found through those two B-trees, so if either root page lies past the end of the file, nothing in it can be listed.

## The check

Python 3, standard library only:

```python
import os, struct, sys

path = sys.argv[1]
size = os.path.getsize(path)
with open(path, "rb") as f:
    h = f.read(514)
if len(h) < 514 or h[:4] != b"!BDN":
    sys.exit(f"{path}: not a PST/OST (no !BDN signature in the first 4 bytes)")
ver = h[10]
if ver in (14, 15):
    layout, eof, nbt, bbt, page = "ANSI", struct.unpack_from("<I", h, 168)[0], struct.unpack_from("<I", h, 188)[0], struct.unpack_from("<I", h, 196)[0], 512
elif ver >= 23:
    eof, nbt, bbt = (struct.unpack_from("<Q", h, o)[0] for o in (184, 224, 240))
    layout, page = ("Unicode 4K (OST)", 4096) if ver == 36 else ("Unicode", 512)
else:
    sys.exit(f"{path}: wVer {ver} is not a version MS-PST defines")
index_end = max(nbt, bbt) + page
print(f"{path}: wVer {ver} ({layout})")
print(f"  size when last written (ibFileEof): {eof:,} bytes")
print(f"  size on disk:                       {size:,} bytes")
print(f"  B-tree roots end at:                {index_end:,} bytes")
if eof > size or index_end > size:
    print(f"  INCOMPLETE: {eof - size:,} bytes missing" + ("; the folder index is past the end" if index_end > size else ""))
else:
    print("  header says the file is complete")
```

## What it printed

The whole file:

```
enron.pst: wVer 23 (Unicode)
  size when last written (ibFileEof): 13,984,768 bytes
  size on disk:                       13,984,768 bytes
  B-tree roots end at:                13,923,840 bytes
  header says the file is complete
```

The first half of it, as an interrupted copy leaves it:

```
enron-half.pst: wVer 23 (Unicode)
  size when last written (ibFileEof): 13,984,768 bytes
  size on disk:                       6,992,384 bytes
  B-tree roots end at:                13,923,840 bytes
  INCOMPLETE: 6,992,384 bytes missing; the folder index is past the end
```

Look at where the index is: in this file both B-tree root pages sit in the last 60,928 bytes, more than 99.5% of the way through. Where they land depends on the file's history, but wherever they are, a copy that stops before them has lost the one structure every folder and message is reached through. For this file that means any copy short of 99.5% shows **nothing** in any reader, not the mail that did arrive.

## Reading the verdict

- **Sizes match, index inside the file:** the header has no complaint. The file can still be damaged inside — this check only rules out the most common cause.
- **`ibFileEof` larger than the file:** an incomplete copy. Copy or download it again from the source, and compare sizes; a repair tool cannot invent the missing part.
- **Not `!BDN`:** not a PST at all, whatever its extension says. In Mailward's failure telemetry such files turn out to be ZIPs, PDFs, HTML pages saved under a download's name, or MBOX and EML files renamed.

Mailward reads the same fields when it opens a PST, and when an incomplete copy then fails to parse it says so with both sizes instead of calling the file corrupt. Complete files it reads [in the browser](https://pst.aivismonitor.com), multi-gigabyte ones included, [OST too](https://pst.aivismonitor.com/ost-viewer).

## Sources

- [MS-PST] 2.2.2.6 HEADER and 2.2.2.5 ROOT (Unicode and ANSI layouts): [learn.microsoft.com/openspecs/office_file_formats/ms-pst](https://learn.microsoft.com/en-us/openspecs/office_file_formats/ms-pst/)
- The format overview these notes extend: [Email archive formats: field notes](./)

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
