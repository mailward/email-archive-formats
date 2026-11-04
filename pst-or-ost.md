---
title: PST or OST? Four header fields that tell you, and why some readers refuse OSTs
description: A PST and an OST share one container format, but four header fields say which one a file is, how its pages are laid out and how its data is encoded. A 26-line Python script reads them; here is what it printed for two public PSTs and a 4 KB-page OST.
---

# PST or OST? Four header fields that tell you, and why some readers refuse OSTs

Outlook writes two kinds of data file in the same container format: a **PST** is a personal archive or export, an **OST** is the offline copy Outlook keeps of an Exchange or Microsoft 365 mailbox. Tools that open one often refuse the other, and renaming `.ost` to `.pst` changes nothing, because the file says what it is in its first 514 bytes. Notes from writing the reader in [Mailward](https://pst.aivismonitor.com) (disclosure: I build it); the fields are from Microsoft's [MS-PST] specification and the libpff format documentation, and every output below was produced on 4 November 2026 from three public files: `enron.pst` from the pst-extractor package's `example/testdata/`, `dist-list.pst` from java-libpst's test resources, and `pstextractortest@outlook.com.ost` from pst-extractor's own tests (SHA-256 `daacd8d4…5f0`, 16,818,176 bytes).

## The fields

| What | Where | Values |
|---|---|---|
| `wMagicClient` | bytes 8–9 | `SM` in a PST, `SO` in an OST |
| `wVer` — layout | bytes 10–11 | 14 or 15: ANSI (32-bit offsets); 23: Unicode (64-bit offsets); 36: Unicode with 4,096-byte pages; 37: written with Windows Information Protection |
| `wVerClient` | bytes 12–13 | [MS-PST] gives 19 for the format it describes |
| `bCryptMethod` — how the data blocks are encoded | byte 461 (ANSI) or 513 (Unicode) | 0x00 none, 0x01 permute, 0x02 cyclic, 0x10 Windows Information Protection |

[MS-PST] itself only allows `SM` ("MUST be { 0x53, 0x4D }") and does not list `wVer` 36; both come from libpff's documentation, which records `SO` as the OST value and 36 as the 64-bit 4k-page format first seen in Outlook 2013 OSTs.

## The script

Python 3, standard library only:

```python
import os, sys

CRYPT = {0x00: "NDB_CRYPT_NONE", 0x01: "NDB_CRYPT_PERMUTE", 0x02: "NDB_CRYPT_CYCLIC", 0x10: "NDB_CRYPT_EDPCRYPTED"}

path = sys.argv[1]
with open(path, "rb") as f:
    h = f.read(514)
if len(h) < 514 or h[:4] != b"!BDN":
    sys.exit(f"{path}: not a PST/OST (no !BDN signature)")
client = h[8:10]
ver = int.from_bytes(h[10:12], "little")
ver_client = int.from_bytes(h[12:14], "little")
kind = {b"SM": "PST", b"SO": "OST"}.get(client, f"unknown ({client!r})")
if ver in (14, 15):
    layout, crypt = "ANSI, 32-bit offsets, 512-byte pages", h[461]
elif ver == 36:
    layout, crypt = "Unicode, 64-bit offsets, 4,096-byte pages", h[513]
elif ver >= 23:
    layout, crypt = "Unicode, 64-bit offsets, 512-byte pages", h[513]
else:
    sys.exit(f"{path}: wVer {ver} is not a version the format defines")
print(f"{os.path.basename(path)}: {os.path.getsize(path):,} bytes")
print(f"  wMagicClient  {client.decode('latin-1')!r:6} -> {kind}")
print(f"  wVer          {ver:<6} -> {layout}" + (" (37 = written with Windows Information Protection)" if ver == 37 else ""))
print(f"  wVerClient    {ver_client}")
print(f"  bCryptMethod  0x{crypt:02x}   -> {CRYPT.get(crypt, 'not a value the format defines')}")
```

## What it printed

```
enron.pst: 13,984,768 bytes
  wMagicClient  'SM'   -> PST
  wVer          23     -> Unicode, 64-bit offsets, 512-byte pages
  wVerClient    19
  bCryptMethod  0x01   -> NDB_CRYPT_PERMUTE
dist-list.pst: 271,360 bytes
  wMagicClient  'SM'   -> PST
  wVer          23     -> Unicode, 64-bit offsets, 512-byte pages
  wVerClient    19
  bCryptMethod  0x01   -> NDB_CRYPT_PERMUTE
pstextractortest@outlook.com.ost: 16,818,176 bytes
  wMagicClient  'SO'   -> OST
  wVer          36     -> Unicode, 64-bit offsets, 4,096-byte pages
  wVerClient    12
  bCryptMethod  0x00   -> NDB_CRYPT_NONE
```

## Reading it

- **`SO` is where strict readers stop.** A reader that checks `wMagicClient` against the specification rejects every OST at byte 8, however readable the rest is — the specification describes only PSTs. If a tool says an OST is "not a valid PST", run the script: `SO` with a sensible `wVer` means the file is an ordinary OST, not a damaged one.
- **36 changes the page size, and not every reader knows it.** Its B-tree pages are 4,096 bytes instead of 512, and a reader that checks `wVer` against the versions it knows refuses the file outright. Knowing it is not the whole job: pst-extractor 1.12.0 opens this OST, but in its "United States holidays" calendar, whose own count says 160 items, it lists 24 — a ten-line Node script calling `getNextChild()` until it returned null stopped there on 4 November 2026. pst-extractor finds each row of a folder's contents table with arithmetic for the 8,176-byte data blocks of the Unicode PST layout; this OST keeps the whole table, 160 rows of 339 bytes, in one deflated block of 54,240 bytes, so from the 25th row on it reads the wrong place and stops. Mailward reads that table row after row for version-36 files and lists all 160.
- **`wVerClient` is not a reliable check.** The specification gives 19; this OST says 12, and pst-extractor never reads the field. A reader that insisted on 19 would refuse a file that opens fine.
- **`bCryptMethod` decides whether the bytes can be read at all.** Both PSTs use the permute encoding (0x01) and the OST none (0x00). pst-extractor refuses cyclic-encoded files (0x02), and so does Mailward, which also refuses `wVer` 37: it cannot decrypt Windows Information Protection.

Mailward opens [OSTs in the browser](https://pst.aivismonitor.com/ost-viewer), the 4,096-byte-page ones included, without Outlook or the Exchange account the OST came from, and the file never leaves your computer.

## Sources

- [MS-PST] 2.2.2.6 HEADER (`wMagicClient`, `wVer`, `wVerClient`, `bCryptMethod`): [learn.microsoft.com/openspecs/office_file_formats/ms-pst](https://learn.microsoft.com/en-us/openspecs/office_file_formats/ms-pst/c9876f5a-664b-46a3-9887-ba63f113abf5)
- libpff, "Personal Folder File (PFF) format" (`SO`, version 36): [github.com/libyal/libpff/documentation](https://github.com/libyal/libpff/tree/main/documentation)
- The test files: [pst-extractor](https://github.com/epfromer/pst-extractor) (`example/testdata/enron.pst`, `src/__tests__/testdata/pstextractortest@outlook.com.ost`; MIT) and [java-libpst](https://github.com/rjohnsondev/java-libpst) (`src/test/resources/dist-list.pst`; Apache 2.0 / LGPL)
- Checking whether a PST or OST is complete: [Is this PST complete?](is-my-pst-complete.md)
- The format overview these notes extend: [Email archive formats: field notes](./)

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
