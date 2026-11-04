---
title: What is inside a winmail.dat? Listing its attachments with the Python standard library
description: A winmail.dat is Outlook's TNEF wrapper around one message. A 73-line standard-library Python script walks its attributes, finds every attachment and its long file name, and checks every checksum — run here on Apache POI's public test files.
---

# What is inside a winmail.dat? Listing its attachments with the Python standard library

When Outlook sends a message in its own Rich Text format and the receiving program does not understand it, the recipient sees a plain message with one attachment called `winmail.dat` (or `win.dat`). Microsoft's Exchange documentation describes exactly that: clients that don't support TNEF "typically display TNEF formatted messages as plain text messages with Winmail.dat or Win.dat attachments". The attachments the sender meant to send are inside that file. Notes from writing the TNEF reader in [Mailward](https://pst.aivismonitor.com) (disclosure: I build it); the layout is from Microsoft's [MS-OXTNEF] specification, and every output below was produced on 4 November 2026 from two of the public winmail.dat files in Apache POI's test data (`test-data/hmef/`, Apache License 2.0).

## The layout

A TNEF stream is a 6-byte preamble and then a run of attributes, all little-endian:

| What | Size |
|---|---|
| Signature `78 9F 3E 22` (0x223E9F78) | 4 bytes, once |
| Legacy key | 2 bytes, once |
| **Each attribute:** level — 1 for the message, 2 for an attachment | 1 byte |
| Attribute id | 4 bytes |
| Length of the data | 4 bytes |
| The data | *length* bytes |
| Checksum | 2 bytes |

The attachment-level attributes that matter:

| Attribute | Id (low 16 bits) | Holds |
|---|---|---|
| `attAttachRenddata` | 0x9002 | starts a new attachment; everything after it belongs to that attachment |
| `attAttachTitle` | 0x8010 | a file name in an 8-bit code page — often the 8.3 short name |
| `attAttachData` | 0x800F | the file's bytes |
| `attAttachment` | 0x9005 | a MAPI property list, which carries `PR_ATTACH_LONG_FILENAME` (property 0x3707) |

## The script

Python 3, standard library only. It compares only the low 16 bits of each attribute id, as Mailward's reader does, and decodes 8-bit names as Windows-1252 — the file does not record which code page the sender used for them.

```python
import hashlib, os, struct, sys

SIGNATURE = 0x223E9F78
ATTACHMENT_LEVEL = 2
RENDDATA, TITLE, DATA, ATTACHMENT = 0x9002, 0x8010, 0x800F, 0x9005
LONG_FILENAME = 0x3707  # PR_ATTACH_LONG_FILENAME
FIXED = {0x2: 2, 0x3: 4, 0x4: 4, 0x5: 8, 0x6: 8, 0x7: 8, 0xA: 4, 0xB: 2, 0x14: 8, 0x40: 8, 0x48: 16}


def long_name(props):
    """PR_ATTACH_LONG_FILENAME from an attAttachment property list, if present."""
    count, = struct.unpack_from("<I", props, 0)
    pos = 4
    for _ in range(count):
        ptype, pid = struct.unpack_from("<HH", props, pos); pos += 4
        if pid >= 0x8000:  # named property: GUID, kind, then an id or a name
            kind, = struct.unpack_from("<I", props, pos + 16); pos += 20
            if kind == 0:
                pos += 4
            else:
                n, = struct.unpack_from("<I", props, pos); pos += 4 + n + (-n % 4)
        base, multi = ptype & 0x0FFF, ptype & 0x1000
        if base in (0x0, 0x1):
            continue
        variable = base in (0x0D, 0x1E, 0x1F, 0x102)
        values = 1
        if multi or variable:
            values, = struct.unpack_from("<I", props, pos); pos += 4
        for _ in range(values):
            if variable:
                n, = struct.unpack_from("<I", props, pos); pos += 4
                raw = props[pos:pos + n]; pos += n + (-n % 4)
                if pid == LONG_FILENAME:
                    return (raw.decode("utf-16-le") if base == 0x1F else raw.decode("cp1252")).rstrip("\0")
            else:
                size = FIXED[base]; pos += size + (-size % 4)
    return None


def attachments(blob):
    if struct.unpack_from("<I", blob, 0)[0] != SIGNATURE:
        raise SystemExit("not a TNEF stream (no 78 9F 3E 22 signature)")
    pos, found, bad_sums = 6, [], 0  # signature, then a 2-byte legacy key
    while len(blob) - pos >= 11:  # level, id, length, checksum
        level = blob[pos]
        attr_id, length = struct.unpack_from("<II", blob, pos + 1)
        data = blob[pos + 9:pos + 9 + length]
        checksum, = struct.unpack_from("<H", blob, pos + 9 + length)
        bad_sums += checksum != sum(data) % 65536
        pos += 11 + length
        if level != ATTACHMENT_LEVEL:
            continue
        attr = attr_id & 0xFFFF
        if attr == RENDDATA or not found:
            found.append({})
        if attr == TITLE:
            found[-1]["title"] = data.rstrip(b"\0").decode("cp1252")
        elif attr == DATA:
            found[-1]["data"] = data
        elif attr == ATTACHMENT:
            found[-1]["long"] = long_name(data)
    return found, bad_sums, len(blob) - pos


path = sys.argv[1]
items, bad_sums, left = attachments(open(path, "rb").read())
print(f"{os.path.basename(path)}: {len(items)} attachment(s), {bad_sums} bad checksum(s), {left} byte(s) after the last attribute")
for a in items:
    data = a.get("data", b"")
    print(f"  {a.get('long') or '-'}\n      title {a.get('title', '-')}, {len(data):,} bytes, sha256 {hashlib.sha256(data).hexdigest()[:16]}")
    if len(sys.argv) > 2 and data:
        os.makedirs(sys.argv[2], exist_ok=True)
        open(os.path.join(sys.argv[2], os.path.basename(a.get("long") or a.get("title"))), "wb").write(data)
```

Run it as `python3 winmail_list.py winmail.dat` to list, or `python3 winmail_list.py winmail.dat out/` to also save the files.

## What it printed

`quick-winmail.dat` (66,276 bytes):

```
quick-winmail.dat: 5 attachment(s), 0 bad checksum(s), 0 byte(s) after the last attribute
  quick.doc
      title quick.doc, 19,968 bytes, sha256 1240639edc264abf
  quick.html
      title QUICK~1.HTM, 428 bytes, sha256 5e7daab0b3edcfee
  quick.pdf
      title quick.pdf, 18,638 bytes, sha256 263bea348ce44185
  quick.txt
      title quick.txt, 235 bytes, sha256 becf39adaa5a3526
  quick.xml
      title quick.xml, 143 bytes, sha256 cc1704ac3bf0c4b8
```

POI ships the five original files next to this sample (`quick-contents/`). Saved with the script and compared with `cmp`, all five are byte-for-byte identical to them.

`bug52400-winmail-with-attachments.dat` (321,402 bytes, an Exchange Online message):

```
bug52400-winmail-with-attachments.dat: 2 attachment(s), 0 bad checksum(s), 2 byte(s) after the last attribute
  scion_tc_2007_maintenanceguide.pdf
      title scion_tc_2007_maintenanceguide.pdf, 193,258 bytes, sha256 b617b1efa60d79c4
  Duke_Wave.png
      title Duke_Wave.png, 122,016 bytes, sha256 7c02c7331088a316
```

## Reading it

- **Save under the long name.** In `quick-winmail.dat` the title of the HTML attachment is the 8.3 name `QUICK~1.HTM`; the real name, `quick.html`, is only in the MAPI property list. The Exchange Online sample puts the full name in both places, so a reader that uses only the title gets some files right and some wrong.
- **The checksum is the sum of the data bytes.** In both files every attribute's checksum equals the sum of its data bytes modulo 65,536 — zero mismatches. A failed sum is a quick sign that the file was damaged in transit, though a reader can still take the attachments it can parse.
- **Trailing bytes are not damage.** POI's two Exchange Online samples (`bug52400-winmail-simple.dat` too) each end with 2 bytes after the last attribute: `0D 0A`, a line break — the end of the MIME part the file was saved from. A parser that demands the stream end exactly on an attribute boundary reports these files as corrupt; fewer bytes than an attribute header (11) can be ignored.
- **The message text is not an attachment.** In `quick-winmail.dat` it is Rich Text, compressed as `PR_RTF_COMPRESSED` ([MS-OXRTFCP]); POI's `message.rtf` next to the sample is its decompressed form. The two Exchange samples carry HTML instead. This script lists attachments only.

[Mailward's winmail.dat viewer](https://pst.aivismonitor.com/winmail-dat-viewer) reads the same attributes in a browser tab — attachments under their long names, plus the message text decoded from its HTML or compressed RTF — and the file never leaves your computer.

## Sources

- [MS-OXTNEF]: Transport Neutral Encapsulation Format (TNEF) Data Algorithm — [learn.microsoft.com/openspecs/exchange_server_protocols/ms-oxtnef](https://learn.microsoft.com/en-us/openspecs/exchange_server_protocols/ms-oxtnef/)
- [MS-OXRTFCP]: Rich Text Format (RTF) Compression Algorithm — [learn.microsoft.com/openspecs/exchange_server_protocols/ms-oxrtfcp](https://learn.microsoft.com/en-us/openspecs/exchange_server_protocols/ms-oxrtfcp/)
- Exchange Server: TNEF conversion options (why recipients see Winmail.dat) — [learn.microsoft.com/exchange/mail-flow/content-conversion/tnef-conversion](https://learn.microsoft.com/en-us/exchange/mail-flow/content-conversion/tnef-conversion)
- The test files: Apache POI, [`test-data/hmef/`](https://github.com/apache/poi/tree/trunk/test-data/hmef) (Apache License 2.0)
- The format overview these notes extend: [Email archive formats: field notes](./)

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
