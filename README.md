# ps4-mts-linux-driver

A Linux gigabit Ethernet driver for the PlayStation 4 Slim's Baikal
southbridge, written from a hardware specification recovered by reverse
engineering. There was no vendor source, no datasheet and no documentation.

It reaches roughly **850 Mbit/s** by the driver's own counters, which is about
90% of the ~940 Mbit/s a TCP payload can occupy on a gigabit link. An external
speed test over the public internet reports 711 down / 768 up, with an ISP in
the path, so treat that as a floor rather than a measurement of the link.

Not yet measured with `iperf3` against a host on the same LAN, which is what
would actually characterise it.

```
eth0: Baikal GBE at xx:xx:xx:xx:xx:xx, IRQ 34
eth0: reg04=0x00000b19 up=1 speed=2 fdx=1 bmsr=796d/796d stat1000=7800
eth0: tx=659265/789878661 drop=0 err=0 q=0 carrier=1 stall=0
```

---

## What this is

Sony's PlayStation 4 runs Orbis, a FreeBSD derivative. Its Ethernet driver is
called `mts`, and nothing about the hardware it drives is public. The Baikal
southbridge's GBE function has no datasheet, no vendor Linux driver, and no
register map anywhere.

This project recovered one. Register offsets, bit meanings, descriptor
layouts, initialisation ordering and PHY calibration were derived by reading
live silicon and disassembling the firmware, then written up as a
specification. The driver is an implementation of that specification.

**[`docs/baikal-gbe-register-map.md`](docs/baikal-gbe-register-map.md) is the
real deliverable.** The driver is one implementation of it; the specification
is what makes other implementations possible.

### Provenance

This is **specification-driven reimplementation with a documented paper
trail**, not clean-room. Clean-room requires two parties with one who never
sees the original, and this project had no such separation: the same person
read the firmware and wrote the driver.

The discipline actually applied: **register offsets, bit positions, value
sequences and ordering are facts about hardware and are recorded. No
decompiler output produced by this project entered the repository.** No
firmware images and no ROM dumps are present here or in this history.

One qualification, because the stronger claim would be false. The SATA patch
necessarily shows the lines it removes, and those lines are themselves a
decompiler dump of Sony's driver that already ships in the `ps4-linux` tree.
That is quoted vendor-derived code, not this project's own decompilation, and
it cannot be avoided in a patch. `docs/sata-phy-trace-length.md` describes
that code in prose rather than reproducing it, for the same reason.

---

## The three bugs

All three were invisible to the obvious checks, and each took a different kind
of evidence to find.

### 1. The shutdown residue

The PHY arrives after a kexec holding a bit that Orbis's **teardown** routine
sets on its way out: a write to Clause 45 register `0x0147` devad `0x1e`, the
last PHY access before the firmware stops. A power cycle clears it. A kexec
does not.

Every comparison the project ran checked driver values against Orbis's
**running** state, and found them identical. That was correct and it was
useless: nothing on either side *programs* that bit, so it sat outside the
entire search space until the teardown routine itself was disassembled.

Clearing one bit brought the link up. `reg04` bit0 set 39 ms later, after
twenty-nine boots of it reading down.

### 2. The last-descriptor hang

With the link up, the transmitter ran for exactly 255 packets, advanced to the
final descriptor of the ring's 4 KB page, and stopped. `txcur` frozen at
`base+0xff0`, the engine-active bit still asserted, **no error reported
anywhere**. Twelve times in a row, 255 packets apart, zero jitter.

A ROM read proved the driver's index-255 descriptor was **byte-identical** to
what the firmware emits for that slot, which is what made the real hypothesis
testable: not *what* we write, but *where*. Using 255 of 256 descriptors, so
WRAP lands on index 254 and index 255 is never queued, eliminated it across
roughly 47 ring wraps and 659,265 packets.

Throughput went from **0.26 Mbit/s to ~848 Mbit/s** (driver counters) by
leaving one descriptor unused. The shape fits a descriptor prefetch crossing
the ring's page boundary. That is inference, not an established cause.

### 3. The SATA PHY tuned from a hardcoded constant

Found while investigating something else. `drivers/ata/ahci.c` decodes the
Baikal SATA PHY trace length out of `ctlr->apcie_bpcie_buffer`, a `void *`, by
shifting and masking it as though it were a packed byte. The result selects
PHY tuning registers in a `switch`.

The field is assigned in exactly one place, and not in `drivers/ata`:
`drivers/usb/host/xhci-aeolia.c` sets it to a literal `(void *)0x04` with the
comment `//for ahci 0x024`, then calls the PHY init and frees the struct. That
is the only caller of `bpcie_sata_phy_init()` in the tree. Decode the literal
and the trace length is **4**, which 21 of 21 Linux boot captures confirm with
no exceptions, against the firmware's 6 and 8 for the two controllers.

Calling it a placeholder is probably wrong, though, and the comment is why.
`0x04` and `0x24` decode to the *same* 4, because the arithmetic discards the
top bits below `0x60`, so the comment cannot mean "use `0x24` instead". The
likelier reading is that both are **offsets into the board data**, which is
what a field named `apcie_bpcie_buffer` would hold, and that the byte read
through it was lost when decompiled code was turned back into C. That is a
hypothesis; `docs/sata-phy-trace-length.md` states it as one and gives the
test that would settle it.

Nine of twelve read-modify-writes differ between `4` and the correct `6`.

**Nothing measurable changed when it was fixed**, and that is the honest
result. The link negotiated 3.0 Gbps before and after. Trace lengths 6, 7 and
8 share one tolerance bucket, so `4` sat one notch below correct rather than
somewhere wild. Tuning a PHY from an undocumented constant is still worth
closing; it just was not the cause of any visible trouble.

---

## Repository layout

```
driver/ps4_gbe.c                  the driver
driver/Kconfig                    vendor menu entry, required by the wiring patch
driver/Makefile                   build rule, required by the wiring patch
docs/baikal-gbe-register-map.md   the recovered hardware specification
docs/mts-init-sequence.md         firmware function inventory + corrections log
docs/sata-phy-trace-length.md     the SATA PHY defect, in detail
patches/                          kernel-tree changes the driver depends on
```

### The corrections log

[`docs/mts-init-sequence.md`](docs/mts-init-sequence.md) ends with a
corrections log, every wrong conclusion this project reached, what the
evidence actually said, and what it cost.

It is long, and most entries are the author's own: an inverted `stat1000` bit
mapping that drove four builds of wrong reasoning, an invented clock scale, a
"missing" ROM step that was never missing, sample counts produced by grepping
across two boots, and a carrier fix claimed twice before it worked.

It is kept deliberately. The bugs above were found by a process that was wrong
repeatedly and wrote down why, and the third attempt at each only worked
because the first two were recorded rather than quietly overwritten.

---

## Building

Targets the PS4 Linux kernel tree (5.4.247, `ps4-linux` lineage).

```sh
# 1. create the vendor directory and install all three files.
#    Kconfig and Makefile are required: the wiring patch sources them, and
#    the build fails at configure time without them.
mkdir -p <kernel-tree>/drivers/net/ethernet/sony
cp driver/ps4_gbe.c driver/Kconfig driver/Makefile \
   <kernel-tree>/drivers/net/ethernet/sony/

# 2. apply the kernel-side changes
cd <kernel-tree>
patch -p1 < <this-repo>/patches/5.4.247-bpcie-glue-read32-and-gbe-wiring.patch
patch -p1 < <this-repo>/patches/5.4.247-baikal-sata-phy-trace-length.patch

# 3. enable it, then build
#    CONFIG_PS4_GBE depends on X86_PS4, so the PS4 platform must be on.
make menuconfig          # Device Drivers -> Network device support -> Sony devices
make
```

### What the base tree already provides

Two things the driver needs are **already in the `ps4-linux` tree** and are
deliberately not duplicated in the patches:

- **`PCI_DEVICE_ID_SONY_BAIKAL_GBE`** (`0x90d8`) is defined in
  `include/linux/pci_ids.h`, alongside the other Baikal function IDs. The
  driver's ID table uses it directly.
- **`sc->bar2`** is mapped by `drivers/ps4/ps4-bpcie.c` in its own probe. The
  glue patch adds a reader for that window but does not need to map it.

On a tree that lacks either, the build will fail on an undefined symbol rather
than silently misbehave, so the failure mode is obvious.

`CONFIG_PS4_GBE=y`, the driver is builtin rather than a module, because the
rootfs it would be loaded from is often on the network it provides.

### Patches

- **`5.4.247-bpcie-glue-read32-and-gbe-wiring.patch`**, adds
  `bpcie_glue_read32()` and `bpcie_glue_bar2_read32()`, read-only accessors
  for the Baikal PCIe glue windows. The driver needs BAR 4 for per-part EFUSE
  calibration. **There is deliberately no write accessor for BAR 2**: that
  window holds a reset array shared with SATA and USB, and asserting the wrong
  entry would reset a block the running kernel depends on.
- **`5.4.247-baikal-sata-phy-trace-length.patch`**, the SATA fix above.
  Independent of the Ethernet driver; useful on its own.

### Module parameters

Each exists so a hypothesis stays falsifiable from the same image. All default
to the correct behaviour.

| parameter | default | effect of changing it |
|---|---|---|
| `residue` | `1` | `0` leaves the shutdown bit set, reproduces the dead link |
| `txring` | `255` | `256` restores the last-descriptor hang |
| `carrier` | `1` | `0` asserts carrier unconditionally, even when the PHY reports no link |
| `phy_ctrl` | `0` | `1` runs the firmware's PHY link-acquisition loop |

`ps4_gbe.txring=256` is the useful one: it reproduces the headline bug on
demand from a working kernel, which is what makes the fix verifiable by
someone other than the author.

---

## Known gaps

- **Why the last descriptor hangs is not understood.** The workaround is
  reliable and costs one descriptor; the cause is inferred, not established.
- **`0x0147.1e` bit4 has a measured effect and no name.** Ten Clause 45
  registers this firmware touches are named in no public MediaTek source.
- **No phylib integration.** MDIO is driven directly rather than exposed as a
  bus, so MediaTek's PHY driver cannot bind. This is the main thing between
  this driver and something upstreamable.
- **The SATA trace length is a constant, not a measurement.** The firmware
  reads it from per-console board data over ICC; the transport exists in
  `drivers/ps4/ps4-bpcie-icc.c` but the command for that read has not been
  decoded. Correct for this board, wrong for a board with different routing.
- **The SATA patch neutralises the bad read without removing it.** Both arms
  of the conditional now reach 6, but the placeholder is still loaded and
  tested. Deleting the conditional outright is the better change.

---

## Hardware

PS4 Slim CUH-2xxx, Baikal southbridge. The GBE is PCI function `00:14.1`,
device ID `104d:90d8`. The PHY is a MediaTek MT7531-family GPHY, identified
from its IEEE OUI and an exact PHY-ID match, confirmed by five independent
behavioural fingerprints.

Not tested on PS4 Pro or on the older Aeolia/Belize southbridge.

---

## Licence

GPL-2.0-only, including the documentation. See [LICENSE](LICENSE).

The specification and notes are the larger half of this project and the part
most worth reusing, so they carry the same licence as the code rather than a
second one.

## Author

Gabriel Stewart, 2026.

### How this was written

The reverse engineering, the hardware testing and every decision about what to
believe are mine. The driver was written and the notes were drafted with heavy
use of an AI assistant, working from UART captures and disassembly that I ran
and checked.

Saying so matters here for a specific reason. This repository asks to be
trusted on measurements, and the corrections log exists because several
confident claims in it turned out to be wrong. A reader deciding how much
weight to give a number should know how it was produced. Every figure quoted
in these documents came off hardware; the reasoning around them had a second
pair of eyes on it, not always a reliable one, which is the point of keeping
the log.
