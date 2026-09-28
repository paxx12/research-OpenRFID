# Anycubic ACE 1 Pro (Gen 1) — third-party tag reading / UID exposure

Notes and a working route for making an **Anycubic ACE 1 Pro (Gen 1, KlipperGo-stack)** read
**third-party** NFC filament tags (NTAG / OpenSpool / Bambu / Creality) and expose the raw UID, without
an external reader and without hardware modification.

## Why it matters for RFID research

* The Gen 1 reader is an **MFRC522-compatible** chip behind the stock firmware's own primitives:
  `read_reg` at `0x08019D12`, `write_reg` at `0x08019E1C`, tag identification at `0x0800A6A0`.
  Only its version register and factory register values are reachable without a firmware-side hook —
  reads returned `0xA1` for the version register and consistent vendor values.
* The ACE exposes **no file system** for tag data. Tags are parsed by the firmware into a per-slot
  record (**164 bytes each, base `0x20006518`, stride 164**) with a `rfid` flag at record `+0x16`
  (`1` = unrecognised, `2` = recognised), followed by magic/version and a code string.
* The raw UID buffer sits at a fixed address that differs per build: `0x2000012B` (stock image) and
  `0x20006224` (community firmware build).
* There is **one reader chip per slot pair** — chips A and B, pairs `1&3` and `2&4` (inside a pair the
  neighbour is `slot xor 2`, confirmed by writing `TReloadRegL (0x2D)` and reading it back:
  `ctx0→0xA2, ctx1→0xA3, ctx2→0xA2, ctx3→0xA3`). Slots inside a pair cannot be separated by the
  reader driver (TX1/TX2 drive a shared antenna); separation is purely mechanical.

## Working route (currently flashed)

The community CFW already parses third-party tags — the stock path marks them unrecognised before
those parsers run. So the fix is a single hook plus a 108-byte appended stub:

| | |
|---|---|
| base | public community firmware (CFW) image, md5 `9f7b9a678a96caf98d6a08842d3ff971`, 113720 B |
| hook | `VA 0x08016B9A`: `f8 93 10 36` (`ldrb.w r1,[r3,#0x36]`) → `f0 0d f8 4d` (`bl 0x08023C38`) |
| stub | 108 B at end of image (VA `0x08023C38`): position-independent armv7-m code + `0123456789ABCDEF` + literal `0x20006224` |
| output | 113828 B, md5 `6148cfc52431fc235536c6d64b5334ef`, crc16 `0xAE92` |

The stub writes `rfid = 2`, magic `0x007B`, version `0x0065` and 14 hex characters of the UID into the
slot record when the UID buffer holds data, and falls through to the original behaviour otherwise.

Nothing else in the image is touched: exactly 4 modified bytes and 108 appended bytes versus the base.

The route is a proof that a **firmware-side hook can interpose on reader state without an external
reader** — useful when the tag payload is only reachable inside the vendor parser.

## Limitations

* The hook fires only when no stock parser understood the tag. For tags a parser *did* understand, the
  code stays non-unique (literally `OPENSPOOL`), so the record cannot be mapped back to a spool.
  A second hook at `VA 0x08016B32` (parser-succeeded branch) appending `,` + 14 hex UID addresses it;
  968 bytes of update-slot headroom remain.
* Cross-reads inside a slot pair are inherent (shared antenna).
* Flashing resets slot/binding tables.

## Full material

Report (EN), Russian write-up, flasher, stub sources, and images (patched CFW, stock+register-tunnel
variant, clean stock for rollback):

**https://github.com/Godless50/ACE-PRO-v1.-NFC-UID**
