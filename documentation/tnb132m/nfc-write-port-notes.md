# TNB132M NFC write support -- port notes (WIP, not working yet)

Branch: `testing/nfc-tnb132m-i2c`. Do not merge to `main` or open an upstream
PR based on this until the "not working yet" issue below is actually resolved
and verified end-to-end.

## What this is

`oepl_hw_nfc_write_url()` / `oepl_hw_nfc_write_raw()` in
`firmware/oepl_hw_abstraction.c` were stubs (`// Todo: implement nonblocking
I2C driver for NFC`, always `return false`). This ports the I2C write logic
for these functions from
[OpenDisplay/Firmware_Silabs](https://github.com/OpenDisplay/Firmware_Silabs)
(`opendisplay_ble.c`, the `od_nfc_*` function family:
`od_nfc_tnb132m_prime_type3()`, `od_nfc_type3_paged_block_read16/write16()`,
`od_nfc_write_ndef_text_en()`, `od_nfc_init_sequence()`), used with the
original author's permission. Their driver bit-bangs SCL/SDA with plain GPIO;
this port drives the same wire-level protocol through the EFR32's hardware
I2C peripheral instead (the same pins our existing boot-time TNB132M capture
sequence in `oepl_efr32xg22_get_config()`/setup already uses).

See the comment block directly above `oepl_hw_nfc_write_url()` in
`firmware/oepl_hw_abstraction.c` for the full reverse-engineered I2C register
map (`dev 0x30`/`0x43` = prime/status, `dev 0x48 sub 0x00` = Type-3
Attribute Information block, `dev 0x40 sub 0x10/0x20` = NDEF data blocks)
and the 32-byte total-message length limit (`Nbr=2` anchor cache).

## Existing resources in this folder

- `traces/TNB132M-poweron.sal`, `traces/TNB132M-tagread.sal`: Saleae Logic
  captures of a real TNB132M power-on and tag-read sequence (Saleae's
  container format, a Zip archive -- needs the Saleae Logic software to
  open/decode, not attempted as part of this port). Could be useful to
  compare against the write sequence below if this gets picked up again.

## Hardware test result (2026-09-10, EL016F5C4C, ctrltype 0x37)

Ran `oepl_hw_nfc_write_url()` end-to-end on a real tag via the AP's built-in
"Set NFC URL" content card (`DATATYPE_NFC_URL_DIRECT` ->
`application_process_nfcu_block()` in `firmware/oepl_app.c` -> this code),
short test URL (well under the 27-character limit for the URI-record
wrapper).

**Runs to completion without crashing or destabilizing the tag**: the block
is received, MD5-checked, and applied (AP-side `updatecount`/content hash
advance on every retry), and the tag keeps checking in normally afterward on
its usual schedule.

**But the result is not readable as an NFC tag by a real reader**:
- NXP TagInfo only identifies the chip at the raw IC level ("FeliCa, 6KB
  EEPROM") -- no NDEF section shown.
- "NFC Tools" (which does active Type-3/FeliCa polling, not just iOS's
  passive background NDEF reader) reads nothing at all.
- iPhone's passive background tag-read notification never fires either, but
  that alone wouldn't be conclusive -- iOS's passive reader's exact
  Type-3/FeliCa coverage is unclear; NFC Tools failing too is the stronger
  signal.

## Open question: where does it actually fail?

Two live hypotheses, not distinguished yet:

1. **The I2C write itself is silently failing.**
   `application_process_nfcu_block()` in `oepl_app.c` never checks this
   function's return value before recording the update as applied -- a
   silent I2C NACK on the AI or data-block write would look externally
   identical to what we observed (AP thinks it succeeded either way, since
   it only tracks "did we receive+apply the block", not the return value of
   the HW write function).
2. **The AI block we construct doesn't satisfy what the chip's RF-side
   firmware needs** to advertise NDEF / System Code `0x12FC` over the air.
   That RF-facing behavior is entirely internal to the TNB132M and invisible
   over the I2C map we've reverse-engineered from the MCU side -- we can
   write bytes to `dev 0x48`/`0x40`, but we have no visibility into whether
   the chip's own firmware is happy with what we wrote (right
   `Ver`/`Nbr`/`Nbw`/`Nmaxb`, right checksum algorithm, etc.).

`oepl_hw_nfc_write_raw()` (the Text-record framing that actually matches
OpenDisplay's verified reference, `od_nfc_write_ndef_text_en()`) has **not**
been separately hardware-tested -- the AP's web UI only exposes a "Set NFC
URL" content card, no raw-NDEF one, and there was no time to wire up a manual
`save_cfg` call for `DATATYPE_NFC_RAW_CONTENT` this session. Given
`write_url()`'s failure, don't assume `write_raw()` fares any better
untested; both funnel through the same underlying
`tnb132m_write_ndef_message()`/`tnb132m_write_ndef_blocks()`.

## Next step, if this gets picked up again

A temporary SWD-readback diagnostic exists on top of the port, as its own
commit (`WIP: temporary SWD-readback diagnostic for NFC write (DO NOT
MERGE)`) -- **revert that commit** before flashing anything meant to
actually work, it parks the CPU in a busy-loop after one write attempt and
the tag never checks in again once it's running.

With that commit applied: trigger a write via the AP as above, wait for one
check-in to confirm the block was processed, then attach with a **plain**
`openocd ... -c "halt"` (not `reset halt` -- that wipes the RAM state we're
trying to inspect; a plain halt works here specifically because the CPU is
spinning, not asleep, so it doesn't hit the EM2/3 debug-clock-gated timeout
this fork ran into elsewhere this session) and dump:

```
g_nfc_debug_write_ok      (bool)   -- did tnb132m_write_ndef_blocks() itself report success?
g_nfc_debug_ai_read_ok    (bool)   -- could we read back dev 0x48 sub 0x00 afterward at all?
g_nfc_debug_ai            (16B)    -- what's actually sitting in the AI block now
```

(exact symbol addresses are printed by `arm-none-eabi-nm` on that specific
build and noted in a comment above the globals in `oepl_hw_abstraction.c` --
they'll shift if the code around them changes, re-run `nm` for a fresh
build).

That tells us which of the two hypotheses above is live: `write_ok == false`
means an I2C-level failure (chase wiring/timing/NACK); `write_ok == true` but
still unreadable by a phone means the AI/data content itself is wrong in a
way we don't understand yet, which likely needs either a real TNB132M
datasheet or decoding the existing `.sal` traces in this folder against a
known-working write from another source (e.g. asking the OpenDisplay author
for a capture of theirs).
