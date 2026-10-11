# The Baikal SATA PHY is tuned from a hardcoded constant

> **A note on references.** This repository publishes the specification, the
> driver and the corrections log. It does not publish the per-boot serial
> captures, the per-build records, or the working drafts those documents cite
> as evidence, because they are specific to one console and one machine.
> Citations below name them (`the bring-up log`, `attempt 30`, `build #46`)
> and the reasoning stands on its own; the raw logs are not part of this tree.


Found while investigating something unrelated: the PS4's own boot log
complaining about its disk. It has nothing to do with the Ethernet driver this
repository is otherwise about.

**Scope.** This affects Baikal consoles running the `ps4-linux` kernel lineage.
It is gated on `apcie_bpcie_buffer` being non-NULL, and a NULL field takes
a different branch that uses 6 anyway. In practice the assignment in
`xhci-aeolia.c` makes the field non-NULL on every boot, so the decode always
runs and always yields **4**.

## The defect

`drivers/ata/ahci.c` picks the Baikal SATA PHY trace length by decoding a
packed byte out of `ctlr->apcie_bpcie_buffer`, a `void *` field on the
controller struct. The decode takes the top three bits as one term, subtracts
2 with a floor of 0, adds the low five bits, and accepts the result if it is
at most `0x12`. That value then selects which PHY tuning registers get
written.

Describing it rather than quoting it is deliberate: the code in `ps4-linux` is
itself a decompiler dump of Sony's driver, and reproducing it here would mean
republishing decompiled vendor code. The patch has to show the lines it
removes; this document does not.

**The field is not a pointer to anything.** It is assigned once, and not in
`drivers/ata`:

```c
/* drivers/usb/host/xhci-aeolia.c */
ctlr = kzalloc(sizeof(*ctlr), GFP_KERNEL);
ctlr->r_mem = r_mem;
ctlr->dev_id = 0x90DE104D;
ctlr->apcie_bpcie_buffer = (void *)0x04;   /* "//for ahci 0x024" */
bpcie_sata_phy_init(&pdev->dev, ctlr);
```

A literal `4`, cast to a pointer. Run the decode on it: `4 >> 5` is 0, the
floor clamps the first term to 0, the low bits give 4, so the trace length
comes out **4**.

That is measured, not argued. Across every Linux boot captured during this
work the value is the same, with no exceptions:

```
ci_aeolia 0000:00:14.7: Baikal SATA PHY Trace length : 4      21 of 21 boots
```

For contrast, the firmware on the same console reports `ahci0: ... : 8` and
`ahci1: ... : 6`, 25 and 26 times across the same captures. The two sources
never agree and the Linux figure never varies.

The struct makes that determinism easy to see. It is `kzalloc`'d, three fields
are set, it is passed to `bpcie_sata_phy_init()` and freed immediately, so
nothing else can reach it between assignment and use. `bpcie_sata_phy_init()`
is exported from `ahci.c` and this is its **only caller in the tree**;
`ahci.c` never calls it itself.

### `0x04` is probably an offset, not a value

The obvious reading is that `0x04` is a placeholder someone meant to replace
with a real trace length. The comment argues against it. Put both numbers
through the same arithmetic:

| field value | `>> 5` | clamped first term | `& 0x1F` | trace length |
|---|---|---|---|---|
| `0x04`, what is assigned | 0 | 0 | 4 | **4** |
| `0x24`, what the comment names | 1 | 0 | 4 | **4** |

They are identical, because the clamp discards the top three bits for any
value below `0x60`. So the comment cannot mean "use `0x24` here instead" --
that changes nothing. And neither number can be a trace length that was
intended to be correct, since the firmware wants 6 and 8, which this decode
produces from `0x06` and `0x08` (or `0x26` and `0x28`, equivalently).

What fits is that `0x04` and `0x024` are **offsets into the board data**, one
per controller, and the field is named `apcie_bpcie_buffer` because that is
what it was: a pointer to a buffer. The code would then read one byte at that
offset and decode it. The vendor firmware does fetch exactly that kind of
board data over ICC.

**This is a hypothesis, not a finding.** It fits the field's name, the
comment, the arithmetic, and the mismatch with the firmware's values, and
nothing here proves it. What makes it worth writing down is that it is
testable, and the test is in the open-questions section below.

## What the right value is

The vendor firmware reports the real value for both controllers, reading it
from per-console board data over ICC. On a Slim it prints:

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
twelve values differ** between the decoded `4` and the correct `6`:

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
length-indexed equaliser and de-emphasis table. The `4` selected the bucket
**one notch below** correct.

## Why this is safer to change than it sounds

**Trace lengths 6, 7 and 8 share one bucket.** The firmware's `8` for the
other port selects the identical writes, so being wrong by ±2 changes nothing.
The correct value sits in the middle of a tolerant band rather than on a
cliff.

Measured on hardware, before and after: `SStatus 123 SControl 300`,
`1 ports 6 Gbps` and `configured for UDMA/133` are **byte-identical** across
three boots at trace length 4 and one at 6. The drive enumerates identically
and no transport errors appear on either.

**The link runs at 3.0 Gbps, and the fix does not change that.** `SStatus
0x123` decodes as DET 3 (device present, communication established), **SPD 2,
which is Gen2 at 3.0 Gbps**, and IPM 1. Linux prints the same conclusion two
lines away as `ata1: SATA link up 3.0 Gbps`. The `1 ports 6 Gbps` line is the
*controller's* capability and says nothing about the negotiated speed.

Worth being blunt about, because it is an easy mistake and was made here
first: fixing the trace length did **not** take the link from 3 to 6 Gbps. It
was 3.0 before and 3.0 after. Whatever holds the link at Gen2 is the drive or
the negotiation, not the PHY tuning, and this change does not address it.

That leaves the honest summary: the fix replaces a hardcoded constant with a
sourced one and nothing measurable changed. That is the expected result
when the old value was already inside the correct tolerance bucket, and it is
not evidence that the old value was fine.

## The fix

[`../patches/5.4.247-baikal-sata-phy-trace-length.patch`](../patches/5.4.247-baikal-sata-phy-trace-length.patch)
replaces the derivation with the constant. The patch is independent of the
Ethernet work and useful on its own.

**It leaves the read in place, and that is a shortcoming.** The patch changes
what happens inside the `if (bpcie_buffer)` branch but keeps the branch, so
the field is still loaded and tested even though both arms now reach 6.
The cleaner change deletes the conditional and its temporaries outright. That
is a larger diff against a file this project did not write, and it was kept
minimal deliberately, but a port that wants the defect gone rather than
neutralised should remove the whole thing.

## What is still wrong with it

**It is a constant, not a measurement.** Trace length is a property of the
board's routing. 6 is the value the firmware reports for the Slim's 6 Gbps
port; a board routed differently needs a different value, and this patch does
not discover it.

The firmware reads it over ICC. The transport already exists in Linux as
`bpcie_icc_cmd()` in `drivers/ps4/ps4-bpcie-icc.c` — what is missing is the
major/minor for the board-data read, which has not been decoded. A related
call was spotted in the firmware's PHY thread (`icc_nvs_read(block 4,
offset 0x38, len 1)`, inferred to be a trace-length or board-variant code),
which is probably where to start.

**There is a sharper test available, and it has not been run.** If `0x04` and
`0x024` are offsets into a board-data buffer rather than values, then the byte
at offset `0x24` in whatever buffer the firmware pulls should decode to 6, and
the byte at `0x04` should decode to 8, matching what the firmware prints for
the two controllers. Decoding to 6 requires a byte of `0x06` or `0x26`; to 8,
`0x08` or `0x28`. So: find any ICC call that returns a board-data buffer, read
the bytes at those two offsets, and run the decode. If the numbers come out 6
and 8, the offset hypothesis is confirmed and the ICC command is identified in
the same move. If they do not, the hypothesis is dead and the field really is
a value someone hardcoded.

That is the most useful next step on this defect, and it is cheap.

Until then this trades an undocumented constant for a sourced one, which is
an improvement and not a solution.

## Telling this apart from media failure

A mistuned PHY and failing storage both end in a corrupt filesystem, and they
are distinguishable in a kernel log before anyone guesses.

A PHY problem shows up at the transport: `blk_update_request: I/O error`,
`exception Emask`, `SError`, CRC failures, link resets. The filesystem damage
is downstream of those.

Failing flash or an unclean shutdown shows up as metadata damage with a
**clean transport underneath**: inconsistent block bitmaps, doubly-allocated
inodes, `error count since last fsck` climbing, and no ATA errors at all.

Check which device the errors name. On a PS4 running Linux from a USB stick
there are at least two block devices in play, and the EXT4 errors frequently
name the one that is not the SATA drive.
