# The Baikal SATA PHY is tuned from an uninitialised pointer

> **A note on references.** This repository publishes the specification, the
> driver and the corrections log. It does not publish the per-boot serial
> captures, the per-build records, or the working drafts those documents cite
> as evidence, because they are specific to one console and one machine.
> Citations below name them (`the bring-up log`, `attempt 30`, `build #46`)
> and the reasoning stands on its own; the raw logs are not part of this tree.


Found while investigating something unrelated: the PS4's own boot log
complaining about its disk. It has nothing to do with the Ethernet driver this
repository is otherwise about.

**Scope.** This affects Baikal consoles running the `ps4-linux` kernel lineage,
and only when `apcie_bpcie_buffer` happens to be non-NULL. A NULL pointer takes
a different branch that uses 6 anyway. The value observed on the one console
this was found on was **4**. How often it bites elsewhere, and what garbage
value other machines land on, is not known.

## The defect

`drivers/ata/ahci.c` computes the Baikal SATA PHY trace length like this:

```c
void *apcie_bpcie_buffer;                       /* declared in ahci.h */

bpcie_buffer = (int)ctlr_->apcie_bpcie_buffer;
if ( bpcie_buffer )
{
    trace_length = 6;
    v55 = (unsigned __int8)bpcie_buffer >> 5;
    v56 = v55 - 2;
    if ( v55 <= 2 )
      v56 = 0;
    v57 = v56 + ((_QWORD)ctlr_->apcie_bpcie_buffer & 0x1F);
    if ( v57 <= 0x12 )
      trace_length = v56 + ((_QWORD)ctlr_->apcie_bpcie_buffer & 0x1F);
    switch ( trace_length ) { /* ... PHY tuning register writes ... */ }
}
```

`apcie_bpcie_buffer` is **declared in `ahci.h` and assigned nowhere in
`drivers/ata`.** The code casts an uninitialised pointer to an `int`, takes
its low bits, and uses the result to select which PHY tuning values get
written.

On the console this was found on it produced **4**.

## What the right value is

The vendor firmware reports it, for both controllers, on the same hardware.
It reads the value from per-console board data over ICC:

```
ahci0: Baikal SATA PHY Trace length : 8      (controller advertises 3 Gbps)
ahci1: Baikal SATA PHY Trace length : 6      (controller advertises 6 Gbps, carries the drive)
```

Linux brings up one controller, which advertises `1 ports 6 Gbps` and carries
the same drive the firmware enumerates on `ahci1`. On that basis its trace
length is **6**.

That identification is an inference and worth stating as one. It rests on the
controller's advertised capability and on which drive is attached, not on
anything that names the port directly. It is the best available mapping, and
if the single Linux controller is in fact the firmware's `ahci0`, the right
value would be 8 instead. See the bucket note below: 6, 7 and 8 select
identical writes, so the inference does not change the outcome either way.

`ahci.c` already hardcodes 6 on its other path — the
`apcie_bpcie_buffer == NULL` branch.

## What the wrong value actually did

Twelve read-modify-writes, same registers and order on both paths. **Nine of
twelve values differ** between the garbage `4` and the correct `6`:

| # | Register | Field | at `4` | at `6` |
|---|---|---|---|---|
| 1 | `0x204C` | `[21:16]` | 30 | **32** |
| 2 | `0x204C` | `[29:24]` | 0 | **1** |
| 3 | `0x2054` | `[14:13]` | `0x2000` | `0x2000` |
| 4 | `0x207C` | `[11:6]` | `0x840` | **`0x880`** |
| 5 | `0x207C` | `[17:12]` | 2 | **6** |
| 6 | `0x205C` | `[29:28]` | `0x10000000` | `0x10000000` |
| 7 | `0x2080` | `[11:6]` | `0x8C0` | **`0x900`** |
| 8 | `0x2080` | `[17:12]` | 7 | **15** |
| 9 | `0x205C` | `[31:30]` | `0x40000000` | `0x40000000` |
| 10 | `0x204C` | `[3:0]` | 3 | **5** |
| 11 | `0x206C` | `[11:8]` | 1 | **2** |
| 12 | `0x2084` | `[7:0]` | `0x43` | **`0x55`** |

Values rise monotonically with trace length across all five buckets
(`0x204C[21:16]`: 0x1D → 0x1E → 0x20 → 0x24 → 0x26), so this is a
length-indexed equaliser and de-emphasis table. The garbage `4` selected the
bucket **one notch below** correct.

## Why this is safer to change than it sounds

**Trace lengths 6, 7 and 8 share one bucket.** The firmware's `8` for the
other port selects the identical writes, so being wrong by ±2 changes nothing.
The correct value sits in the middle of a tolerant band rather than on a
cliff.

Measured on hardware, before and after: `SStatus 123 SControl 300`,
`1 ports 6 Gbps` and `configured for UDMA/133` are **byte-identical** across
three boots at trace length 4 and one at 6. The drive enumerates the same; no
transport errors appear on either.

**The link runs at 3.0 Gbps, and the fix does not change that.** `SStatus
0x123` decodes as DET 3 (device present, communication established), **SPD 2,
which is Gen2 at 3.0 Gbps**, and IPM 1. Linux prints the same conclusion two
lines away as `ata1: SATA link up 3.0 Gbps`. The `1 ports 6 Gbps` line is the
*controller's* capability and says nothing about the negotiated speed.

Worth being blunt about, because it is an easy mistake and was made here
first: fixing the trace length did **not** take the link from 3 to 6 Gbps. It
was 3.0 before and 3.0 after. Whatever holds the link at Gen2 is the drive or
the negotiation, not the PHY tuning, and this change does not address it.

That leaves the honest summary: the fix replaces an uninitialised pointer with
a sourced constant and nothing measurable changed. That is the expected result
when the old value was already inside the correct tolerance bucket, and it is
not evidence that the old value was fine.

## The fix

[`../patches/5.4.247-baikal-sata-phy-trace-length.patch`](../patches/5.4.247-baikal-sata-phy-trace-length.patch)
replaces the derivation with the constant. The patch is independent of the
Ethernet work and useful on its own.

## What is still wrong with it

**It is a constant, not a measurement.** Trace length is a property of the
board's routing. 6 is correct for this one; a board routed differently needs a
different value.

The firmware reads it over ICC. The transport already exists in Linux as
`bpcie_icc_cmd()` in `drivers/ps4/ps4-bpcie-icc.c` — what is missing is the
major/minor for the board-data read, which has not been decoded. A related
call was spotted in the firmware's PHY thread (`icc_nvs_read(block 4,
offset 0x38, len 1)`, inferred to be a trace-length or board-variant code),
which is probably where to start.

Until then this trades an uninitialised pointer for a documented constant,
which is an improvement and not a solution.

## A caution about attributing disk errors

This was found while chasing EXT4 corruption, and the corruption turned out
**not** to be caused by it. On the machine in question the damage was a
genuinely failing USB stick: every `blk_update_request: I/O error` in the log
was on the USB device, none on the SATA drive, and the kernel logged
`I/O error while writing superblock` against the stick.

Both misattributions are easy and they point opposite ways. A mistuned PHY
produces transport errors — I/O errors, CRC failures, link resets. Unclean
shutdowns and dying flash produce filesystem metadata damage — inconsistent
bitmaps, doubly-allocated inodes — with a clean transport underneath.

They look similar in a bug report and are distinguishable in a kernel log.
Check which device the errors name before concluding anything.
