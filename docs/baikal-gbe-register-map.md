# Baikal GBE — hardware register map and programming model

> **A note on references.** This repository publishes the specification, the
> driver and the corrections log. It does not publish the per-boot serial
> captures, the per-build records, or the working drafts those documents cite
> as evidence, because they are specific to one console and one machine.
> Citations below name them (`the bring-up log`, `attempt 30`, `build #46`)
> and the reasoning stands on its own; the raw logs are not part of this tree.


Hardware description for the gigabit Ethernet MAC in the PS4 Slim (CUH-2xxx) **Baikal**
southbridge, driven by Orbis as the `mtsc` (controller) / `mts` (port) pair.

This is the **specification the driver is written from**: register offsets, bit positions,
reset values, legal value sequences, descriptor layouts, and required ordering. Facts about a
device, not authored expression. The Linux driver is written from this document with no
decompiler open.

Derived from static analysis of the Orbis 9.00 kernel (`sha256 6be7c091…8f8644`), used only
to determine what the hardware requires; no derived text enters this repository. Values
marked ✓ were predicted from that analysis and then **confirmed against live silicon** on
2026-09-22 over UART, with the hardware still in the state Orbis left it. Nine predictions,
nine matches — the nine ✓ rows in §3. (Recorded as "seven" until 2026-09-26; the enumeration
was always nine.)

---

## 1. Device

| | |
|---|---|
| Vendor / device | `0x104d` (Sony) / Baikal GBE function |
| Register window | BAR0, 4 KB, at `0xc2000000` on the test unit |
| Register width | 32-bit throughout |
| Bus master | required |
| Interrupt | one line, level, shared-capable |
| PHY | on-die, reachable by Clause 22 and Clause 45 through reg `0x00` |
| PHY identifier | MII regs 2:3 read **`03a2:9441`** on live silicon (2026-09-25) |
| PHY part | **MediaTek MT7531-family gigabit PHY** — identified 2026-09-28, §1.1 |

Not a Marvell Yukon-2 — no shared register layout, and the `0xe00`–`0xfff` power block the
Aeolia/Belize `mskc` glue pokes reads back all zeroes here. `sky2` cannot drive this part.

### 1.1 The PHY is MediaTek — identified 2026-09-28

**`03a2:9441` is an exact match for `MT7531` in the mainline Linux MediaTek PHY driver.**
`PHYIDR1 = 0x03a2`, `PHYIDR2 = 0x9441`, giving model 4, revision 1. Taken together with the
corroboration below this is an identification, not a coincidence of sixteen bits.

**It is still not Realtek (`~0x001c`), not Marvell (`0x0141`), not Broadcom
(`0x0020`/`0x0362`)**, and `lspci` shows no Realtek device on the hardware under test — so the community
claims that PS4 ethernet is Marvell or an RTL8111H/G remain refuted. That was always the
answer to the community claim; it is now complete rather than merely negative.

#### The OUI decodes to MediaTek in the IEEE registry, independently of Linux — 2026-09-28

The earlier decode `00e8a5` was recorded as provisional because PHY-ID OUI encoding is
bit-reversed. **It is no longer provisional.** Reversing the bit order within each byte of the
as-transmitted field gives

```
as transmitted (what the ID registers pack)   00 e8 a5
bit-reversed per byte  ->  canonical OUI      00 17 A5
```

and **`00:17:A5` is assigned to MediaTek Inc in the IEEE MA-L registry.** That is a fact about
the silicon derived by arithmetic from registers 2 and 3 plus a public registry lookup, and it
does **not** depend on any Linux driver agreeing with it. The ID-pair match against
`MTK_GPHY_ID_MT7531` and this registry lookup are now two independent routes to the same
vendor.

> **Practical note for anyone reading a boot log.** ~~The driver prints the **raw
> as-transmitted field**, so the line reads `oui 00e8a5` and not `0017a5`.~~ True of **every
> log this project holds** — attempts 1–23, builds up to and including #38 — and those logs
> do not change, so read them that way: a reader who greps one for a MediaTek OUI will not
> find one, and a reader who sees `e8a5` will not recognise it. **From build #39 the driver
> prints both forms** — `oui 00:17:a5 canonical, 00e8a5 raw as-transmitted` — so no future
> log needs the reversal done by hand. The raw field is still printed, deliberately: it is what
> registers 2 and 3 actually contain, and it is what a register dump can be checked against.

#### How confident this is, stated plainly

**What is settled:** the vendor is MediaTek, by the registry and by the ID pair; the part is a
MediaTek GPHY whose Token Ring node layout, vendor page, register names and two tuning
*constants* match mainline's MT7531 support exactly.

**What is not:** that it is literally an MT7531 die. `0x03a29441` is an exact match for
`MTK_GPHY_ID_MT7531` and model 4 is unique to MT7531 among the MediaTek GPHYs mainline knows,
but a different MediaTek part sharing the same IP and model number would present identically
from here. Nothing found contradicts MT7531 and nothing available proves it beyond that. The
practical consequence is nil — every decode below rests on behaviour that was observed on
*this* part — and it is recorded so that a future contradiction is read as "different part,
same IP" rather than as the identification collapsing.

**Corroboration, which is what makes this an identification.** Five independent lines, none
of them the ID:

1. **The behavioural fingerprint — the strongest of the five, with row 5 below as its sharpest
   instance.** The six vendor writes in §4.1
   are not opaque triples; they are MediaTek's **Token Ring** debug-node accessor, with the
   third word a structured command carrying channel, node and data address. Decoded that way,
   **two of the six carry values identical to MediaTek's own constants**: `MrvlTrFix`
   100Kp/100Kf/1000Kp/1000Kf = 6/7/6/7, and `SSTrKp`/`SSTrKf` ×6 = 5/6/5/6/5/6. An access
   *idiom* matching is a strong hint; an idiom matching **and** two multi-field tuning words
   inside it matching bit-for-bit is a different order of evidence. Full decode in §4.1.

2. **Entry 5 is MediaTek's own tuning word, at MediaTek's own node, with one field retuned —
   and the low half is byte-identical.** This is the sharpest single datum in the section.
   Mainline `drivers/net/phy/mediatek/mtk-ge.c`, in `mtk_gephy_config_init()`, issues

   ```
   mtk_tr_modify(phydev, 0x1, 0xf, 0x17, SLAVE_DSP_READY_TIME_MASK,
                 FIELD_PREP(SLAVE_DSP_READY_TIME_MASK, 0x5e));
   ```

   — channel `0x1`, node `0xf`, data address `0x17`, field `SLAVE_DSP_READY_TIME_MASK` =
   bits **[22:15]**. Our §4.1 row 5 command word `0x8fae` decodes by the §4.1 formula to
   **bit15=1, bit13=0, ch=1, node=`0xf`, data=`0x17`** — the same accessor, the same channel,
   the same node, the same data address. And the datum:

   | | high (`0x12`) | low (`0x11`) |
   |---|---|---|
   | ours | `0x0006` | **`0x0671`** |
   | the same word with `SLAVE_DSP_READY_TIME` set to MediaTek's `0x5e` | `0x002f` | **`0x0671`** |

   **The low half is byte-identical**, and the two words differ in exactly one field —
   `SlvDSPready`, `0x0c` against `0x5e` — and in nothing else. (Checked by arithmetic here,
   not taken on trust: `0x00060671 >> 15 & 0xff = 0x0c`, and substituting `0x5e` gives
   `0x002f0671`. Both halves check.)

   **State the significance plainly.** Sony's ROM is not merely using MediaTek-shaped
   registers. It is writing **MediaTek's own tuning word at MediaTek's own debug node with one
   field retuned.** That is the same silicon programmed from the same vendor sequence — not IP
   resemblance, and not an access idiom that happens to be shared.

   **Provenance note.** This match was spotted several days before it was recorded, and was
   deliberately **kept out of the documentation** until the MediaTek side of it could be read
   from mainline rather than recalled from memory. See the corrections log in
   `notes/mts-init-sequence.md` — it is filed there as the provenance rule working, which is
   the one entry in that log that is not a correction.

3. **The vendor extended page is the same number.** The mainline driver uses vendor extended
   page **`0x52b5`** — exactly the page this ROM selects via `reg 0x1f = 0x52b5` before each
   of its `0x11`/`0x12`/`0x10` triples (§4.1). A sixteen-bit magic page number agreeing by
   chance is not plausible.

4. **Most of the Clause 45 registers this ROM touches are *named* in MediaTek sources**, with
   the devad matching where the source states one. This table is the canonical naming
   reference for the whole document; §4.2 uses the names and does not repeat them.

   | This document | MediaTek name | Field, where named | Source |
   |---|---|---|---|
   | `0x003c`.**07** | `MDIO_AN_EEE_ADV` (reg 60 in `MDIO_MMD_AN`) | whole register; writing 0 disables EEE advertisement | `mtk-ge.c`, openwrt-es/lede `mt7531.c` |
   | `0x00e0`.1e | `MTK_PHY_RG_ANA_CAL_RG5` | **`REXT_TRIM`**, `GENMASK(13, 8)` | `mtk-ge-soc.c` |
   | `0x0115`.**1f** | `MTK_PHY_RG_BG_RASEL` | `GENMASK(2, 0)` | `mtk-ge-soc.c` |
   | `0x0122`.1e | `MTK_PHY_RG_LPI_PCS_DSP_CTRL_REG122` | **`LPI_NORM_MSE_HI_THRESH1000`**, `GENMASK(7, 0)` | `mtk-ge-soc.c` |
   | `0x0172`.1e | `MTK_PHY_RG_CR_TX_AMP_OFFSET_A_B` | two halves, pairs **A and B** | `mtk-ge-soc.c` |
   | `0x0173`.1e | `MTK_PHY_RG_CR_TX_AMP_OFFSET_C_D` | two halves, pairs **C and D** | `mtk-ge-soc.c` |
   | `0x0189`.1e | `PHY_DEV1E_REG_189` — a placeholder name, **not** a functional one | bit0 `DESCRAMBLER_CLEAR_EN` | openwrt-es/lede `mt7531.c` |
   | `0x0268`.**1f** | `PHY_DEV1F_REG_268` — placeholder, but commented *"Adjust TX class AB driver 1"* | — | openwrt-es/lede `mt7531.c` |
   | `0x0012`/`0x0016`.1e | `MTK_PHY_TXVLD_DA_RG` / `TX_I2MPB_TEST_MODE_A2` | pair **A** | `mtk-ge-soc.c` |
   | `0x0017`/`0x0018`.1e | `TX_I2MPB_TEST_MODE_B1` / `_B2` | pair **B** | `mtk-ge-soc.c` |
   | `0x0019`/`0x0020`.1e | `TX_I2MPB_TEST_MODE_C1` / `_C2` | pair **C** | `mtk-ge-soc.c` |
   | `0x0021`/`0x0022`.1e | `TX_I2MPB_TEST_MODE_D1` / `_D2` | pair **D** | `mtk-ge-soc.c` |
   | c22 page 3, reg `0x1c` | `MTK_PHY_LPI_REG_1c` | **`SMI_DET_ON_THRESH`**, `GENMASK(13, 8)` | `mtk-ge-soc.c` |

   `0x0115` is the single register in the whole EFUSE sequence that sits at devad `0x1f`
   rather than `0x1e` (§4.2 row 2). That MediaTek independently places it in VEND2 is the
   strongest single datum here: an oddity in our table is explained rather than merely
   matched.

   **Three of these agree on a *field width or position* we derived independently**, which is
   a stronger check than a name matching an offset:

   - `0x00e0`.1e — §4.2 row 1 writes `f << 8` with `f` six bits wide, landing at bits **13:8**.
     `REXT_TRIM` is `GENMASK(13, 8)`. Exact.
   - `0x0115`.1f — §4.2 row 2 writes a **three**-bit field bare. `BG_RASEL` is
     `GENMASK(2, 0)`. Exact.
   - The 12 trim-pair registers — see §4.2, "the shift asymmetry", which is the sharpest of the
     three and gets its own argument there.

   `0x0122`.1e is a fourth of the same kind, one step weaker: our write is `0xffff`, which
   sets `LPI_NORM_MSE_HI_THRESH1000` to `0xff` — reported to be MediaTek's own value for this
   part. A full-`0xffff` write is not distinctive on its own, so this corroborates and does not
   prove.

   **`0x0189` and `0x0268` are only half-named.** `PHY_DEV1E_REG_189` and `PHY_DEV1F_REG_268`
   are placeholders spelt from the register number; they do not say what the register is. What
   is real is bit0 of `0x0189` — `DESCRAMBLER_CLEAR_EN` — **which our write agrees with**: our
   `0x0110` clears bit0, as MT7531's does, and additionally sets bits 4 and 8, which are
   unnamed anywhere. And `0x0268` carries the comment *"Adjust TX class AB driver 1"*, where
   **MT7531 writes `0x3aa` and we write `0x07f4`** — a genuine divergence on a TX driver
   setting, recorded in §8 rather than acted on.

5. **The driver performs EFUSE-based calibration of the same kinds this ROM does** — REXT,
   TX_OFFSET, TX_AMP/TX_R50 and TX_VCM. §4.2 was decoded with no knowledge of that driver and
   arrives at the same shape.

**Sources.** `drivers/net/phy/mediatek/mtk-ge.c`, `drivers/net/phy/mediatek/mtk-ge-soc.c` and
`drivers/net/phy/mediatek/mtk.h` in mainline Linux; the original driver posting at
<http://lists.infradead.org/pipermail/linux-mediatek/2021-May/024689.html>.

**What this does not settle.** The MediaTek driver is for the PHY embedded in MediaTek's own
SoCs and switches. Sony's part is on-die in Baikal and reached through this device's MDIO
window; nothing here says the two are register-identical outside the registers named above,
and no MediaTek driver code has been or will be copied. The names are used as *evidence about
this silicon* and as a decoding aid, nothing more.

#### The open list — a clean negative, and where the missing names probably live

**Ten Clause 45 registers and one Token Ring node that this ROM touches are named in no
MediaTek source found:**

```
0x0033.1e  0x0037.1e  0x0039.1e  0x0096.1e  0x00a2.1e
0x0107.1f  0x0147.1e  0x0171.1e  0x0174.1e  0x0175.1e
Token Ring  ch2 / node 0xd / data 0x0c      (§4.1 row 2)
```

`0x0174`/`0x0175` are unnamed but **not** undecoded — §4.2 establishes their two-halves layout
from the ROM's own masks and from read-back, independently of any name.

**`0x0147.1e` is now in the same position, and it is the most important register on this list.**
It is still unnamed in every source searched, and **bit4 now has a measured function**:
clearing it is what makes this part transmit and report link (attempt 30, below). So the list
still has ten unnamed entries, but one of them is no longer opaque — what is missing is a
vendor *name* and a mechanism, not a behaviour.

#### Five of the ten now have a measured pre-touch value — attempt 29, build #45

Read at the pre-touch point, before this driver's first PHY write
(`notes/hardware-bringup-log.md`, attempt 29):

| Register | Pre-touch value | What §4.2's calibration does to it | Net effect on this value |
|---|---|---|---|
| `0x0147.1e` | **`0x0010`** | **nothing — no init path writes it** | see below; this is the result |
| `0x0037.1e` | `0x0033` | `cl45 write 0x0037.1e = 0x0033` | **identical already** — no-op |
| `0x0096.1e` | `0x8000` | `cl45 write 0x0096.1e = 0x8000` | **identical already** — no-op |
| `0x0039.1e` | `0x000f` | `&= 0xb7ff`, `\|= 0x2000`, `&= 0xdfff` | bits 14/11 already clear; bit13 pulsed. Final `0x000f` |
| `0x0107.1f` | `0x4000` | `&= 0xefff` | bit12 already clear. Final `0x4000` |
| `0x0171.1e` | `0x0065` | `\|= 0x0180` then `&= 0xfe7f` | bits 8/7 pulsed. Final `0x0065` |

The five remaining entries — `0x0033.1e`, `0x00a2.1e`, `0x0174.1e`, `0x0175.1e` and the
Token Ring node — have pre-touch values recorded elsewhere (`0x0033.1e` and `0x00a2.1e` in
attempt 29's `cl45[pretouch]` and `gate[pretouch]` lines, `0x0174`/`0x0175` in §4.2's
read-back, the node in §4.1) and are not repeated here.

**Values, not names.** Every row above is a reading; **not one of these ten registers is
named**, and this table does not make any of them so. What it closes is a
different gap: the list had ten entries and zero measurements against it since 2026-09-28.

**One row has since gained a decode, by experiment rather than by naming — 2026-10-04.**
`0x0147.1e` bit4 is measured to inhibit transmit (below). That is one bit of one register out
of ten, obtained by writing it and watching the device, which is a method available for the
other nine and has not been applied to them. Nine of the ten remain values without meanings.

**The rows in the last three columns are arithmetic, not measurement.** The calibration was
not re-read after these three RMWs; the "net effect" column applies §4.2's own recorded
sequence to the measured starting value. Two of the three pulse a bit high and low again,
so they are no-ops **in final value** and not in time, and whether the transient does work
is not something a single pre-touch sample can say.

#### `0x0147.1e` — the shutdown residue, MEASURED 2026-10-04 at attempt 29

**It reads `0x0010`, bit4 set, at the pre-touch point.** That is exactly the value the ROM's
GBE shutdown routine `0xde1653f0` writes there as its last act on the PHY, **after** both DMA
engines have stopped and before a `DELAY(1000)` — §8 question 4 has the sequence and the
function. **Nothing in the ROM's init path writes this register and nothing in this driver
ever has**, so the value we read is not one either side programmed on the way up.

That is the **shutdown-residue** hypothesis (§8 class 3's successor) meeting its
pre-registered positive criterion: the state this driver inherits across a kexec is
post-shutdown, so this is a bit Orbis left set that **no comparison against Orbis's
*running* values could ever have caught**. §8 question 4 carries the full fact / inference /
caveat split.

> ~~⚠ **`0x0010` could be this register's reset default, and that would make the whole lead a
> clean negative.** The register is unnamed in every MediaTek source searched, it has been
> read exactly once ever, and **what bit4 does is not known**. Residue and reset-default are
> both consistent with the reading and nothing in this project separates them. Build #46
> clears bit4 to separate them.~~
>
> ### ⚑ SEPARATED, AND IT IS NOT A RESET DEFAULT — attempt 30, build #46, banner `#49`, 2026-10-04
>
> ```
> [    2.876262] residue: 0x0147.1e 0010 -> 0000 bit4 1 -> 0 cleared=1 ret=0
> ```
>
> **Bit4 is writable, it cleared, and the device then transmitted for the first time in thirty
> boots** — `tx=71/6014`, with `reg04` bit0 and `BMSR` bit2 both setting for the first time and
> holding for 362 s. **Attempt 29 is the control**: same `0x0010` on arrival, same PLL4 write,
> no residue clear, and `r04 = 0x00000b18` with `tx=0/0` for its whole boot. A reset default
> that nothing depends on would have cleared to no effect.
>
> **FACT: bit4 of `0x0147.1e` inhibits the transmit path on this part.** Measured, with a
> control, not inferred.
>
> ⚠ **What bit4 does *mechanically* is still unknown and must stay recorded as unknown** — not
> a power-down, not an isolate, not a lane quiesce, not a loopback. The register is still
> unnamed. A port must clear it; a document must not claim to know why that works.
> `notes/hardware-bringup-log.md`, attempt 30, and §8 question 4.

**Searched, and this is the negative worth recording**: mainline `mtk.h`, `mtk-ge.c`,
`mtk-ge-soc.c`, `mtk-phy-lib.c`, `mtk-2p5ge.c`, `drivers/net/dsa/mt7530.{c,h}`, and
openwrt-es/lede's MT753x tree. The list shrank from twelve to ten only by naming `0x0189` and
`0x0268`, both half-named at that; nothing else moved. This is not a search that has more to
give.

**Where the rest of the names almost certainly are, and it is a specific artefact rather than
a wish.** In the OpenWrt tree, `mt7531_internal_phy_calibration()` is a **stub — `return 0;`**.
MediaTek's actual MT7531 software calibration was never published. That is the routine whose
job is exactly what six of these ten registers are doing in our ROM, at exactly the position in
the sequence where our ROM does it. **The thing to hunt is a vendor SDK or firmware drop
containing a non-stub `mt7531_internal_phy_calibration()`** — not more passes over the public
trees, which have now been searched and do not contain it.

---

## 2. Access model

32-bit MMIO; `readl`/`writel` on the ioremapped BAR. Required ordering:

- Descriptor writes must be visible to the device **before** the owning bit is handed over.
- Ring base addresses must be programmed **before** the DMA engines are enabled.
- The reference clock (`0x7c`) must be set **before** the MAC is enabled.

### 2.1 Delays and poll bounds — the ROM's unit is the microsecond

Every timing figure quoted in this document comes from the ROM's `DELAY` at `0xde14e6b0`, and
**its argument is microseconds**. `DELAY` is FreeBSD's `delay_tc()`: it computes
`end = freq * n / 1000000`, where the divide is the `0x431BDE82D7B634DB` multiply-high plus
`shr rdx, 0x12` at `0xde14e6f5` / `0xde14e73c` — the standard divide-by-10^6 reciprocal. `freq`
is `tsc_freq` when `tsc_is_invariant`, otherwise the active timecounter's `tc_frequency`.

**Linux `udelay(n)` is the exact equivalent**, and `DELAY(n)` in this document should be read
as `udelay(n)` with no conversion.

Recorded because it was checked rather than assumed, and because the check paid: with the unit
settled, **every delay and every poll bound on the link path in this driver was verified against
the ROM and already matched it exactly**. Timing is not a candidate explanation for the link
anomaly in §4.4. The one bound that did *not* match was off the link path — the multicast poll,
`0xf4240` × 1 µs = **1 s** at `0xde16a070` and `0xde16a3b0`, against 10 ms in builds #23–#35,
100× short. It only ever expires on hardware that has already stopped answering, which is why
it survived thirteen boots unnoticed; fixed in build #36.

---

## 3. BAR0 register map

**This map is closed.** An exhaustive audit of `mtsc`/`mts` (2026-09-26) resolved **all 115
`bus_space` accessor sites** in `0xde165000`–`0xde16c800`, every one of them through the
single bus handle at `softc+0x3068`. There is no second BAR, no alternate window and no
hidden aperture, and the offsets below are the **complete** set the vendor driver touches —
no physical-address base register, no second descriptor-list pointer, no DMA context or
channel selector, no mode bit that changes how descriptor `word1` is interpreted. "Which
registers does this device have" is no longer an open question. Audit detail:
[`mts-init-sequence.md`](mts-init-sequence.md).

| Offset | Role | Live value | ✓ |
|---|---|---|---|
| `0x00` | SMI / MDIO command and data (§4) | `0x79498100` | |
| `0x04` | MAC control / status; link, speed and duplex (§3.6); `0x80003000` cleared at init | `0x00000b18` | |
| `0x08` | MAC config; `0x07597c00` OR-ed in at init | `0x0f597c00` | ✓ |
| `0x0c` | control; bit7 cleared at init | `0x03b0030c` | |
| `0x10` | RX filter control (§3.2) | `0x00000085` | ✓ |
| `0x14` | MAC address, high 2 bytes, byte-swapped (§3.1) | `0x0000XXXX` | ✓ |
| `0x18` | MAC address, low 4 bytes, byte-swapped (§3.1) | `0xXXXXXXXX` | ✓ |
| `0x1c` | Multicast hash command port (§3.3) | `0x0003207b` | |
| `0x30` | `= 0x10100` at init | `0x00010100` | ✓ |
| `0x34` | TX DMA control (§3.4) | `0x00000000` | |
| `0x38` | RX DMA control (§3.4) | `0x00000000` | |
| `0x3c` | TX ring current pointer (§3.5) | `0x10000d80` | |
| `0x40` | RX ring current pointer (§3.5) | `0x10004430` | |
| `0x44` | TX ring base address (§3.5) | `0x10000000` | |
| `0x48` | RX ring base address (§3.5) | `0x10004000` | |
| `0x50` | Interrupt status, write-1-to-clear — and the write **does** clear bit6; nothing else does (§3.7) | `0x00000000` | ✓ |
| `0x54` | Interrupt **MASK**. **A set bit MASKS its source** (§8 question 1, closed 2026-09-27) | `0x00001018` | ✓ |
| `0x74` | `= 0x2277` at init | `0x00002277` | ✓ |
| `0x78` | bit0 cleared at init | `0x00000000` | |
| `0x7c` | Reference clock, Hz; `= 25000000` | `0x017d7840` | ✓ |
| `0x9c` | error/link handling; bit6 **pulsed** on status `0x500000` — cleared, then set again | `0x0000006f` | |
| `0xac` | `= 9` at init | `0x00000009` | ✓ |
| `0x140` | secondary MAC high 2 bytes; bit31 = enable | `0x00000000` | |
| `0x144` | secondary MAC low 4 bytes | `0x00000000` | |
| `0x1bc` | pattern-match filter: data port | `0x00000000` | |
| `0x1c0` | pattern-match filter: index; bit7 = write strobe | `0x00000000` | |
| `0x1c4` | pattern-match filter: mode (0 / 1 / 3) | `0x00000000` | |
| `0x1c8` | pattern-match filter: lengths `(len<<8)\|len`; bits 31:30 = enable | `0x00a00000` | |
| `0x1d0` | pattern-match filter: status; bit0 = ready | `0x00000000` | |
| `0x1d4` | `= 1` at init | `0x00000001` | ✓ |
| `0x200` | `= 0` — first write of the whole init sequence | `0x00000000` | ✓ |
| `0x204` | interrupt coalescing timer; armed with `0x10001388`, disarmed with `0` (§7.4) | `0x00000000` | |

**Where "Live value" comes from, and the whole-table re-read that confirms it.** The column
was originally the surviving artifact of the phase-1 `sky2` survey (2026-09-22,
`notes/kernel-re.md`), which dumped MMIO `0x000`–`0x1FF` from inside `sky2_probe()` before
anything touched a device the console had just been running Orbis on. **Attempt 25 (build
#41, banner `#44`, 2026-10-02) re-read all thirty-two from this driver**, at probe and again
after `gbe_hw_init()` returns, and the two readings are ten days and eleven builds apart:

- **Every value the column already carried is unchanged**, except the two ring *current*
  pointers `0x3c` and `0x40` — which §3.5 says the device advances, so they record where
  Orbis's rings stood when it stopped and have no reason to repeat. The ring **bases**
  `0x44`/`0x48` are identical, which is the signature §3.5 predicts and a check on §3.5.
- **The eight offsets that had no value now have one**, measured: `0x1c`, `0x34`, `0x38`,
  `0x9c`, `0x140`, `0x144`, `0x200`, `0x204`. `0x200` and `0x204` sit **outside** the survey's
  window and had never been read at any time. `0x204` reading `0x00000000` confirms §7.4's
  disarm; `0x34 = 0x38 = 0` says Orbis **stopped both DMA engines** in its teardown; `0x9c`
  and `0x1c` are first-ever values and are **not decoded** — see the bring-up log, attempt 25.
- **`postinit` is byte-identical to `pretouch` on 31 of 32.** The exception is `0x00`, the
  MDIO port, which changes because the driver did MDIO. So **every register-value write in
  §7.1 is a no-op on arrival** — the per-step table is in the bring-up log, attempt 25 — and
  the consequence is in §8 C.

**Attempt 27 (build #43, banner `#46`, 2026-10-03) re-reads all thirty-two again and agrees on
every offset but one.** `pretouch` and `postinit` are identical to each other and to attempt
25's, with the same single exception (`0x00`, the MDIO port, `79498100 → 10408000` — the same
pair of values attempt 2 saw). The one offset that differs **between boots** is `0x1c`, which
reads `0x0003206d` here against attempt 25's `0x0003207b`. It is the multicast hash **command
port** (§3.3), whose low byte this document explicitly declines to decode, so the delta is
recorded and not interpreted — but **treat `0x1c`'s "Live value" above as a sample rather than
a constant**, which is not true of any other row in the table. `0x08 = 0x0f597c00` at both
sample points, bit27 included, is the reading that empties §8 C's class 1.

### 3.1 MAC address — `0x14` / `0x18`

Byte-swapped on the way in: high half is a 16-bit swap of the first two address bytes, low
half a 32-bit swap of the remaining four. A secondary address at `0x140`/`0x144` uses the
same encoding, enabled by bit31 of `0x140`; Orbis programs it only when a per-revision
configuration flag is set.

### 3.2 RX filter control — `0x10`

**bit4 — promiscuous.** Set to receive all frames, clear for normal filtering.

Init writes `(v & 0xffffff6e) | 0x81`, which clears bits 0, 4 and 7 then sets bits 0 and 7,
leaving **bit4 clear**. This agrees independently with the filter path, which sets exactly
bit4 for promiscuous mode — two unrelated paths assigning the same bit the same meaning.

### 3.3 Multicast hash — `0x1c`

A command port, not a memory-mapped table. Operations are distinguished by the value written:

| Write | Meaning | Completion |
|---|---|---|
| `0x80000000` | clear the entire hash table | poll until **bit17 (`0x20000`) set** |
| `0x7000 \| index` | add hash `index` (8-bit) | poll until **bit14 (`0x4000`) clear** |

Each poll is bounded at 1,000,000 iterations of a 1 µs delay.

**Hash function.** Ethernet CRC-32 — polynomial `0x04C11DB7`, MSB-first, initial value
`0xFFFFFFFF`, **no final inversion and no reflection** — over the 6 address bytes. The 8-bit
table index is assembled from two **disjoint** slices of that CRC:

```
index[7:3] = crc[31:27]
index[2:0] = crc[2:0]
```

Unusual — most MACs use one contiguous slice. Both slices are required.

"Receive all multicast" is not a mode bit: it is implemented by writing all 256 index values,
`0x7000 | 0..0xFF`, in a loop.

### 3.4 DMA engine control — `0x34` (TX), `0x38` (RX)

Identical layout **in the bits you write**. Whether they read back the same way
**depends on the state the part is in** — read "What `0x34` does after RUN" at the
end of this section before quoting bit0 or bit3 anywhere. The short version, because
this section was wrong about it for four boots: with the PHY carrying the shutdown
residue (§1.1, §8 question 4) `0x34` latched bit0 and never asserted bit3; with the
residue cleared it does neither — it behaves exactly like `0x38`.

- **bit0 — run.** Set to start the engine. **On `0x38` it does not read back.**
  Every post-RUN read behind that claim is a read of **`0x38`**; `0x34` behaves the
  opposite way and is recorded below. `0x38` reads
  `0x0` before RUN and `0x8` after, never `0x1`. Confirmed on hardware 2026-09-27
  (attempt 13) by **three separate post-RUN reads**, all `0x00000008` and none ever
  `0x00000009`: the `engines:` line at t=97.903428, `isr[2]` at t=97.905416, and the
  second ring dump at t=97.910482 — the last of which also shows 25 descriptors'
  worth of pointer movement, so the engine was demonstrably live while bit0 read as
  clear. The `0x0` reading is the **pre-RUN** sample (the first dump's header, taken
  before `DMA_RUN`) and is not evidence about bit0 under load — it is the value this
  section predicts anyway. **Treat bit0 as write-only on both registers and never poll
  it to decide whether the engine is running.** ~~That instruction is correct for `0x38`
  and **wrong for `0x34`**, where bit0 reads back set and stays set.~~ **Corrected
  2026-10-04, attempt 30**: `0x34` bit0 reads back set only while the PHY carries the
  shutdown residue. In the boot that transmits it reads `0x00000000` at `probe`,
  `after-reset`, `after-base` **and `after-run`**, and `0x00000000` on all 66 `txwatch`
  samples — write-only, exactly like `0x38`. See below.
- **bit1 — reset.** Write 1; the bit self-clears when the reset completes. Poll until clear,
  bounded at 1,000,000 iterations of a 1 µs delay.
- **bit2 — poll demand.** The doorbell, and the only one. Write after handing
  descriptors to the device.
- **bit3 — engine active.** Read-only status. On `0x38` it appears once RUN is written
  and stays set while the engine is live. ~~**On `0x34` it has never been observed set,
  on any boot.**~~ **OBSERVED SET on `0x34` — 2026-10-04, attempt 30 (build #46, banner
  `#49`), the first boot that transmitted.** All eight `xmit[]` lines sample `0x34`
  either side of the doorbell and read `txctl=0x00000000->0x00000008`: `gbe_doorbell()`
  writes bit2 and **bit3** is what comes back. On `0x34` the assertion is **transient**
  — it reads `0x00000000` again at every `txwatch` sample, i.e. when the engine is idle —
  where on `0x38` it stands continuously because RX is always armed. Same meaning,
  *active*, on both registers. It was read as "started and immediately suspended
  itself" for twelve boots; that reading was wrong — see §8 A.

  ⚠ **"Transient" is true of a healthy engine only. On a stalled one bit3 stands set
  indefinitely while nothing moves** — 48 observations at attempt 31, the longest holding 46 s
  with `txcur` frozen and `tx_packets` flat. **Active does not mean making progress, and bit3
  must not be polled to decide whether TX is healthy.** See "`0x34` bit3 is 'active', and
  active is NOT 'making progress'" at the end of this section.

  **Further confirmed on hardware 2026-09-27 (attempt 14), over two minutes rather than
  milliseconds.** `0x38` read `0x00000008` on the `engines:` line, on the boot's only
  interrupt, at poll entry, and on **all twenty-four** samples of a 5 s register poll from
  t=103.459923 to t=221.219986 — twenty-seven post-RUN reads, never once `0x00000009`.
  Across those samples the engine demonstrably received: `GBE_RXRING_CUR` advanced through
  **thirteen distinct values**, and one drain had already delivered 18 frames to the stack.
  So bit3 is set on an engine that is provably moving data, sustained, and bit0 still does
  not read back. Bit3 means *active*.

  **The ROM neither confirms nor refutes this, and cannot.** A pass over every `0x34`/`0x38`
  access in the ROM (2026-09-27) found it **never reads bit0 or bit3 anywhere**. The only bit
  it ever reads back is bit1, and only in the stop path, waiting for the reset to self-clear.
  So the bit0 and bit3 decodes above rest **entirely on hardware observation** — there is no
  second source for them and there never will be one from this firmware. Recorded because
  "the ROM agrees" has been offered as corroboration for register decodes in this project
  before, and for these two bits it is not available. **That note covers the `0x34`
  asymmetry below as well** — the ROM reads neither bit on either register, so there is
  no second source for any of it and there never will be.

#### What `0x34` does after RUN — observed 2026-10-03, attempt 26 (build #42, banner `#45`)

**Attempt 26 is the first time `0x34` has been read after RUN on any boot.** Build #42's
sampler took the register at four stage points inside `gbe_start_engines()` and ten more
times out to +4880 µs after the RUN write. `0x34` does the **opposite** of what the
twenty-seven `0x38` reads above established:

```
es[pre-rst]    034=00000000 038=00000000 050=00000000 09c=0000006f 1c8=00a00000
es[post-rst]   034=00000000 038=00000000 050=00000000
es[post-base]  034=00000000 038=00000000 050=00000000
es[post-run]   034=00000001 038=00000008 050=00000040
es[poll] +0 → +4880us   034=00000001 038=00000008 050=00000000   (unchanged, all 10)
```

| | `0x38` (RX) | `0x34` (TX) |
|---|---|---|
| after RUN | `0x00000008` — bit0 **clear**, bit3 **set** | `0x00000001` — bit0 **set**, bit3 **clear** |
| bit0 across the window | — | **set on all ten poll samples** to +4880 µs, and `txctl=0x00000005` hundreds of ms later (bit2, the doorbell, on top of it) |
| bit3 across the window | set, on an engine demonstrably moving data | **never set, in any sample** |

So on this part:

- **`0x34` bit0 latches at the RUN write and persists.** It is not write-only here.
- **`0x34` bit3 never asserts.** TX latches RUN without ever becoming active, while RX
  in the same function microseconds earlier asserts bit3 immediately and demonstrably
  receives — 54 packets on 52 interrupts with `kick=0` in that same boot.

**The reading, and it is labelled a reading rather than a fact:** on TX the RUN request
is **latched but unconsumed** — the engine holds a pending start it never acts on —
where RX consumed its start immediately. That is a **different state from "started and
then stalled"**, and attempt 26 is the first evidence that distinguishes the two. Like
the bit0 and bit3 decodes above it rests **entirely on hardware observation**, and the
ROM cannot corroborate it.

**`0x50` stays quiet through all of it.** `0x00000040` once at `post-run` — bit6, RX
completion, which is RX arriving as it should — and zero on every sample after.
**No TX error source appears at any point**, so the device reports nothing wrong about a
TX engine that is doing nothing. Consequence in §8 B.

**`0x9c` and `0x1c8` do not participate in engine start.** `0x9c = 0x0000006f` and
`0x1c8 = 0x00a00000` at `pre-rst`, **unchanged in every later sample**, so both are
**struck from the engine-start candidate list**. Neither is decoded (§3's table, §8
question 2); they are simply not part of this sequence.

**`pre-rst` and `post-rst` are both `0x00000000`, which closes §8 question 6
negatively** — the bare `DMA_RESET` full-word store destroys nothing, because there is
nothing in `0x34` to destroy. `post-base` equals `post-rst`, so writing `0x44`/`0x3c` has
no side effect on `0x34` either. **The TX reset must not be removed on that hypothesis**;
§8 question 6 has why.

#### What `0x34` does after RUN **in a boot that transmits** — attempt 30, build #46, banner `#49`, 2026-10-04

**The reading above is state-dependent, and this is the state that matters.** Build #46
clears bit4 of Clause 45 `0x0147.1e` — the shutdown residue (§8 question 4) — at the top of
PHY bring-up, and nothing else about the engine-start path changed. The same sampler at the
same four stage points:

```
es[pre-rst]    034=00000000 038=00000000 050=00000000 09c=0000006f 1c8=00a00000
es[post-rst]   034=00000000 038=00000000 050=00000000
es[post-base]  034=00000000 038=00000000 050=00000000
es[post-run]   034=00000000 038=00000008 050=000000c0
es[poll] +0 → +4880us   034=00000000 038=00000008 050=000000c0   (unchanged, all 10)
txctl[4]: probe=0x00000000 after-reset=0x00000000 after-base=0x00000000 after-run=0x00000000
xmit[1..8]:  txctl=0x00000000->0x00000008        (sampled either side of the doorbell)
txwatch[1..66]: txctl=0x00000000
```

| | Attempt 26 / 29 — residue present | Attempt 30 — residue cleared |
|---|---|---|
| `0x34` after RUN | `0x00000001` — bit0 **set**, bit3 clear | `0x00000000` — **neither** |
| `0x34` bit0 later | set for the rest of the boot, `0x5` with the doorbell on top | **never set, in any of 66 samples** |
| `0x34` bit3 | never set, in any sample | **set on all 8 doorbell read-backs** |
| `0x50` at `post-run` | `0x00000040` — bit6 only | `0x000000c0` — bit6 **and bit7** |
| TX | `tx=0/0` | `tx=71/6014`, `prod==cons`, `drop=0 err=0` |

So on this part, with the residue gone:

- **bit0 is write-only on `0x34` as well.** The latch was not how this register behaves; it
  was how this register behaves while the transmit path is inhibited.
- **bit2, the doorbell, does not read back.** Write bit2, read bit3. (With the residue
  present bit2 *did* read back and stick — `0x00000005` — which is what `gbe_doorbell()`'s
  clear-then-set exists to work around and is still the right thing to do.)
- **bit3 asserts transiently**, for as long as the engine is working, and reads clear at
  idle.

> **The "latched RUN that never activates" anomaly is not explained — it is absent.** It was
> the defining symptom of the blocker from attempt 26 to attempt 29 and it does not occur in a
> boot that transmits. §3.4's bullets above are corrected rather than annotated; the attempt
> 26 block is kept because it is the correct description of the inhibited state and because
> the contrast between the two tables is the measurement.

**One thing in this boot is unexplained and is recorded as such.** `0x50` bit7 (TX
completion) is already set at `isr[2]` (t=20.840) and at `es[post-run]` (t=20.917), **before
the first `xmit[]` at t=21.334** — before any descriptor had been handed over, with the TX
ring initialised to `word0 = OWN` on all 256 entries so there was nothing to complete. The
four later standalone `status=0x00000080` interrupts each follow a transmit and precede a
reclaim, which is bit7 behaving as decoded; the first assertion is not accounted for.

#### `0x34` bit3 is "active", and active is NOT "making progress" — attempt 31, build #48, banner `#51`, 2026-10-09

**Bit3 stands set indefinitely on a transmitter that is doing nothing**, and that is the
qualification the "transient on `0x34`" reading above needs. Attempt 31 stalled the TX engine
twelve times at index 255 (§5.4, §8 question 8) and sampled `0x34` throughout:

```
txwatch[12]: txctl=0x00000008 txcur=0x7ffffff0 isr=0x00000000 imr=0x00001018 prod=49 cons=255 free=206 txd w0=7000004d w1=7fe0d780 w2=00000000 w3=00000000 tx=255/40875 drop=0 err=0 q=0 carrier=1
txstuck[1]:  tx=255 frozen with 69 descriptors outstanding, txctl=0x00000008 txcur=0x7ffffff0 -- running the ROM's TX restart
txrst[1]:    txctl=0x00000000 txcur=0x7ffff000 base=0x7ffff000 errctl=0x0000006f
```

| | value | samples |
|---|---|---|
| `0x34` with `txcur` frozen at `base+0xff0` | `0x00000008` — **bit3 set** | **36 of 62** `txwatch`, plus all 12 `txstuck` reads = **48** |
| `0x34` with `txcur` advancing inside the ring | `0x00000000` | the other **26** `txwatch` samples |
| `0x34` after the §7.4 restart | `0x00000000` | all 4 logged `txrst[]` |

**So bit3 means the engine is mid-operation, not that it is completing anything.** Across the
36 stalled samples `txcur` does not move off `0x7ffffff0`, `tx_packets` does not advance, and
the longest single stall holds bit3 set for **46 seconds**. Attempt 30's reading — transient,
asserting on the doorbell read-back, clear at idle — is reproduced by the 26 healthy samples
and is still correct; what it cannot be read to mean is *progress*. **A device that had
declined the descriptor would be idle, and those 26 samples are what establish that idle reads
`0x00000000` on this register.** Anything that polls bit3 to decide whether TX is healthy will
be told "active" by a dead engine.

**Bit2, the doorbell, still does not read back** in this boot either: `0x00000008` never
appears as `0x0000000c`, so what comes back from a `gbe_doorbell()` write is bit3 alone,
exactly as recorded above for the residue-cleared state.

**`0x9c` after the restart reads `0x0000006f`** on all four logged restarts — §7.4 step 1
pulses bit6 clear and then set, and the value afterwards is **the same `0x0000006f` attempt 25
read at `bar0[pretouch]`**, before this driver had written anything. The pulse is a no-op in
final value.

### 3.5 Ring pointers — `0x3c`/`0x44` (TX), `0x40`/`0x48` (RX)

Each ring has a **base** and a **current pointer**. At startup both are programmed to the
same value: base first (`0x44`, `0x48`), then current (`0x3c`, `0x40`). The device advances
the current pointer as it consumes descriptors.

Both are 32-bit and take bus addresses, so **the rings must live in the low 4 GB**. The live
values are `0x4000` apart, matching the two coherent allocations, and both observed current
pointers are 16-byte aligned — independently confirming the 16-byte descriptor size.

### 3.6 Link status — `0x04`

Decoded at instruction level, 2026-09-24. `0x04` is a control register *and* the link status
register; the bits below are the status half.

| Bits | Meaning |
|---|---|
| 0 | link up |
| 3:2 | speed — `0` = 10, `1` = 100, `2` = 1000 Mbit/s. **Code `3` is not a speed**: the ROM's own decode gives it no speed bit at all, so there is no fourth speed in this field |
| 4 | duplex — set = full |
| 8 | **meaning undecoded.** The ROM carries it through its link-status word as a flag (§3.6.1) and never branches on it. Set in our live `0x00000b18` |

**Speed code `2` = 1000 Mbit/s is corroborated from a second, independent direction —
2026-10-03.** The link-state handler at `0xde1681b0` (§3.6.1) decodes `0x04` into a packed
status word, twice over with byte-identical logic: bit0 → word bit0, bit4 → word bit1
(duplex), bit8 → word bit2, and bits 3:2 as a code selecting `0x10` / `0x20` / `0x40` for
codes 0 / 1 / 2, with code 3 selecting nothing. The second copy ORs in the constant
`0x474700`, which is `(0x47 << 8) | (0x47 << 16)` — and `0x47` = `0x1 | 0x2 | 0x4 | 0x40` is
*exactly* the word this same function computes for **link up, full duplex, bit8 set, speed
code 2**. The ROM therefore hardcodes "good gigabit link" as this decode's output for code 2.

**What that says about our register.** `0x00000b18` carries code 2, bit4 and bit8 — so the ROM
would compute precisely `0x47`, its own gigabit constant, for our `0x04` if bit0 were set.
**The MAC is reporting 1000 Mb/s full duplex with link down**, not reporting nothing. Bit0 is
the only part of this register that disagrees with a healthy gigabit link.

This is what the ~70 ms poll in the suspend path is waiting for: bit0, link-up. The live value
`0x00000b18` therefore reads as link **down** (bit0 clear) with the hardware in the state Orbis
left it, full duplex, speed field `2` (1000).

#### ⚑ CONFIRMED POSITIVELY — attempt 30, build #46, banner `#49`, 2026-10-04

Every decode in the table above had been confirmed only by the ROM's arithmetic and by bit0's
unexplained *absence*. In the boot that transmits, all three read the other way:

```
[   20.815086] linkchg[1]:    status=0x00000004 reg04=0x00000b19 up=1 speed=2 fdx=1
[   21.021800] linkctrl[1]:   reg04=0x00000b19 up=1 speed=2 fdx=1 bmsr=796d/796d ... stat1000=7800 phylink=1 aneg=1 carrier=1
[  362.012285] linkctrl[112]: reg04=0x00000b19 up=1 speed=2 fdx=1 bmsr=796d/796d ... stat1000=7800 phylink=1 aneg=1 carrier=1
```

- **`reg04 = 0x00000b19` — bit0 SET — on all 114 post-probe samples**, t=20.815 to t=362.012.
  The only `0x00000b18` in the boot is the pre-touch read, taken before any write, when the
  link genuinely was down. **Bit0 = link up now has a positive confirmation** and not only an
  unexplained zero.
- **`speed = 2`, `fdx = 1` on all 114.** Together with bit8, that is exactly the input for
  which the ROM's link-state handler computes its hardcoded gigabit constant `0x47`
  (§3.6.1) — so the speed-field and duplex decodes, and the `0x47` cross-check, are confirmed
  on silicon rather than derived from the ROM alone.
- **The PHY agrees for the first time.** `bmsr = 0x796d`, bit2 set on **both** reads of a
  latch-low bit, on all 113 PHY samples; `stat1000 = 0x7800` on all 113. The PHY-side and
  MAC-side link bits read the same thing.
- The switch agrees too, which is the first external observation in the project: the OpenWrt
  box reports the port at `1 GbE` with live traffic both ways
  (`notes/hardware-bringup-log.md`, attempt 30).

**And the `0x04` → carrier decode is now confirmed by a live interrupt as well as by polling —
attempts 30 to 34.** The `linkchg[1]` line above is the `0x50` bit2 handler: bit2 asserted,
the handler read `0x04`, found bit0 set, and the link was up for the rest of the boot. **So
this register's bit0 has a second, independent confirmation path**, the edge-driven one the ROM
itself uses (§3.6.1, §3.7.2), and not only the 3 s poll. Attempt 32 is the sharpest case — its
link came up at **t=99.39**, 80 s after `gbe_open()`, and bit2 is what reported it; a driver
that only read `0x04` at open would have missed the transition entirely. **A port must
therefore service `0x50` bit2 and must not rely on a single read of `0x04` at open.** Attempt
33 is the first boot where that mattered in practice: with the carrier honest (driven from bit0
rather than asserted unconditionally), the ISR's `netif_carrier_on()` is the call that brought
the carrier up, 208 ms before `gbe_open()` reached its own carrier line.

> **⚑ Reproduced at attempt 34, and the reproduction is a no-regression result rather than new
> evidence.** `reg04 = 0x00000b19` on **172 of 173** post-probe reads — the one `0x00000b18`
> is the `pretouch` read at t=2.538241 — `bmsr = 796d/796d`, `speed=2 fdx=1`, `phylink=1`,
> `carrier=1` and exactly one `linkchg[]`, out to t=526.714988. That boot carried **four
> builds in one flash** (#50, #51, #52 and #53's `ahci.c` change), so what it establishes is
> that none of them disturbed this register's behaviour. **It establishes nothing about load:
> 79 packets transmitted and 100 received in 527 s, an idle session.** The load evidence is
> still attempt 32's and attempt 33's. `notes/hardware-bringup-log.md`, attempt 34.

**What made the difference is not in this register.** Bit4 of Clause 45 `0x0147.1e` — the
shutdown residue — was cleared, and `r04` went `0x00000b18` → `0x00000b19` 39 ms later with
only that write and the PLL4 bypass write in between; attempt 29 performs the PLL4 write
without the residue clear and `r04` stays `0x00000b18` for its whole boot. §8 question 4.

**The paragraph below is the record of the ten boots where bit0 read down. It is kept because
"do not gate driver behaviour on bit0" is still the right instruction** — a port must still
decide for itself when to sample `0x04`, and a probe-time `0x00000b18` is still not evidence
of a dead link.

**Bit0's decode is right; *when* it is valid is not established, and "not yet valid" no longer
covers the observations.** `0x00000b18` is the value at **probe** — but attempts 20 and 21 read
`0x04` again on a 3 s tick for the whole boot, 12 and 21 samples respectively, and it is
`0x00000b18` **every time**, bit0 clear, while the driver receives 528 and 50 frames on real
interrupts. Earlier boots received 535 (attempt 16) and 542 (attempt 17) under the same
probe-time value. So bit0 is not merely early: it reads down on a settled, receiving interface,
minutes in. §4.4 has the PHY-side half of the same picture — `BMSR` bit2 also clear, on both
reads of a latching-low bit, with the 1000BASE-T PCS fully trained and a zero idle error
count on every settled sample.

**Do not treat a clear bit0 as evidence the link is down**, at probe or at any other time, and
do not gate driver behaviour on it. The ROM gives no help: its link-state handler at `0xde1681b0`
is reached from `0x50` bit2 and **trusts bit0 directly**, with no settling delay anywhere in the
path. A port must therefore decide for itself when to sample `0x04`, and must not treat a
probe-time `0x00000b18` as evidence the link is down.

**It does re-read, and that is not a settling delay.** This paragraph previously said "no
settling delay and no re-read"; the re-read half is wrong. On one of the three device revisions
the handler reads `0x04` a second time before publishing (§3.6.1), and `mts_intr` then reads it
a **third** time immediately afterwards for its own event push (§7.4). All three reads are
back-to-back on the same interrupt with nothing between them, so none of them is waiting for
anything — the conclusion stands, the mechanism was misstated. Corrections log.

`0x04` is **written in exactly one place in the whole ROM** — §7.1 step 9 — and always as
`saved & 0x7fffcfff`.

**That claim has now survived two independent passes, and it has been load-bearing in both.**
The ROM has exactly two readers of `0x04`, and **neither of them writes it**:

| Reader | Read | Writes to `0x04` |
|---|---|---|
| `SceGbeMtsPhyCtrl` @ `0xde169aa0` | one read, `0xde169d11`/`18`/`30` — the function's **only** MMIO access of any kind in 1153 bytes | **none** |
| link-state handler @ `0xde1681b0` | two reads, `0xde1681ca`/`dd` and `0xde168252`/`62` — the function's **only** two MMIO accesses in 272 bytes | **none** |

Read 2 in the handler resolves to the same `struct resource *` as read 1, so it is a fresh
re-read of the same register on the same device and not a sibling port. Recorded explicitly
because each pass was dispatched to test a hypothesis that needed a second writer to exist, and
both came back with the same clean negative. **Do not add a `0x04` write.** Writing
speed/duplex/link into this register would be an invention, not a port.

### 3.6.1 What the ROM does with `0x04`, and the TX gate it does not contain

**Decoded 2026-10-03 from all 272 bytes of the link-state handler at `0xde1681b0`.** This is
the ROM's single "recompute and publish link status" entry point. It is reached from three
places: the `0x50` bit2 branch of the ISR, an `mts_ioctl` path immediately after a
`BMCR = 0x800` PHY power-down, and the suspend path. Its whole hardware contact is the two
`0x04` reads in §3.6.

**Three outcomes, selected by the two per-revision flags.** `softc+0x30dc` and `softc+0x30e0`
are set from the revision tables at attach — `{0,1,1}` and `{0,1,0}` respectively — and between
them they pick which of three things this function does:

| Revision index | `0x30dc` | `0x30e0` | Behaviour |
|---|---|---|---|
| 0 | 0 | 0 | publishes the bare decoded word (zero when bit0 is clear) |
| 1 | 1 | 1 | **publishes nothing, ever** — both the link-up and link-down paths return immediately |
| 2 | 1 | 0 | re-reads `0x04`, publishes `0x474700 \| word` |

So the `0x474700` constant of §3.6 is reachable on **exactly one** revision, and on revision
index 1 this ROM never reports link state from here at all. `softc+0x30e0` is also the flag that
decides whether `SceGbeMtsPhyCtrl` is started, so the two behaviours line up: revisions 0 and 2
run the PHY thread *and* publish link state; revision 1 does neither.

> **The ROM does not gate transmission on link state. There is no such gate in the firmware.**

`0xde1681b0` touches **no** `0x34`, no `0x3c`/`0x44`, no TX ring, no `tx_prod`/`tx_cons`
(`softc+0x3058`/`0x3060`) and no OACTIVE-style flag. It was the last candidate site in the ROM
for a software link-state push into the transmit path — §3.6's other reader, the PHY thread, had
already come back empty — and it is empty too. Taken with `mts_tx`/`mts_txeof` making zero
register accesses and the absence of any MAC transmitter-enable, **the whole "we are missing a
software link handshake that would let TX run" hypothesis family is closed.** There is nothing
to implement, because there is nothing there.

**The scope of that, stated precisely, because the distinction is the whole value of the
result.** What is closed is the **software** gate: the ROM contains no link state pushed into
the TX path, and no port can be missing one. A **hardware** gate inside the MAC — silicon that
holds the transmitter off while its own link state machine reads down — remains entirely
possible, and it is **unfalsifiable by reading firmware**, because the silicon would not need
the ROM's help to do it. The correct reading of this section is *"the ROM contains no such gate,
and we can no longer cite the ROM for one"* — **not** "TX is not gated on link". Anything in
this repository that cites firmware as the reason the transmitter waits for link is unsupported;
see the corrections log.

### 3.7 Interrupt status — `0x50`, and what actually acknowledges a source

**Write-1-to-clear, and that is sufficient. Confirmed on hardware 2026-09-27 (attempt 15).**
The ISR reads `0x50` and writes the value straight back (§7.4); the source deasserts.

```
[   20.595723] eth0: isr[1]: status=0x00000040 after-w1c=0x00000000 imr=0x007bfffe txctl=0x00000001 rxctl=0x00000008
[   20.696481] eth0: ack[1]: pre=0x00000000 postdrain=0x00000000 postbell=0x00000000 done=2
```

Bit6 is RX completion. It was asserted, the status write cleared it to `0x00000000`, and the
NAPI poll that followed read `0x00000000` at poll entry as well — an independent read, in a
different function, 100 ms later. **The write is the acknowledge.**

**This section previously said the opposite, and the question it posed is dissolved rather
than answered.** Attempt 14 read `after-w1c=0x00000040` and that was recorded as "the write
does not clear bit6", with three candidate mechanisms and none adopted. All three are now
refuted, and the readback that started it has a duller explanation: the write and the
readback are two separate register accesses, and in attempt 14 there were 18 completions
pending with the engine still receiving, so the source had every opportunity to re-assert in
between. Bit6 was never unclearable. What kept it asserted for the rest of that boot is that
**nothing ever wrote `0x50` again** — the ISR is the only writer, and the ISR ran once. The
persistence was a *consequence* of the interrupt famine, not its cause. The cause was the
mask (§8 question 1).

**The measurement, and what each of the three candidates died of.** Build #28 sampled `0x50`
at three points in the poll — before the drain, after the drain, and after the `DMA_POLL`
doorbell — precisely to separate them:

| Candidate (attempt 14) | Refuted by |
|---|---|
| 1. bit6 is a **level** meaning "completions pending", clearing only when the ring is drained | `after-w1c=0x00000000` was read with **two completed, undrained** descriptors in the ring (`rxd[0] w0=b0440087`, `rxd[1] w0=b0040056`, both OWN\|FIRST\|LAST), and `ack[1]` read `0x00000000` again before the drain ran. A level meaning "completions pending" cannot read zero with completions pending. |
| 2. the **`DMA_POLL` doorbell** (`0x38` bit2) is the real acknowledge | `ack[2]`–`ack[4]` read `pre=0x00000062 postdrain=0x00000062 postbell=0x00000062`, `done=64`. A 64-descriptor drain followed by the doorbell moved **nothing**. The doorbell is not an acknowledge. |
| 3. the status write is **not the acknowledge** for this source at all | `isr[1]`: it is. |

**Record that this probe answered by returning a negative on both candidates it was built to
test.** It was built to decide between the drain and the doorbell; it eliminated both, and it
is that double negative — not a positive result — which moved attention to the mask, where
the actual defect was. A diagnostic that refutes its own hypothesis space is doing its job.

**`0x62` is bits 1, 5 and 6, and bits 1 and 5 are undecoded.** All 70 watchdog samples of
attempt 15, t=25.634210 to t=378.914276, read `isr=0x00000062`, with the ISR having run once
at t=20.59 and nothing writing the register afterwards. Do not guess at bits 1 and 5: they
are two more entries in the unenumerated `0x50` taxonomy (§8 question 2). What they are not
is evidence of a fault — an unacknowledged status register holds whatever has happened since
the last acknowledge, which after one interrupt in a 358-second boot is "several things".

**Consequence for a driver.** `0x50` needs no special handling: write the status back, as
§7.4 says, and the source deasserts. The real hazard is next door, in `0x54`: because the
ISR's own mask write masks RX completion (§8 question 1), a driver that copies the ROM's
mask values gets exactly one RX interrupt per boot and then relies on a timer. See §7.4.

### 3.7.1 The full interrupt protocol for `0x50`, confirmed at scale

**Attempt 16 (build #29, banner `#31`) closes the last open question about this register.**
Two facts, both measured over 535 interrupts in one boot:

**1. `write 0x50 = status` on ISR entry is sufficient. Nothing else is required.** The
bounded `isr[]` log — eight lines against a bound of eight — reads `status=0x00000040`,
`after-w1c=0x00000000` on **every** line, and the watchdog reads `isr=0x00000000` on **all
31** of its samples. Attempt 15 proved this for one interrupt; attempt 16 shows the
acknowledge holding for a whole boot with the engine continuously receiving. There is no
drain requirement, no doorbell requirement and no second register.

**2. Unmasking a source whose status is clear re-delivers.** This was the live doubt — under
edge-triggered MSI a mask lifted over an already-asserted level might produce no edge at all,
which would have made acknowledge-then-unmask mandatory. The `unmask[]` probe samples `0x50`
either side of the unmask:

```
[   18.334810] eth0: unmask[1]: isr-before=0x00000000 isr-after=0x00000000 imr=0x00001018 irqs=1
[   45.901625] eth0: unmask[8]: isr-before=0x00000000 isr-after=0x00000000 imr=0x00001018 irqs=9
```

`isr-before=0x00000000` on **every** line. **The question dissolves rather than being
answered**: because the ISR acknowledged on entry, there is never a standing level at unmask
time, so the edge-versus-level distinction cannot arise in a correctly written driver. And
the interrupts kept arriving — 535 of them, `kick=0` throughout — so unmask-alone is the
correct and sufficient sequence.

**The protocol, entire:** in the ISR, read `0x50`, write it straight back, write the service
mask to `0x54`, schedule the poll. On poll completion, write the **operating** mask to `0x54`
— not the service mask, which is the defect §8 question 1 describes. Nothing else.

**One thing this does not cover.** `pkts` equals `irqs` at all 31 samples, so this part
delivers **one interrupt per frame** with no coalescing, which follows from the `0x204` timer
never being programmed (§8 question 1's history, and attempt 9). Correct, but it means the
interrupt rate tracks the frame rate one-for-one. Fine on an idle link; it needs revisiting
before any load testing, and `0x204` is the register that would do it.

### 3.7.2 ⚑ Bit2 — link change — ASSERTS. Confirmed on four boots, 2026-10-04 to 2026-10-09

**Bit2 is the link-change source** (the ROM's branch for it is §7.4, and its handler is
§3.6.1). **For twenty-nine boots it never asserted and this document said so as a property of
the hardware. It is not one, and the claim had already been false for four boots before anyone
wrote this section.** The arming path works.

| Boot | Event |
|---|---|
| attempts 1–29 | **no bit2, ever.** The link never came up, so there was no link *change* to report |
| **attempt 30**, build #46, banner `#49` | **FIRST ASSERTION.** `isr[1]: status=0x00000004` at t=20.802394, `linkchg[1]` at t=20.815086 — the boot where the shutdown residue was cleared and the link came up (§3.6, §8 question 4) |
| **attempt 31**, build #48, banner `#51` | once, t=20.534368 |
| **attempt 32**, build #49, banner `#52` | once, **t=99.392716** — 80 s after `gbe_open()`, so the link came up well into the boot and bit2 is what reported it |
| **attempt 33**, build #50, banner `#53` | once, t=20.621237 |
| **attempt 34**, build #53, banner `#56` | once, t=21.043460 — `isr[1]` at t=21.043408, so the status read and the handler are 52 µs apart |

```
[   20.620260] eth0: isr[1]: status=0x00000004 after-w1c=0x00000000 imr=0x007bfffe txctl=0x00000000 rxctl=0x00000000
[   20.621237] eth0: linkchg[1]: status=0x00000004 reg04=0x00000b19 up=1 speed=2 fdx=1
```
*(attempt 33. Attempt 30's pair is byte-identical in form at t=20.802394 / t=20.815086.)*

| | |
|---|---|
| Bit2 | **asserted**, `status=0x00000004`, bit2 alone |
| How many times per boot | **exactly one**, on each of the five boots. One link transition, one interrupt |
| Acknowledge | `after-w1c=0x00000000`. **Write-1-to-clear clears bit2**, like every other source (§3.7) |
| Masking | bit2 is **not** in the running mask `0x1018` (bits 3, 4, 12), so it was deliverable on every earlier boot too |
| `0x04` on the same interrupt | `0x00000b19` — bit0 set, speed code 2, full duplex (§3.6) |

**Why it never asserted before attempt 30, and why that was never a defect.** The link had
never come up, and a link that never comes up produces no link *change* to report. The earlier
silence was never a masking problem and never the "second defect" the driver's comment called
it — **it was the absence of the event.** The observation was conditional on the device's
state and this document recorded it as a property of the register, which is the same reasoning
error as §3.4's `0x34` bit0/bit3 entry. **And then it was not revisited when the condition
changed**: attempt 30's own entry quotes the `linkchg[1]` line that refutes it, and attempt
32's entry counts `linkchg[1]` in its `reg04` sample tally, so the refutation sat in this repo
for four boots with the claim still standing in five places. See the corrections log in
`notes/mts-init-sequence.md`.

**What this confirms for a port.**

- **The `0x50` bit2 path is usable on this hardware.** §7.4's instruction that *"a port cannot
  rely on this arming path"* is **withdrawn** — the arming path fires, once the residue is
  cleared.
- **The decode in §3.6.1 is confirmed end-to-end on silicon**: bit2 arrived, the handler read
  `0x04`, bit0 was set, and the link was genuinely up for the rest of the boot. Attempt 32 is
  the strongest of the five, because its link came up 80 s *after* open — nothing but bit2
  could have reported it at that point.
- **Bit2 can arrive combined with other sources, and must be serviced ahead of any coalesce
  early-return.** On these boots it arrived alone, but at attempt 33 `isr[2]` 2 ms later read
  `0x000000c0`, so a combined `0x000000c4` is not hypothetical.
- **Bit2 behaves as an edge**, as far as these boots can say: one assertion per link
  transition, cleared by the status write, and never re-raised while the link stayed up — 167
  samples of `chg=1` at attempt 33 and no second `linkchg[]`; 171 at attempt 34, same result.
- **The edge is serviced before `gbe_open()` can reach its own carrier call, on every boot
  that has one.** At attempt 34 `isr[1]` is at t=21.043408 and `gbe_open()`'s
  `stage engines` — the statement after `gbe_start_engines()` — is at t=21.059472, **16 ms
  later**; `linkopen[0]` at t=21.059785 already reads `carrier=1`. Attempt 33 is the same
  shape with a 380 ms gap. **A port must therefore expect the carrier to be up before its
  open path looks**, and must not treat a no-op at open as evidence the link is down.

**One thing it does not establish.** Whether bit2 asserts on **link down** — no unplug was
performed on any of the five. Build #32's cable experiment found no bit2 on unplug or replug,
but that was a boot whose link had never come up at all, so it tested nothing about this.
**Down-edge behaviour is untested.**

---

## 4. PHY access

Reg `0x00` is the MDIO command and data port for both clause formats.

| Bits | Meaning |
|---|---|
| 31:16 | data — write data, the 16-bit register address during a Clause 45 address phase, or the result after a read |
| 15 | DONE. Write 1 to clear; poll until SET. |
| 14 | Clause 22 read |
| 13 | Clause 22 write |
| 12:8 | 5-bit address: Clause 22 register number, or Clause 45 devad |
| 7:5 | Clause 45 opcode |

**Procedure, per phase:**

1. write `0x8000` to reg `0x00` — write-1-to-clear of the sticky done flag
2. write the command word, with **bit 15 clear**
3. poll reg `0x00` until bit 15 is **set**, bounded at 10000 iterations of 1 us (~10 ms)

Step 1 runs unconditionally at the start of every phase. It is not error recovery, and bit 15
is not a go bit: setting bit 15 in the command word write-1-clears the done flag instead of
starting a transaction, so the poll then passes instantly on a transaction that never ran.

A Clause 22 access is one phase. A Clause 45 access is two — address, then data-or-read —
each with its own clear / issue / poll cycle. Read results are taken from bits 31:16.

Composed command words:

```
c22 read       0x4000 | ((reg   & 0x1f) << 8)                      ; 31:16 zero
c22 write      0x2000 | ((reg   & 0x1f) << 8) | (data  << 16)
c45 address    0x0020 | ((devad & 0x1f) << 8) | (reg16 << 16)
c45 data write 0x0060 | ((devad & 0x1f) << 8) | (data  << 16)
c45 read       0x00e0 | ((devad & 0x1f) << 8)                      ; 31:16 zero
```

**There is no PHY/port address field anywhere in the command word.** The on-die PHY is
implicit. The `0x1e` and `0x1f` values used throughout this section are Clause 45 **devads** —
MMD 30 and 31, vendor-specific 1 and 2 — not PHY addresses.

**Clause 22** reaches the standard MII set: `0x00` BMCR, `0x04` ANAR, `0x09` GTCR, `0x1f` page
select. **Clause 45** reaches the vendor registers by devad; devad 7 is touched once during
calibration.

There is no inter-transaction delay anywhere in the vendor sequence, and the only precondition
on the MDIO path is the 680 us settling delay already recorded in §7.1 step 3 — no clock
enable, no reset deassert, no PHY-address register.

#### The two-phase accessor is standard IEEE 802.3 Clause 45 — named 2026-10-02

Worth stating outright, because the section reads like decoded vendor behaviour and a reader
could take it for a Sony invention: the address-then-data pairing above **is** the IEEE
802.3 Clause 45 MMD access mechanism. Nothing in it is proprietary. The devads already carry
their standard names in this section (`0x1e`/`0x1f` = MMD 30/31, vendor-specific 1 and 2),
and the three opcodes correspond to the Clause 45 frame `OP` field:

| Command word | Bits 7:5 | Clause 45 `OP` | Phase |
|---|---|---|---|
| `0x0020` | `001` | `00` | address |
| `0x0060` | `011` | `01` | write |
| `0x00e0` | `111` | `11` | read |

Reading bit 5 as a *Clause 45 select* and bits 7:6 as the 2-bit `OP` field reproduces the
IEEE encoding exactly — `00` address, `01` write, `11` read, with `10` (read-increment)
simply unused by this ROM. That is a tidier account than three opaque opcode values, and it
predicts what a fourth value would mean. **Marked inferred:** it is a re-reading of the three
observed command words, consistent with all of them, and no instruction has been read that
distinguishes it from the bits-7:5 reading in the table above. The table stays authoritative;
this is the explanation offered for it.

**What this is NOT: the Clause 22 MMD indirection.** There is a second, entirely different
standard route to Clause 45 registers — writing MMD function and device address to Clause 22
register `MII_MMDACR = 0x0d`, then the register address or data through
`MII_MMDAADR = 0x0e`, with `MMDACR_FUNCMASK = 0xc000` and `MMDACR_DADDRMASK = 0x001f`
(FreeBSD's `sys/dev/mii/mii.h`). This device does not use it, and the command words rule it out
two ways:

1. A Clause 22 access to register `0x0d` would set **bit 13** (`c22 write`) with `0x0d` in
   bits 12:8. The observed c45 words have bit 13 **clear** and nonzero bits 7:5.
2. `MMDACR`'s function field cannot express what we observe. Its four encodings are
   `FN_ADDRESS` (`0 << 14`), `FN_DATANPI` (`1 << 14`), `FN_DATAPIRW` (`2 << 14`) and
   `FN_DATAPIW` (`3 << 14`) — one address function and **three post-increment variants of a
   single data function.** Direction comes from whether the following Clause 22 access to
   `0x0e` is a read or a write, so `MMDACR` has no way to distinguish read from write.
   The ROM's three opcodes *do* distinguish them. The mechanisms are not
   interchangeable.

The MAC's MDIO controller therefore implements Clause 45 **natively**, in hardware, rather
than tunnelling it through Clause 22. Recorded with the reason so the Clause 22 reading does
not get raised again. These `MII_MMD*` names are `#define`s — documentation for a reader, not
types to import.

**The Clause 45 addressing is confirmed from the firmware side, not assumed — 2026-10-03.** A
separate Ghidra pass decoded the argument packing of the ROM's own `smi_cl45_read`
(`0xde167c60`) from the accessor's instructions: it packs **`(reg16 << 16) | devad`**. This
driver's `CL45(reg, phy)` macro is `((reg) << 16) | (phy)`. Two independent derivations, the
same answer — the earlier one is in `notes/phyctrl-de169aa0-draft.md` §1.4, which read the same
accessor to fix `0x00a2001e` as register `0x00a2`, devad `0x1e`.

### 4.1 The Token Ring accessor — DECODED 2026-09-28

Vendor registers behind page `0x52b5` are reached by save / switch / write / restore.
**`0x52b5` is MediaTek's vendor extended page number**, the same constant the mainline
MediaTek PHY driver uses — see §1.1, where it is one of the five corroborations of the part
identification.

```
read  reg 0x1f -> saved
write reg 0x1f  = 0x52b5
write reg 0x11  = <a>
write reg 0x12  = <b>
write reg 0x10  = <c>
write reg 0x1f  = saved
```

**This is not an opaque triple of magic numbers, and reading it as one is a trap.** It is
MediaTek's **Token Ring** debug-node accessor, and the encoding is in
`drivers/net/phy/mediatek/mtk-phy-lib.c`:

```
0x11 = data & 0xffff                      /* datum, low half  */
0x12 = data >> 16                         /* datum, high half */
0x10 = BIT(15) | (read ? BIT(13) : 0)
     | (ch_addr << 11) | (node_addr << 7) | (data_addr << 1)
```

So `a` and `b` are one 32-bit **value**, and `c` is a **structured command word carrying the
address it is written to** — channel, node and data address — not a magic constant.

> **This is the one thing in §4 that will silently destroy hardware behaviour if it is read
> wrongly.** Anyone adding a seventh row to the table below by pattern-matching the first six
> — picking a `c` that "looks like the others" — is choosing a **different Token Ring node**
> and writing a 32-bit datum into it. There is no error, no timeout and no log line; the PHY
> simply has a field of its DSP set to something nobody intended. Compose `c` from the
> formula. Never from the neighbours.

Applied with these six triples, in order. The `value` column is `(b << 16) | a`; the names are
MediaTek's for the same channel/node/data addresses.

| # | `0x11` (`a`) | `0x12` (`b`) | `0x10` (`c`) | ch / node / data | value | node |
|---|---|---|---|---|---|---|
| 0 | `0xb90a` | `0x006f` | `0x8f82` | 1 / `0xf` / `0x01` | `0x006fb90a` | **`MrvlTrFix`** — 100Kp/100Kf/1000Kp/1000Kf = 6/7/6/7. **Identical to MediaTek's own write.** |
| 1 | `0xbaef` | `0x002e` | `0x968c` | 2 / `0xd` / `0x06` | `0x002ebaef` | **`SSTrKp`/`SSTrKf` ×6** = 5/6/5/6/5/6. **Identical to MediaTek's own write.** |
| 2 | `0x704d` | `0x0000` | `0x9698` | 2 / `0xd` / `0x0c` | `0x0000704d` | **unnamed** — see §1.1's open list |
| 3 | `0x344f` | `0x0002` | `0x969a` | 2 / `0xd` / `0x0d` | `0x0002344f` | `RegEEE_st2TrKf1000` |
| 4 | `0x0004` | `0x0000` | `0x9686` | 2 / `0xd` / `0x03` | `0x00000004` | **`TrFreeze`** — MediaTek *clears* this field. See §8 C. |
| 5 | `0x0671` | `0x0006` | `0x8fae` | 1 / `0xf` / `0x17` | `0x00060671` | **`SlvDSPreadyTime` = `0x0c`**, `MasDSPreadyTime` = `0x0c`. **MediaTek's own word at the same ch/node/data, differing in this one field only — the low half `0x0671` is byte-identical.** See §1.1 corroboration 2. |

**The decode was checked by arithmetic on our own table, not taken on trust.** Every `c` above
has bit15 set and bit13 clear — a write — and the six `(ch, node, data)` triples it yields are
distinct and well-formed. Entry 5's field split is checkable against its own value:
`0x00060671` gives `bits[22:15] = 0x0c` and `bits[14:7] = 0x0c`, which is the slave/master
pair the name predicts.

**Rows 0 and 1 are the behavioural fingerprint that settles the part identification.** Two
multi-field tuning words whose *values* match MediaTek's constants exactly, reached by an
access idiom that matches exactly, is evidence of a different order from a sixteen-bit ID
match. See §1.1.

**Setting `BIT(13)` in the command word makes the access a read**, with the datum coming back
from `0x11`/`0x12` instead of going out through them. Build #38 does exactly that for all six
nodes, at pre-touch and after calibration; it writes nothing new. A port wanting to inspect
the DSP has the same door. In a log the read command therefore prints with bit13 set — row 5's
`cmd=afae` against the table's write word `0x8fae` — which is the same node, not a different
one.

**All six nodes were read back on silicon, and all six already hold our values before this
driver writes anything — attempt 23, build #38, banner `#40`.** This is the read that the
read-only build existed to take:

```
tr[pretouch] 0 MrvlTrFix          cmd=af82 read=0x006fb90a ours=0x006fb90a
tr[pretouch] 1 SSTrKpKf           cmd=b68c read=0x002ebaef ours=0x002ebaef
tr[pretouch] 2 unnamed            cmd=b698 read=0x0000704d ours=0x0000704d
tr[pretouch] 3 RegEEEst2TrKf1000  cmd=b69a read=0x0002344f ours=0x0002344f
tr[pretouch] 4 TrFreeze           cmd=b686 read=0x00000004 ours=0x00000004
tr[pretouch] 5 DSPreadyTime       cmd=afae read=0x00060671 ours=0x00060671
```

`postcal` is byte-identical to `pretouch` on all six, so **our six writes are idempotent
no-ops**: we write back exactly what Orbis already left there. Two consequences follow and
both are recorded in §8 C — the values in the table above are **Sony's own settings, not
ours**, and the two §8 candidates that rested on us having set them differently are dead.

### 4.2 Per-part calibration from EFUSE

The PHY requires trim values unique to each console, read from the southbridge **function-4
BAR4** window — a different PCI function from the GBE. Gated on a validity bit:

```
if ((bar4_read(0xC06C) & 0x80800000) != 0x80800000)
        skip calibration entirely
```

Five packed words at BAR4 + `0xC05C`, `0xC060`, `0xC064`, `0xC068`, `0xC06C` are unpacked,
each field **clamped to 1..0x3f**, used to index a 64-entry byte table, and written over
Clause 45 to these registers, all at devad `0x1e`:

```
0x00e0  0x0115  0x0172  0x0173  0x0174  0x0175  0x0012
0x0016  0x0017  0x0018  0x0019  0x0020  0x0021  0x0022
```

**This whole section is independently corroborated** by the MediaTek identification in §1.1.
Mainline's `mtk-ge-soc.c` performs EFUSE-based calibration of the same kinds — REXT,
TX_OFFSET, TX_AMP/TX_R50, TX_VCM — and names four of the registers above: `0x00e0` is
`MTK_PHY_RG_ANA_CAL_RG5`, `0x0115` is `MTK_PHY_RG_BG_RASEL` (in **VEND2**, devad `0x1f`,
which is why it is the one row below that is not devad `0x1e`), and `0x0172`/`0x0173` are the
TX amplitude offsets — see the pair decode after the traps. §4.2 was derived from the ROM
with no knowledge of that driver and arrives at the same shape.

**The lookup table.** 64 bytes, needed verbatim by any port — a trim/attenuation curve,
saturated at `0x7f` for indices 0-15 then monotonically decreasing:

```
7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f
7b 7a 75 73 70 67 64 62 57 55 53 51 48 46 44 42
40 37 35 34 32 31 30 26 24 23 22 21 20 16 15 14
13 12 11 10 07 06 05 04 03 02 01 00 00 00 00 00
```

Index derivation, repeated per PHY register. Note the clamp order: the low bound is applied
to the raw value and the high bound to the already-decremented index, so a raw `0x7f`
saturates to index `0x3f`, not to `0x7e`:

```c
v   = bar4_read(0xC060);          /* or C05C / C064 / C068 / C06C */
n   = (v >> shift) & 0x7f;
if (n < 2)    n = 1;
idx = n - 1;
if (n > 0x3f) idx = 0x3f;
code = table[idx];
cl45_read (reg, &cur);
cl45_write(reg, (cur & mask) | field(code));
```

Then fixed writes and read-modify-writes, written `reg.devad`:

```
cl45 write 0x0096.1e = 0x8000
cl45 write 0x0037.1e = 0x0033
cl45  rmw  0x0039.1e : &= 0xb7ff ; |= 0x2000 ; &= 0xdfff ; delay 50 us
cl45  rmw  0x0107.1f : &= 0xefff
cl45  rmw  0x0171.1e : |= 0x0180 ; then &= 0xfe7f
cl45 write 0x0122.1e = 0xffff
cl45 write 0x0268.1f = 0x07f4
cl45 write 0x003c.07 = 0x0000
cl45  rmw  0x0033.1e : &= 0xefff
 c22  rmw  reg 0x04  : &= 0xf3ff          (ANAR)
 c22        reg 0x1f = 3 ; reg 0x1c = 0x0c92 ; reg 0x1f = 0
```

#### `0x003c.07 = 0x0000` is **EEE advertisement off**, and it matters — named 2026-09-28

The one devad-7 access in the whole sequence decodes exactly, against the MediaTek driver
identified in §1.1. Devad 7 is `MDIO_MMD_AN`; register 60 decimal is `0x3c` is
`MDIO_AN_EEE_ADV`. So `cl45 write 0x003c.07 = 0x0000` is

```c
phy_write_mmd(phydev, MDIO_MMD_AN, MDIO_AN_EEE_ADV, 0);
```

which is the same write mainline makes for this part, with the comment *"with EEE
advertised, some link partners fail to establish a stable link."*

**This closes the textbook cause of "trains but will not link", by name.** EEE
advertisement is the standard explanation for a PHY that completes 1000BASE-T training and
then never raises link — and this driver already performs the write, at the ROM's position,
so that cause is **ruled out** for our failure rather than untested. The write is not
optional and must not be dropped by a port on the grounds that EEE looks harmless.

#### The field packing — DECODED, and confirmed by read-back on silicon

Decoded at instruction level `0xde1662c0`–`0xde166728`, 2026-09-27, and first implemented in
build #34. **Eighteen fields out of four words, not fourteen byte lanes.** `w5C`…`w68` are
BAR4 `0xC05C`…`0xC068`; `f` is the extracted field after the clamp above; `T` is `table[idx]`;
`rb` is the destination's value read back immediately before the write. Devad `0x1e` unless
the row says otherwise.

| # | src | bits | width | table | dest | value written |
|---|---|---|---|---|---|---|
| 1 | w68 | [5:0] | 6 | no | `0x00e0`.1e | `f << 8`, full write |
| 2 | w68 | [8:6] | 3 | no | `0x0115`.**1f** | `f`, full write |
| 3 | w60 | [6:0] | 7 | **yes** | `0x0174`.1e | `(T<<8) \| (rb & 0x00ff) \| 0x8000` |
| 4 | w60 | [25:19] | 7 | **yes** | `0x0174`.1e | `(rb & 0xff00) \| T \| 0x0080` |
| 5 | w64 | [12:6] | 7 | **yes** | `0x0175`.1e | `(T<<8) \| (rb & 0x00ff) \| 0x8000` |
| 6 | w64 | [31:25] | 7 | **yes** | `0x0175`.1e | `(rb & 0xff00) \| T \| 0x0080` |
| 7 | w5C | [25:20] | 6 | no | `0x0172`.1e | `(rb & 0xc0ff) \| (f<<8)` |
| 8 | w60 | [12:7] | 6 | no | `0x0172`.1e | `(rb & 0xffc0) \| f` |
| 9 | w60 | [31:26] | 6 | no | `0x0173`.1e | `(rb & 0xc0ff) \| (f<<8)` |
| 10 | w64 | [18:13] | 6 | no | `0x0173`.1e | `(rb & 0xffc0) \| f` |
| 11 | w5C | [31:26] | 6 | no | `0x0012`.1e | `(f<<10) \| f`, full |
| 12 | w5C | [31:26] | 6 | no | `0x0016`.1e | `((min(f,0x3c)+3)<<10) \| f`, full |
| 13 | w60 | [18:13] | 6 | no | `0x0017`.1e | `(f<<8) \| f`, full |
| 14 | w60 | [18:13] | 6 | no | `0x0018`.1e | `((min(f,0x3c)+3)<<8) \| f`, full |
| 15 | w64 | [5:0] | 6 | no | `0x0019`.1e | `(f<<8) \| f`, full |
| 16 | w64 | [5:0] | 6 | no | `0x0020`.1e | `((min(f,0x3c)+3)<<8) \| f`, full |
| 17 | w64 | [24:19] | 6 | no | `0x0021`.1e | `(f<<8) \| f`, full |
| 18 | w64 | [24:19] | 6 | no | `0x0022`.1e | `((min(f,0x3c)+3)<<8) \| f`, full |

The 32-bit masks that appear in the Ghidra database (`0xffffc0ff`, `0xffffffc0`) are recorded
here truncated to the 16-bit register width they are actually applied at.

#### The shift asymmetry — a third independent check, added 2026-09-28

Look at the shift in rows 11–18. `0x0012`/`0x0016` shift the six-bit field by **10**; the other
three pairs — `0x0017`/`0x0018`, `0x0019`/`0x0020`, `0x0021`/`0x0022` — shift by **8**. One
pair out of four is different, and nothing in the ROM explains why. It was decoded that way
because that is what the instructions did, and it has looked like an oddity ever since.

**MediaTek has the same asymmetry, in the same place.** `mtk-ge-soc.c` masks `0x12`/`0x16` with
`GENMASK(15, 10)` and `0x17` through `0x22` with `GENMASK(13, 8)`. A six-bit field shifted by
10 occupies bits 15:10; shifted by 8, bits 13:8. **Exact agreement, on both the rule and the
exception.**

**Why this is worth more than the four name matches in §1.1.** A name matching an offset says
two documents use the same number for the same register. This says two independently derived
descriptions of *where a field sits inside a register* agree — and agree on an **irregularity**.
Getting the regular case right by accident is easy; a decoder that guessed would have used one
shift for all four pairs and produced a tidier table than the truth. The asymmetry is the part
that cannot be arrived at by tidying, and it is the part that matches.

This is a **second independent confirmation of the EFUSE decode**, and it is independent in a
different direction from build #34's read-backs: (b) below compares our computed *values*
against what Sony programmed on this console, while this compares our *field geometry* against
a third party's, on a console neither of us has. A wrong packing would have to survive both.

**Why the decode is trustworthy — three independent arguments.**

**(a) The words tile exactly.** `w60`'s fields `[6:0][12:7][18:13][25:19][31:26]` are widths
7, 6, 6, 7, 6 = **32 bits**, no gap and no overlap. `w64`'s `[5:0][12:6][18:13][24:19][31:25]`
are 6, 7, 6, 6, 7 = **32 bits**, likewise. In both words the two 7-bit fields are precisely
the two that feed the trim table. An off-by-one anywhere breaks the tiling.

**(b) The computed values reproduce the vendor's own, on this console.** Attempt 19's boot
logs every extraction together with the destination's pre-write read-back. For each of the
four read-modify-write registers, the value the decode computes is **already there**:

```
gbe efuse: 5c=XXXXXXXX 60=XXXXXXXX 64=XXXXXXXX 68=XXXXXXXX 6c=XXXXXXXX valid=YES
gbe trim  3: f=0x1e tbl[29]=0x46 reg 0x0174 rb=0xc6c6 -> 0xc6c6
gbe trim  5: f=0x1e tbl[29]=0x46 reg 0x0175 rb=0xc6c2 -> 0xc6c2
gbe trim  7: f=0x06            reg 0x0172 rb=0x0603 -> 0x0603
gbe trim  9: f=0x24            reg 0x0173 rb=0x2427 -> 0x2427
```

Rows 3, 5, 7 and 9 are the **first** touch of their register in that boot, so `rb` is what the
part held before this driver wrote anything — the values Sony's firmware programmed, surviving
the kexec because kexec does not reset the PHY. Four registers, derived from five packed words
through a clamp and a 64-entry table, land bit-for-bit on the vendor's own. A wrong packing
does not survive that.

**This is also the only validation of its kind available**, and it is worth naming why. A
wrong trim is a *silent* defect — it links and then errors under load — so no boot that
"looks fine" can confirm it, and no timeout will announce it. Comparing against values the
vendor already programmed is the one check that does not depend on observing behaviour, and
it is available only because the PHY keeps its state across the kexec. **Read the
destinations before writing them**, on any port, for exactly this reason.

**Two structural traps in this table**, both of which an earlier implementation fell into:

1. **Only rows 3–6 use the trim table.** The other twelve write the extracted field directly.
   Running every field through `table[]` mistunes twelve registers.
2. **`0x0172`, `0x0173`, `0x0174` and `0x0175` each hold two independent codes** — high byte
   `bit15` enable plus `bits[14:8]` code, low byte `bit7` enable plus `bits[6:0]` code. A bare
   full-word write destroys the neighbouring code, and for `0x0174`/`0x0175` leaves both enable
   bits clear. These four must be read-modify-write into one half each.

#### ~~Inferred~~ **DECODED: the two codes per register are the two twisted pairs** (2026-09-28)

The two-codes-per-register structure above was read out of the ROM's masks and was recorded
as an *inference* about what the halves mean. It is no longer an inference. MediaTek names
`0x0172` **`MTK_PHY_RG_CR_TX_AMP_OFFSET_A_B`** and `0x0173`
**`MTK_PHY_RG_CR_TX_AMP_OFFSET_C_D`** (§1.1). The halves are TX amplitude offsets for the
four twisted pairs of 1000BASE-T, two per register, in order A, B, C, D:

| Register | high byte `bits[14:8]` | low byte `bits[6:0]` |
|---|---|---|
| `0x0172` | pair **A** | pair **B** |
| `0x0173` | pair **C** | pair **D** |

`0x0174` and `0x0175` are not named in mainline, but they are the two registers that carry
the *trim-table* codes (rows 3–6) in the identical two-halves-plus-enable-bits layout, so the
same two-per-register reading is the only one consistent with the packing — recorded here as
still inferred for those two, unlike `0x0172`/`0x0173`.

**Consequence for a port:** clobbering a neighbouring half does not mistune "some register",
it mistunes **one specific pair of the cable** while the other three stay correct. That is
exactly the failure shape that links cleanly on a short bench cable and errors under load on
a real run, which is why trap 2 is a trap and not a style note.

**The motivation once given for this decode was wrong and the decode is not.** It was pursued
as the explanation for a link failure that was read out of a misdecoded `stat1000` — see §4.4
and the corrections log. The trim work stands entirely on (a) and (b) above, which never
depended on that story.

**A port that skips this will bring the link up and then see a high error rate, or no link at
all on marginal cable.** The trim is not optional.

### 4.3 Link mode

Autonegotiation: `BMCR |= 0x1200` (enable + restart). Forced modes write BMCR directly:

| Mode | BMCR |
|---|---|
| 10 half | `0x0000` |
| 10 full | `0x0100` |
| 100 half | `0x2000` |
| 100 full | `0x2100` |
| 1000 full, advertised | `0x1140` |

### 4.4 Link status as the PHY reports it, and the anomaly

These are standard IEEE 802.3 Clause 22 registers, not Baikal registers, and they are recorded
here because this project misread one of them for four builds and built work on the misreading.
**The authority is `include/uapi/linux/mii.h`, not memory.**

**`MII_STAT1000` — register 10, 1000BASE-T Status.**

| Bit | Mask | Kernel name | Meaning |
|---|---|---|---|
| 15 | `0x8000` | `LPA_1000MSFAIL` | master/slave resolution **failure** |
| 14 | `0x4000` | — | resolution **result**: 1 = local device is MASTER |
| 13 | `0x2000` | `LPA_1000LOCALRXOK` | **LOCAL** receiver OK |
| 12 | `0x1000` | `LPA_1000REMRXOK` | **REMOTE** receiver OK |
| 11 | `0x0800` | `LPA_1000FULL` | link partner 1000BASE-T **full** duplex capable |
| 10 | `0x0400` | `LPA_1000HALF` | link partner 1000BASE-T **half** duplex capable |
| 7:0 | `0x00ff` | — | **idle error count** |

A switch not advertising 1000BASE-T *half* duplex is normal, so **bit10 clear means nothing**.
It is the bit this project spent four builds treating as local-receiver-OK.

**`MII_BMSR` — register 1.** Bit5 (`0x0020`, `BMSR_ANEGCOMPLETE`) is autonegotiation complete.
Bit2 (`0x0004`, `BMSR_LSTATUS`) is link status and **latches low**: it must be read twice, and
a low-then-high pair means the link dropped and recovered since the previous read. Two low
readings mean it has been down continuously.

#### What this device actually reports — ~~the anomaly~~ **the anomaly, and its cause, 2026-10-04**

> ### ⚑ THE ANOMALY IS RESOLVED — attempt 30, build #46, banner `#49`
>
> It was the **shutdown residue**: bit4 of Clause 45 `0x0147.1e`, left set by the ROM's
> teardown and inherited across every kexec this project ever made. Clear it and both link
> bits set — `bmsr = 0x796d` on all 113 PHY samples, `reg04 = 0x00000b19` on all 114 MAC
> samples, held for 362 s — and the transmitter starts. §8 question 4 has the mechanism,
> §3.6 the MAC-side confirmation, `notes/hardware-bringup-log.md` attempt 30 the boot.
>
> **The block below is the record of the ten boots that reported the anomaly and it stays
> accurate for them.** Everything in it was correctly measured; what it lacked was a cause.

Observed values, with `reg04` from §3.6 alongside. Attempts 20 and 21, logs in `logs/uart/`:

```
stat1000 = 0x7800   master/slave resolved, no fault, local RX OK, remote RX OK,
                    LP gigabit full-duplex capable, idle error count 0
                    -- on every SETTLED sample; see the qualifier below
stat1000 = 0x3800   the same, except the resolution came out SLAVE
bmsr     = 0x7969   autonegotiation COMPLETE (bit5), link status (bit2) CLEAR
                    on both reads of a bit that latches low
reg04    = 0x00000b18   MAC link bit0 CLEAR, speed field 2 (1000), full duplex
```

**A fully trained gigabit PCS in both directions, with a zero idle error count on every
settled sample and autonegotiation complete, alongside two independent link-status bits — one
in the PHY, one in the MAC — both reading down.** Attempt 20 held that state across 30
`stat1000` samples and 54 `bmsr` samples over ~100 seconds; attempt 21 held it across 58 and
62; attempt 22 held it across 187 and 189 over 592 seconds.

##### The idle error count is zero on settled links, and that is **not** the same as zero — corrected 2026-09-28

The counter is **eight bits and saturating**, and **it clears on read**. Each sample therefore
reports errors accumulated since the *previous* read, and `0xff` means "at least 255", not 255.

**~~Three~~ Four samples in the whole project have been non-zero**, one per boot in attempts
19, 20, 22 and 23. **The first three sit at or immediately after a PHY initialisation or an
autonegotiation restart. Attempt 23's does not, and that framing is corrected below.** Attempt
22's is the clearest case of the init kind and the one that shows why the distinction matters:

```
t= 2.452231  pretouch   s1000=7800     <- last read before our PHY init
t=21.072540  linkopen[0] stat1000=78ff  <- first read after it: >=255 errors
t=21.141334  linkctrl[1] stat1000=7800  <- and zero on all 187 samples after
```

The `0xff` covers an **18.6-second window containing this driver's entire PHY calibration and
its autonegotiation restart**. A saturated counter across that span is expected and says
nothing about the receive path. The 187 settled samples that follow, over 571 seconds, are what
the health claim rests on.

**So the correct statement is "zero on every settled sample", and the unqualified form is
wrong.** It was used unqualified, repeatedly, as evidence that the receive path is healthy —
see the corrections log. The conclusion survives; the sentence did not, and the difference is
that the qualified version tells a reader which samples to go and look at.

**The fourth sample is a different kind, and it is a finding rather than an artefact.**
Attempt 23 reads `stat1000 = 0x08ff` at t=385.199340, **366 seconds after the last
autonegotiation restart**, on a settled link — so the "all non-zero samples are at an init"
framing above does not cover it, and the corrections log records that. What makes it a finding
rather than noise is what accompanies it: **at that same sample the two receiver-status bits
clear and never return.**

```
t=382.127823  linkctrl[119]  stat1000=3800   0x00a2.1e bit15=1
t=385.199340  linkctrl[120]  stat1000=08ff   0x00a2.1e bit15=0   <- both flip together
t=388.270868  linkctrl[121]  stat1000=0800   0x00a2.1e bit15=0
```

`0x3800` → `0x0800` is **bit13 (local receiver OK) and bit12 (remote receiver OK) both
clearing**, with bit11 unchanged, and the idle error counter saturated across the transition.
The PCS is **not retraining** — `BMSR` bit2, `reg04` bit0, `chg` and `BMCR` are all unchanged
across it, on all 440 samples of that boot. The receivers simply stop reporting OK on a link
that was never up.

**Scope it honestly: n=1 — and attempt 24 kept it there.** One transition, on one boot. It is
also the one boot that resolved as **slave** (`0x3800`, bit14 clear), which is either a lead or
a coincidence at this sample size.

> **IT DID NOT REPRODUCE. Attempt 24 (build #40, banner `#43`) is the longer run and it is
> clean.** `stat1000` held `0x3800` on **361 / 361** `linkctrl` samples from t=21 to t=1114 —
> **729 seconds past** attempt 23's transition, with more inbound traffic (696 frames against
> 36 measured in attempt 23's window) — and `0x00a2.1e` bit15 stayed **1** in all **363** gate
> samples. The raised `GBE_WATCH_SAMPLES` bound that build #40 existed to provide ran 216 of
> its 300 pairs and recorded `kick=0` throughout.
>
> **What this changes and what it does not.** The attempt-23 measurement stands exactly as
> written above — two registers, one flip, 440 samples, no retraining. What is withdrawn is any
> reading of it as a *property* of this hardware: a degradation that builds up, a thermal or
> counter-saturation mechanism, or an expectation that a long boot will lose the receivers. None
> of that has a second observation behind it, and the one longer boot says the opposite. Do not
> cite the drop as something the link does; cite it as something the link did once.
>
> Attempt 24's single `0x38ff` is the *other* kind: `linkopen[0]` at **t=21.126**, with
> `pretouch` at t=2.447 already reading `0x3800` — the counter saturated inside this driver's own
> calibration-and-restart window, was cleared by that read, and never returned across 361
> settled samples. So "zero idle errors on **every settled sample**" now rests on 361 more
> samples, and the init-window qualifier is again load-bearing.

**Reception after attempt 23's drop was unobserved rather than stopped** — the watchdog dump was
bounded at 72 samples and expired at t=388.59, near the transition by coincidence, so nothing
measured RX afterwards. That hole is **why build #40 raised the bound**, and attempt 24 filled it
from the other side: by not reproducing the drop at all, it leaves "does RX survive the drop?"
still unanswered, but no longer the project's largest open question — there is no recurring
event to survive.

**And the driver is receiving packets the whole time — in the window where that is measured.**
Attempt 20: 528 frames on 497 hardware interrupts, `kick=0` on every watchdog sample. Attempt
21: 50 frames on 49 interrupts and still climbing when the capture was taken. Earlier boots
reached 542. The physical link works and data flows inbound while both link-status bits say it
does not.

This is the open anomaly, and it is not a receiver or analog problem. See §8 for what it
invalidates downstream.

#### The master/slave resolution is not stable across boots

| Boot | pre-touch `stat1000` | steady state | bit14 |
|---|---|---|---|
| Attempt 20, build #35 | `0x3800` | `0x7800` (30/30) | 0 → **1** |
| Attempt 21, build #36 | `0x7800` | `0x3800` (58/58) | 1 → **0** |
| Attempt 22, build #37 | `0x7800` | `0x7800` (187/187) | 1, no change |
| Attempt 23, build #38 | `0x3800` | `0x3800`, then `0x0800` from t=385 | 0, no change |
| Attempt 24, build #40 | `0x3800` | `0x3800` (361/361, to t=1114) | 0, no change |
| Attempt 29, build #45 | `0x3800` | `0x7800` (38/39, to t=133) | 0 → **1** |
| Attempt 30, build #46 | `0x3800` | `0x7800` (113/113, to t=362) | 0 → **1** |
| Attempt 31, build #48 | not in capture (truncated tail) | `0x7800` (106/106, to t=346) | — |
| Attempt 32, build #49 | `0x7800` | **`0x3800` (57/57, to t=268)** | 1 → **0** |

**This table skipped attempts 25, 27 and 28** — their `stat1000` figures are in their own
entries in `notes/hardware-bringup-log.md` (27: `0x3800` on 109/110; 28: `0x3800` on 65/66)
and were never transcribed here. Recorded rather than back-filled, because the pre-touch
column is what the table turns on and re-deriving it for three past boots is a separate job
from recording this one.

Same cable, same switch, same silicon, and **seven boots in this table have taken four
different paths** through these values, with this driver's initialisation the only thing in
between. Attempt 24 is the first repeat: it takes attempt 23's entry (`0x3800` pre-touch,
`0x3800` settled) and then **does not** diverge from it, which is the first evidence that any
of these paths is reproducible at all. **Attempt 29 is the second repeat, and it repeats the
other path** — `0x3800` pre-touch → `0x7800` settled, attempt 20's path, five boots and nine
days later. **Attempt 30 takes that same path a third time**, and does so in the boot where
the link came up and 71 frames went out, so `0x7800`/master is the resolution a working link
has been observed under. That is one observation and **not** a finding about which role is
required: attempt 24 held `0x3800`/slave for 1114 s with reception working, and nothing here
says a slave-resolved boot could not also transmit.

> ### ⚑ AND A SLAVE-RESOLVED BOOT DOES TRANSMIT, AT LINE RATE — attempt 32, 2026-10-09
>
> **The qualification above was written as caution and is now a measurement.** Attempt 32
> settles at **`0x3800` on 57 of 57 samples** — the slave resolution, attempt 21's path, and
> **not** the `0x7800` every previously-working boot took — and in that same boot the console
> moved **1,170,378 packets / 1,519,887,431 bytes at a peak of 847.6 Mbit/s**, with an external
> browser speed test reading 711 Mbit/s down / 768 up. It also takes a **fifth** path through
> the table: `0x7800` pre-touch → `0x3800` settled, which only attempt 21 had taken, and that
> boot transmitted nothing.
>
> **So the master/slave resolution does not gate the datapath and does not gate throughput.**
> That removes it as a candidate explanation for anything on the TX path — the first time any
> boot has been able to say so, because until attempt 30 no boot transmitted at all and until
> attempt 32 every boot that did had resolved master. **The instability itself is still
> unexplained**; what is gone is the suspicion that it mattered.

**Unexplained.** A port must not assume either role, and must not assume the resolution is
stable *within* a boot either.

Two qualifications the first two rows did not need:

- The old blanket "both receivers read OK and the idle error count is zero on the settled
  samples in all four states, so it is not a signal-quality difference" **no longer holds as
  written.** Attempt 23 loses both receiver-OK bits at t=385 (above). For attempts 20, 21, 22,
  24, 29 and 30 the statement stands — attempt 29 settles at `0x7800`, both receiver-OK bits
  set and the idle-error byte zero on all 38 settled samples to t=133, and attempt 30 holds
  the same on all 113 settled samples to t=362.
- ~~Attempt 23 is the only boot that resolved as **slave and stayed there**, and the only one
  whose receivers dropped.~~ **The coincidence is broken. Attempt 24 resolved as slave and
  stayed there too (`0x3800`, bit14 clear, 361/361) and its receivers did not drop**, so
  slave resolution is not what accompanies the drop. Five boots, and the two slave boots now
  disagree about the drop — which is what asking a second boot is for.

#### The pre-touch read, and what it settles

Build #35 added a read of the PHY at the very top of probe, before any write — the first time
this project has looked before touching. It reads `lnk=0` on both boots that have it, with
`reg50 = 0x00000000`.

**The link is already down when we arrive.** Orbis's link does not survive the kexec, so every
hypothesis of the form "our initialisation destroys a working link" is answered negative, and
the probe-teardown bisect built to find which write was responsible had nothing to find. Keep
the pre-touch read: it costs one MDIO transaction at probe and it is the only thing that
distinguishes *broke it* from *found it broken*.

**Attempt 23 extended the pre-touch read from link state to the PHY's whole vendor
configuration, and that is what makes the point decisive.** Build #38 read all six Token Ring
nodes and the Clause 45 registers at the same pre-touch point, and found **the entire vendor
configuration already correct** — every value identical to the one we would write (§4.1). So
the part is not merely *found* with its link down; it is found **correctly configured** with
its link down. The link went down at the kexec boundary with the PHY's register space intact.

> ⚠ **This paragraph used to end "which puts the cause outside that register space". That
> conclusion was wrong, and it was wrong for an instructive reason — corrected 2026-10-04,
> attempt 30.** The cause was **inside** the PHY's register space all along: bit4 of
> `0x0147.1e`. It was invisible because "correctly configured" means *matching Orbis's running
> values*, and `0x0147.1e` is not one of them — nothing in either init path writes it, so there
> was no value to compare against and the register was not in the comparison set. **An
> exhaustive comparison is only exhaustive over the registers somebody programs.** The premise
> of the sentence — the register space survives the kexec intact — is correct and is in fact
> *why* the residue persisted. See §8 question 4 and the corrections log.

---

## 5. Descriptors

### 5.1 Layout — 16 bytes

```c
struct baikal_desc {
        u32 word0;      /* ownership, flags, length    */
        u32 word1;      /* buffer bus address (32-bit) */
        u32 word2;      /* RX: VLAN tag; TX: see below */
        u32 word3;
};
```

256 descriptors per ring = `0x1000` bytes per ring.

### 5.2 Ownership — the central convention

**bit31 set = the driver owns the descriptor. bit31 clear = the hardware owns it.**

Inverted from most MACs, and uniform across both rings:

- TX is initialised to `0x80000000` (driver owns every slot, all idle). The driver fills a
  descriptor and **clears bit31** to hand it over; the device **sets bit31 back** when the
  frame has been transmitted.
- RX is initialised with bit31 **cleared** (hardware owns every slot). The device **sets
  bit31** when it has deposited a frame.

Because the device sets bit31 *and* a non-zero length together on RX, the test for a
completed RX descriptor is `word0 >= 0x80000001`, not a plain bit test.

### 5.3 RX `word0` bits

Confirmed against live completions 2026-09-27 (attempt 13), four consecutive
descriptors after the engine received real traffic:

| | `word0` | b31 OWN | b29 FIRST | b28 LAST | b22 | b19:18 class | length |
|---|---|---|---|---|---|---|---|
| `rxd[0]` | `b0040052` | 1 | 1 | 1 | 0 | 1 | `0x052` = 82 |
| `rxd[1]` | `b0440057` | 1 | 1 | 1 | 1 | 1 | `0x057` = 87 |
| `rxd[2]` | `b0440087` | 1 | 1 | 1 | 1 | 1 | `0x087` = 135 |
| `rxd[3]` | `b0440137` | 1 | 1 | 1 | 1 | 1 | `0x137` = 311 |

This is the first direct observation of the device writing RX status back, and it
confirms the ownership convention (§5.2), the 11-bit length field, and FIRST/LAST
on single-buffer frames. `word2` and `word3` were zero in all four.

**`bit22` is not decoded.** It was clear on the first completion of the boot and set
on every one after it. A first-frame-after-start marker and a receive-status flag
both fit four samples; do not guess it into the driver.

```
[10:0]   packet length in bytes
[16]     checksum status
[17]     checksum status
[19:18]  protocol class
[20]     VLAN tag present
[26]     protocol class, high bit
[30]     WRAP — set on the last descriptor in the ring
[31]     OWN
```

`word2[11:0]` carries the 802.1Q VLAN id when bit20 is set.

**`bit30` WRAP is CONFIRMED on silicon — attempt 30, build #46, 2026-10-04**, and this is the
first boot in which any ring reached index 255. `gbe_ring_init()` sets WRAP on RX descriptor
255 and `gbe_rx()` restores it on re-arm; the watchdog caught the crossing:

```
[  271.903949] watch[49]: ... rxctl=0x00000008 cur=0x7fffef60 pkts=758 irqs=775 kick=0
[  277.023799] watch[50]: ... rxctl=0x00000008 cur=0x7fffe020 pkts=770 irqs=790 kick=0
```

Ring base `0x7fffe000`, 256 × 16 bytes, so index 255 is at offset `0xff0`. `0x7fffef60` is
**index 246**; the next sample reads offset `0x020`, **index 2**. Between them the engine
consumed indices 246–255 and 0–1 — **twelve descriptors** — against `pkts` advancing
**758 → 770, twelve frames**, with `kick=0` either side and `imr` intact. **The device read
the WRAP bit, returned to the ring base instead of walking off the end, and lost nothing doing
it.** The previous record was attempt 15's `0x7fffeb80`, 184 descriptors past the base, so this
bit and the ring's own topology were decoded but unexercised until now.

~~**TX WRAP is still unexercised**~~ **SUPERSEDED — see §5.4.** It was exercised at
attempt 32 and the engine hung on it, twelve times, exactly 255 packets apart.
§5.4 carries the evidence and the 255-of-256 workaround; §7.6 carries the warning
at the point a porter would hit it.

**Checksum offload decode.** Form `class = ((word0 >> 18) & 3) + ((word0 >> 26) & 1) * 4`:

| class | meaning |
|---|---|
| 0 | IP only — IP checksum checked; valid unless bit17 |
| 1, 2 | IP + TCP or UDP — IP checked (bit17), L4 checked (bit16) |
| 5, 6 | L4 verified, IP not applicable (bit16) |

A set status bit means *bad*; report "verified OK" on the bit being clear.

### 5.4 TX `word0`

Decoded at instruction level, 2026-09-24. **The TX layout is not the RX layout with different
names** — the length field alone is five bits wider.

```
[15:0]   segment length in bytes — 16 bits, NOT the 11 that RX uses.
         Each segment is capped at 0xff80.
[18]     LSO/TSO enable
[23]     UDP checksum offload
[24]     TCP checksum offload
[25]     IP checksum offload
[26]     VLAN insert — word2 supplies the tag
[28]     LAST  — last segment of the frame
[29]     FIRST — first segment of the frame
[30]     WRAP
[31]     OWN
```

**FIRST and LAST are not optional.** A single-descriptor frame is both, so both bits are set;
a descriptor claiming to be neither the start nor the end of a frame is not transmitted, and
the device reports neither a packet nor an error for it.

**WRAP is set at transmit time**, on whichever descriptor happens to land at index 255 — not
once at ring init. TX descriptors are initialised without it (§7.2 step 3).

`word2`:

- `0` when the frame carries no VLAN.
- `0x81000000 | (vid & 0xfff)` when bit26 is set.

`word3`:

- `0` unless TSO.
- `(total_len & 0x1fffff) | (mss << 21)` when bit18 is set. **The device supports TCP
  segmentation offload.**

**The idle marker.** Outside those two cases `word2` is set to `0xffff0000`, which marks the
descriptor **idle**. Queuing a frame clears it; reclaiming one restores it. The completion scan
therefore stops when it reaches a descriptor whose `word2` high half **is** `0xffff` — that slot
has already been reclaimed and there is nothing behind it. (Corrected 2026-09-23; this was
stated backwards and the driver inherited the error.)

> ⚠ **"Reclaiming one restores it" is the *driver's* obligation, not an observed silicon
> behaviour, and the distinction matters now that reclaims have been seen.** Attempt 30's
> `txwatch` reads `w2=ffff0000` on every reclaimed descriptor — but `gbe_tx_reclaim()` writes
> `d->word2 = TXD_W2_IDLE` itself, so that reading confirms the reclaim path ran and nothing
> more. **The device has never been observed writing the marker back**, and no boot has
> separated "the device restores it" from "we do". Recorded 2026-10-04 because this entry was
> briefly reported as a silicon confirmation; the corrections log has the row.

**CONFIRMED on silicon — attempt 27 (build #43, banner `#46`), 2026-10-03.** `word2` had never
been sampled on a queued TX descriptor on any boot, which attempt 26 logged as an open
instrument gap. Build #43's `txwatch` dumps it, and on the head descriptor of a queued frame it
reads **`0x00000000`** — on every sample in the boot:

```
[  348.342805] txwatch[64]: txctl=0x00000005 txcur=0x7ffff000 prod=56 cons=0 free=200 txd w0=30000149 w1=7fe00000 w2=00000000 tx=0/0 drop=0 err=0 q=0 carrier=1 pool=0x03000/0x00000 stall=0
```

So **the marker is cleared on a queued descriptor**, exactly as the paragraph above says, and
"maybe the device still sees the slot as idle" is answered **negative**. Together with `w0` and
`w1` from the same line, the descriptor the device is handed is now verified field by field in
**all four words**: `w0 = 0x30000149` (OWN clear so hardware owns it, FIRST and LAST both set,
`[15:0] = 0x0149` = 329 bytes, every offload bit and WRAP clear), `w1 = 0x7fe00000` (a valid
buffer address inside the 31-bit-clamped aperture), `w2 = 0` (no VLAN, marker cleared). **TX
descriptor *content* is closed as a hypothesis.** Note what this does and does not do: it
confirms the *driver* writes the layout this section documents. The field boundaries themselves
are still decoded-from-ROM rather than confirmed by a successful transmit (§8 question 3) —
confirming those still needs a boot that transmits.

#### ⚑ THAT BOOT HAPPENED — attempt 30, build #46, banner `#49`, 2026-10-04

**The device accepted, consumed and returned 71 descriptors written to this layout.** That is
the confirmation §8 question 3 was waiting for, and it is the first evidence about these field
boundaries from the device side rather than from the ROM.

```
xmit[1]:     rawlen=329 len=329 idx=0 off=0x00000 w0=30000149 w1=7fe00000 free=255 txctl=0x00000000->0x00000008 txcur=0x7ffff000
txwatch[66]: txcur=0x7ffff470 prod=71 cons=71 free=256 txd w0=80000000 w1=00000000 w2=ffff0000 w3=00000000 tx=71/6014 drop=0 err=0 stall=0
```

- **`w0 = 0x30000149` was accepted** — bit28 LAST, bit29 FIRST, `[15:0] = 0x0149` = 329 bytes,
  OWN clear, every offload bit and WRAP clear. Attempts 17 through 29 handed the device the
  identical word and it returned nothing, so acceptance is the discriminating observation.
- **`w0 = 0x80000000` on the reclaimed descriptor is the device's write.** `gbe_start_xmit()`
  writes `word0` with OWN **clear**; `gbe_tx_reclaim()` only advances on `w0 & OWN`. So bit31
  set means **the device handed ownership back**, 71 times, having also cleared the length and
  the FIRST/LAST bits in the process. **§5.2's ownership convention is confirmed on the TX
  side**, where it had only ever been confirmed on RX.
- **The stride closes arithmetically.** `txcur` advances from the ring base to `0x7ffff470`;
  `0x470 / 16 = 71`, and `tx` is 71. One 16-byte descriptor consumed per frame, counted two
  independent ways.

> **Scope it exactly.** The layout is confirmed as **accepted and consumed**. It is **not**
> confirmed as *correct on the wire*: nothing in this boot reads a transmitted frame back, so
> the 16-bit length field is confirmed to be where the device expects a length, not confirmed
> to carry the length the peer received. The link partner reports traffic arriving
> (`notes/hardware-bringup-log.md`, attempt 30) but reports no frame contents. Also still
> unexercised: every offload bit (bits 18, 23, 24, 25, 26), `word3`/TSO, `word2`'s VLAN
> encoding, and ~~**WRAP at TX index 255** — 71 descriptors is 71, and the ring is 256~~
> — **TX WRAP is exercised as of attempt 31 and the engine hangs on it; see the two blocks
> below.**

---

#### ⚑ TX WRAP IS EXERCISED, AND THE ENGINE STOPS ON IT — attempt 31, build #48, banner `#51`, 2026-10-09

**The scope note above is answered and the answer is a defect in the device.** TX WRAP at
index 255 is no longer unexercised: attempt 31 reached it **twelve times in one boot**, and
the transmitter parked on it every time. The full per-stall table is in
`notes/hardware-bringup-log.md`, attempt 31; the open question is §8 question 8.

```
txstuck[1]:  tx=255 frozen with 69 descriptors outstanding, txctl=0x00000008 txcur=0x7ffffff0 -- running the ROM's TX restart
txwatch[12]: txctl=0x00000008 txcur=0x7ffffff0 isr=0x00000000 imr=0x00001018 prod=49 cons=255 free=206 txd w0=7000004d w1=7fe0d780 w2=00000000 w3=00000000 tx=255/40875 drop=0 err=0 q=0 carrier=1
```

- **`tx` = 255, 510, 765, 1020, 1275, 1530, 1785, 2040, 2295, 2550, 2805, 3060** — eleven
  consecutive differences of **exactly 255**. The ring is 256 and one descriptor per frame, so
  the engine fails on the **first** descriptor of every lap that carries WRAP and on no other.
- **`txcur = 0x7ffffff0` on all twelve**, and `txctl = 0x00000008` on all twelve. The base is
  `0x7ffff000`, so that is `base + 0xff0` = `base + 255 × 16` — **index 255**, exactly where
  this section says WRAP is written.
- **The descriptor is the word this section specifies.** Twelve distinct `(word0, word1)`
  pairs, each held across three watchdog samples, one per stall:

  | `word0` | `[15:0]` | decode |
  |---|---|---|
  | `0x70000042` | 66 | bit31 OWN **clear** — the device owns it; bit30 **WRAP set**; bit29 FIRST; bit28 LAST; every offload bit clear |
  | `0x7000004d` | 77 | as above |
  | `0x7000005e` | 94 | as above |
  | `0x70000135` | 309 | as above |
  | `0x7000050c` | 1292 | as above |
  | `0x700005ea` | 1514 | as above |

  `word2 = 0x00000000` (idle marker cleared, no VLAN) and `word3 = 0x00000000` (no TSO) on all
  thirty-six samples; `word1` a valid pool address inside the 31-bit-clamped aperture.

- **`notes/tx-wrap-draft.md` derived `0x70000042` from the ROM independently**, as what
  `mts_tx` emits for a 66-byte single-segment frame at `tx_prod == 0xff` — bit-for-bit the
  word the device parked on. **This section is not wrong and the driver is not wrong.** The
  WRAP bit, its position and its timing are all as decoded; there is no descriptor-content
  hypothesis left and **the index-255 descriptor must not be "fixed"**.
- **It is not length-dependent.** 66 bytes and 1,514 bytes hang identically. `0x5ea` = 1514 is
  also the first near-MTU frame TX has ever carried on this device — attempt 30's 71 frames
  averaged ~85 bytes.
- **The device reports nothing.** `isr = 0x00000000` with `imr = 0x00001018` on **all 36**
  stalled samples, so `0x50` bits 20|22 are unmasked and clear; `drop=0 err=0 carrier=1`
  throughout. §7.4 and §8 question 8 carry what that does to the restart's meaning.
- **§7.4's restart recovers it.** Twelve restarts ran and transmission resumed after every
  one, `txcur` observed back at the ring base on the four the log prints; the boot moved
  **3,115 packets / 799,018 bytes**. The recovery is real and the cause is open.

> **What is still unexercised on TX**, unchanged by this boot: every offload bit (18, 23, 24,
> 25, 26), `word3`/TSO, and `word2`'s VLAN encoding. And on-wire frame *contents* are still
> unread in either direction on every boot.

#### ⚑ AND THE CAUSE IS THE POSITION, NOT THE BIT — attempt 32, build #49, banner `#52`, 2026-10-09

**Walk 255 of the 256 descriptors, put WRAP on index 254, never queue index 255, and the hang
disappears.** That is the whole fix, and it resolves the two candidates the block above left
open onto the first of them.

```
[    2.452328] txring: using 255 of 256 TX descriptors, WRAP on index 254
[  197.170415] txwatch[19]: ... tx=659265/789878661 drop=0 err=0 q=0 carrier=1 stall=0
[  263.729505] txwatch[32]: ... tx=1170378/1519887431 drop=0 err=0 q=0 carrier=1 stall=0
```

| | prediction, written before the boot | observed |
|---|---|---|
| **the position** — the last descriptor of the 4 KB ring page, or `txcur` reaching `base+0xff0` | ring wraps cleanly, stall gone | **this** |
| **the WRAP bit** — bit30 mishandled wherever it lands | parks at index 254, `txcur=0x7fffffe0`, Δ`tx`=255 cadence intact | **not observed** |

- **Zero `txstuck[]` and zero `txrst[]` in 268 seconds**, against twelve stalls in 321 s at
  attempt 31. **`0x7ffffff0` appears zero times in the entire boot**; the highest `txcur`
  sampled is `0x7fffffd0` (index 253).
- **`tx = 1,170,378 / 1,519,887,431`** — **4,589 complete laps** of the ring, every one
  crossing WRAP at index 254, where the engine could not previously survive one.
- **Peak 847.6 Mbit/s** from the driver's own counter (542,444,670 bytes / 5.120071 s), mean
  frame 1,448 bytes, and an **external** browser speed test of **711 Mbit/s down / 768 up**
  against attempt 31's 0.37 / 0.26. Provenance of both is distinguished in
  `notes/hardware-bringup-log.md`, attempt 32.

**So §5.4's WRAP decode is correct and is now confirmed three independent ways:** attempt 30
showed the device *reads* bit30 (on the RX ring), attempt 31 showed the word this driver
writes at index 255 is byte-identical to what `mts_tx` emits there, and attempt 32 shows the
engine **traverses** a WRAP it meets at index 254. **Nothing about the bit, its position or
its timing needs changing, and the index-255 descriptor still must not be "fixed".**

> **What a port must do, stated as a requirement rather than as a finding.** **Allocate 256
> descriptors in one 4 KB page and walk only 255 of them**, with WRAP on index 254. The
> physical ring, the descriptor layout, the WRAP semantics and §7.4's restart are all
> unchanged; only the logical length moves. The cost is one descriptor. On this driver it is
> the `txring` module parameter, defaulting to 255, and **`txring=256` reproduces the hang on
> demand from the same image** — which is what makes the result falsifiable rather than a
> before-and-after across two builds.
>
> ⚠ **Not established: that index 255 is safe under any other arrangement.** No boot has
> tested a differently-aligned ring, a larger allocation, or a ring that is not one 4 KB page.
> The working configuration is exactly the one above.
>
> **Why it hangs is a hardware question and the answer is an inference.** The shape fits a
> **descriptor prefetch that crosses the ring's page boundary** — the engine reads ahead past
> the descriptor it is working on, and at index 255 that read leaves the page — which would
> account for the device raising no error at all, because from its point of view nothing
> failed. **That mechanism is a hypothesis this project cannot test.** What is established is
> a positional hardware erratum and a one-descriptor workaround. §8 question 8 carries it.

---

## 6. Rings and buffers

| | TX | RX |
|---|---|---|
| Descriptors | 256 allocated, **walk 255** (§7.6) | 256 |
| Descriptor ring | `0x1000` coherent | `0x1000` coherent |
| Buffer pool | `0xa0000` (640 KB) | `0x60000` (384 KB) |
| Per-buffer stride | variable, 128-byte granular | `0x600` (1536) |

RX geometry closes exactly: 256 × `0x600` = `0x60000`.

**Both directions are copy-based.** Neither maps network buffers for DMA; both copy between
pre-allocated coherent pools and the skb. No scatter-gather, no per-packet mapping failure
path, no bounce-buffer interaction with the IOMMU.

**TX pool** is a byte-granular ring independent of the descriptor ring: allocation size is
the frame length rounded up to 128 bytes; a head offset advances per frame and wraps to 0
when `head + need >= 0xa0000`; a tail offset is recomputed on completion from the bus address
the device reports; each descriptor segment is capped at `0xff80` bytes.

**RX pool** is fixed: descriptor *i* permanently points at pool offset `i * 0x600`. Buffers are
never recycled or re-pointed — the frame is copied out of the pool and the same buffer is handed
straight back.

**There is an RX refill path, and it is required.** (Corrected 2026-09-24; this section
previously said "there is no RX refill path to write", which was wrong.) The buffer address
never changes, but the ROM does not rely on that: as it consumes each descriptor it rewrites the
descriptor completely before releasing it, exactly as at ring init —

```
word0 = 0x80000600                    (OWN | length 1536)
word1 = rx_pool_busaddr + idx*0x600
if idx == 255: word0 |= 0x40000000    (WRAP)
word0 &= 0x7fffffff                   (clear OWN — hand to hardware)
```

Without the re-arm the ring delivers one lap and then stalls: the device consumes the length
field, and a descriptor handed back without it is not usable again. Re-arm on every pass, in the
RX drain (§7.5), not only at start.

---

## 7. Operating sequences

### 7.1 Cold init (once, at attach)

```
 1.  write 0x200 = 0
 2.  read 0x50, write it straight back
 3.  delay 680 us
 4.  c22 read PHY regs 2 and 3            (PHY identifier)
 5.  write 0xac = 9
 6.  EFUSE-driven PHY calibration          (§4.2)
 7.  paged Clause 22 programming           (§4.1)
 8.  write 0x7c = 25000000                 (reference clock)
 9.  rmw   0x04 &= 0x7fffcfff
10.  rmw   0x78 &= ~1
11.  rmw   0x0c &= ~0x80
12.  write 0x14 = swap16(mac[0..1])
13.  write 0x18 = swap32(mac[2..5])
14.  optionally write 0x140/0x144          (secondary MAC)
15.  write 0x74 = 0x2277
16.  rmw   0x08 |= 0x07597c00
17.  write 0x1d4 = 1
18.  rmw   0x10 = (v & 0xffffff6e) | 0x81
19.  write 0x30 = 0x10100
20.  c22 rmw BMCR |= 0x1200                (autoneg enable + restart)
```

Steps 8–19 must follow calibration; the reference clock in particular precedes MAC enable.

> ### ⚑ A PORT THAT ARRIVES BY KEXEC MUST ADD ONE STEP THE ROM DOES NOT HAVE
>
> **Before step 6, clear bit4 of Clause 45 register `0x0147` devad `0x1e`.** One
> read-modify-write, once.
>
> ```
> cl45 rmw 0x0147.1e : clear BIT(4)
> ```
>
> This is **not** part of the ROM's sequence and is deliberately not numbered into it above —
> the list is the firmware's and stays the firmware's. The ROM has no such step because it never
> needs one: its own shutdown routine sets that bit as its last PHY act (§8 question 4) and the
> power cycle that follows clears it, so the firmware never meets its own residue. **A kexec has
> no power cycle.** The PHY's register space survives intact, the bit survives with it, and on
> this part **bit4 inhibits the transmit path and holds both link-status bits down** — measured
> at attempt 30 against attempt 29 as a control (§1.1, §8 question 4).
>
> Without it: the PCS trains, autonegotiation completes, RX works, `reg04` bit0 and `BMSR` bit2
> both read down for as long as you care to watch, and **nothing is ever transmitted**. That was
> twenty-nine boots of this project. With it: gigabit full duplex, both link bits set, and
> frames out.
>
> **Position: first, before anything else in PHY bring-up.** It is the removal of a teardown
> bit, not a configuration step, and every write below it configures a PHY that would otherwise
> still be carrying it. **What bit4 does mechanically is unknown** — clear it anyway; do not
> write a comment explaining it.

**Step 9 is not an in-place read-modify-write, and the difference is observable.** The line
above reads as "read `0x04`, mask, write it back, all at step 9". The ROM does not do that. It
**reads `0x04` at `0xde1668a9`, before PHY calibration**, holds the value across ~900 bytes of
Clause 22 work, and writes `saved & 0x7fffcfff` back at `0xde166c2d`. The read is hoisted above
the calibration; only the write is at step 9.

That matters exactly when calibration changes a bit inside `0x7fffcfff`. The ROM would
**restore the pre-calibration value** for every such bit; a driver doing a fresh read-modify-write
at step 9 **keeps the post-calibration value**. Our driver does the fresh read, so the two
diverge on any bit calibration touches. Reproducing the ROM exactly means capturing `0x04`
before step 6 and using that captured word at step 9.

**MEASURED 2026-10-02, attempt 25 — the divergence is real and unobservable on this part.**
This paragraph previously said *"nothing in this repository has read `0x04` either side of
calibration"*. That was already stale: `gbe_stage()` prints `0x04` at every stage boundary,
and attempt 25's log carries **31 samples in one boot** — from `stage reset`, before the PHY
identifier read, through `stage pg5`, after the whole paged sequence and both Token Ring
passes, to `stage engines` at t=20.64 — reading **`r04=0x00000b18` on every one of them**.
Both `bar0` dumps agree. **Calibration changes no bit of `0x04`**, so the hoisted read and
the fresh read produce the same word and the divergence cannot be the cause of anything here.
It stays documented, because it is a genuine difference from the ROM and a port to a part
whose calibration *does* touch `0x04` would hit it — but it is not a lead, and it must not be
cited as an untested risk again.

> ### ⚠ CORRECTED 2026-10-04, attempt 30 — "calibration changes no bit of `0x04`" is now FALSE on this part
>
> It was true of every boot up to and including attempt 29, and it is false of the first boot
> that works. Build #46 clears the PHY shutdown residue at the **top** of `gbe_phy_calibrate()`,
> and `0x04` bit0 sets 39 ms later, inside the calibration window:
>
> ```
> [    2.869840] stage cfg5      r04=0x00000b18
> [    2.876262] residue: 0x0147.1e 0010 -> 0000 bit4 1 -> 0 cleared=1 ret=0
> [    2.915227] stage efuse     r04=0x00000b19
> ...
> [    3.342022] stage anrestart r04=0x00000b19
> [    3.348248] stage machi     r04=0x00000b18
> ```
>
> Twenty-one of the boot's thirty-one `stage` samples read `0x00000b19`. **So the hoisted read
> and the fresh read now produce different words**, and they differ on **bit0**, which is inside
> the `0x7fffcfff` mask and therefore written back. The ROM would restore the pre-calibration
> `0` there; this driver writes the post-calibration `1`.
>
> **Scoped honestly: this is almost certainly harmless and it is not measured.** Bit0 is the
> status half of `0x04` (§3.6) and the ROM never writes link state into this register — §3.6
> says in terms that inventing such a write would not be a port. Writing an observed `1` back
> into a status bit is most likely a no-op, and the boot in which it happens is the boot that
> works. What is corrected is the **claim**, which said the divergence is unobservable on this
> part: it is observable now, and a port that chooses to reproduce the ROM's hoisted read must
> capture `0x04` **before** the residue clear, not merely before the EFUSE trim.

### 7.2 Start (interface up)

Guarded: return immediately if the interface is already running.

```
 1.  tx_prod = tx_cons = 0 ; tx_free = 256      (ROM value; walk 255 — §7.6)
 2.  zero the TX descriptor ring
 3.  for each TX descriptor: word0 = 0x80000000 ; word2 |= 0xffff0000
 4.  sync the TX ring for device
 5.  rx_idx = 0
 6.  zero the RX descriptor ring
 7.  for i in 0..255:
         word0 = 0x80000600                (OWN | length 1536)
         word1 = rx_pool_busaddr + i*0x600
         if i == 255: word0 |= 0x40000000  (WRAP)
         word0 &= 0x7fffffff               (clear OWN — hand to hardware)
 8.  sync the RX ring for device
 9.  write 0x44 = tx_ring_busaddr          (base first)
10.  write 0x3c = tx_ring_busaddr          (then current)
11.  write 0x48 = rx_ring_busaddr
12.  write 0x40 = rx_ring_busaddr
13.  rmw   0x34 |= 1                       (TX engine run)
14.  rmw   0x38 |= 1                       (RX engine run)
15.  write 0x54 = normal interrupt mask    (0x1018 on this part)
16.  clear the `0x204` coalescing-armed flag (§7.4)
```

**This list is complete, verified at instruction level 2026-09-27.** `mts_init_locked` makes
**exactly seven register writes** and they are steps 9–15 above, in that order: `0x44`, `0x3c`,
`0x48`, `0x40`, `0x34 |= 1`, `0x38 |= 1`, `0x54`. There is no eighth. The remaining steps are
software-side ring and counter work that touches no register.

That is a **clean negative on the whole "we dropped a step" family of hypotheses**, which had
been the standing explanation for TX not working for two builds. It is not a weak absence
either: the enumeration is of every store in the function, not a search for a particular
offset.

**Step 15's `0x1018` is confirmed at its source, not inferred from a readback.** The ROM writes
`0x54` from `softc+0x3098`, and that field is written **once in the entire driver** — at
`0xde165c77` in `mtsc_attach`, as the literal `0x1018`. Nothing else ever assigns it, so the
normal mask is a build-time constant of this part and not something negotiated at runtime.

**And there is no MAC transmitter enable.** See §8 question 6 — that is a separate finding
large enough to have its own entry, and it is what makes this section's completeness
load-bearing rather than merely tidy.

**Step 15 is last, and that ordering is a trap for a Linux port.** The ROM writes
`0x54` *after* both engines are running, and this document reproduces that faithfully
because it is what the hardware is known to accept.

`0x1018` is bits 3, 4 and 12. It does **not** contain bit6, the RX-completion source, and
`0x54` is a mask in which a set bit **masks** its source (§8 question 1) — so `0x1018`
leaves bit6 **unmasked**, and the wider `0x007bfffe` written on first interrupt (§7.4)
**masks** it. Two consequences, and they point in opposite directions:

- **The ROM's order is safe for the ROM.** Its ISR is an internal loop that drains
  everything in one entry, so it does not need a second RX interrupt and does not care that
  `0x007bfffe` masks bit6 for the rest of the session.
- **The order is a hazard for a driver whose poll path re-arms a mask.** If a poll can run
  concurrently with the start path, step 15's trailing write lands after the poll's and
  overwrites it. That is what happened in attempt 13: `napi_enable()` ran before the engines
  were started, the first poll overlapped `gbe_up()`, step 15 landed last, and two
  interrupts fired and then nothing for the remaining 1100+ seconds. Either enable polling
  after step 15, or write the mask the poll path would have written. The failure is silent:
  the engine keeps receiving, descriptors keep completing, and nothing reports an error.

**The hazard is confirmed by experiment.** Attempt 14 ordered `napi_enable()` after the
engines start and changed nothing else, and the readback moved exactly as predicted:

```
[   98.245814] eth0: engines: txctl=0x00000001 rxctl=0x00000008 imr=0x00001018 | ...
```

`imr=0x00001018` — step 15's own value, intact, where attempt 13 read `0x007bfffe` at the
same point and had it overwritten immediately after. The reordering is what let a drain run
at all: attempt 14 is the first boot on which frames reached the stack (§8 B).

**This note previously drew the opposite conclusion from the same facts, and the inversion
matters.** It said step 15 clobbers the re-arm "back to a mask with RX completion
**disabled**". With the polarity now settled, `0x1018` is the value that leaves RX
completion **live**, and `0x007bfffe` is the one that suppresses it. The *ordering* hazard is
unaffected — a late write that overwrites a concurrent one is a race whichever value wins —
but the direction of harm is reversed, and the useful deviation for a port is now the
opposite of what this section used to imply. A driver should write `0x1018` **before**
`DMA_RUN` (nothing can assert before the engines run, so nothing is lost) so that no write
here can land on top of a poll's, **and** it must stop re-arming `0x007bfffe` from the ISR
and the poll path, because that mask is what starves RX of every interrupt after the first.
Build #29 is the first build to do the second half.

### 7.3 Stop (interface down)

```
1.  write 0x54 = 0x7ffffa
2.  write 0x34 = 2 ; poll until bit1 clear   (TX reset)
3.  write 0x38 = 2 ; poll until bit1 clear   (RX reset)
4.  free every TX buffer; set each TX word0 = 0x80000000
5.  free every RX buffer; set each RX word0 = 0x80000000
```

Order matters: **mask interrupts, then reset the engines, then reclaim buffers.**

Step 1 is one of the evidence lines that closed the polarity question — line (c) of §8
question 1, and the ROM writes the same value in the same position in its other teardown
(`0xde1653f0`, line (d) there).
`0x7ffffa` sets every bit the register uses except bit0 and bit2; as a *mask* that is
"suppress almost everything", which is what a teardown wants before it frees the buffers the
sources refer to. Read as an *enable* it would mean "turn nearly every source on, then reset
the engines", which is why the enable reading never made sense here and was never examined.

### 7.4 Interrupt service

```
if not coalescing_armed:                     (softc+0x309c — see below)
        coalescing_armed = true
        write 0x204 = 0x10001388
        write 0x54  = 0x007bfffe      (mask: this MASKS bit6, RX completion)

loop:
        status = read 0x50
        if (status & ~normal_mask) == 0:
                drain RX  (§7.5)
                rmw 0x38 |= 4                (RX doorbell — re-ring, always)
                return
        write 0x50 = status                  (write-1-to-clear)
        if (status & ~normal_mask) == 0x40000 and coalescing_armed:
                coalescing_armed = false     (timer expiry, and nothing else)
                write 0x204 = 0
                write 0x54  = normal_mask    (unmasks bit6 again)
        if status & 0x82:                    (bits 1, 5 — RX events)
                drain RX  (§7.5)
                rmw 0x38 |= 4                (RX doorbell — re-ring, always)
        if status & 0x80:                    (bit7 — TX completion)
                transmit completion  (§7.7)
                if tx_free > 0xff:  clear OACTIVE     (ring fully drained)
                else:               rmw 0x34 |= 4     (TX doorbell — re-ring)
        if status & 4:                       (bit2 — link state)
                link-state handler: read 0x04  (§3.6, §3.6.1)
                read 0x04 again              (a third read of the register)
                lock   softc+0x31a8          ("gbe:phy_ctrl", if_mts.c:4357)
                softc+0x31c8 |= (reg04 & 1) ? 0x100 : 0x1
                wakeup(&softc+0x31c8)
                unlock softc+0x31a8          (if_mts.c:4365)
        if status & 0x7be600:                (error and link events)
                if status & 0x500000:        (bits 20, 22 — TX error)
                        full TX restart      (below)
```

**The `0x50` bit2 branch does two things, and the second one is what arms the ROM's PHY worker
— decoded 2026-10-03.** After the link-state handler publishes (§3.6.1), the ISR re-reads `0x04`
and ORs an event bit into `softc+0x31c8` under the `gbe:phy_ctrl` mutex, then wakes the thread
sleeping on that address: **`0x04` bit0 set → event bit8; bit0 clear → event bit0.** Those are
the two work items `SceGbeMtsPhyCtrl` dispatches — bit8 is link acquisition, bit0 is the
autoneg restart — so this is the only edge-driven arming the ROM's link-repair loop has. The
interface-up path (`if_init`) supplies the other one, writing `0x31c8 = 0x10100` and waking.
Producer inventory, including one event bit with **no producer anywhere in the driver**:
`notes/mts-init-sequence.md`.

**~~A port cannot rely on this arming path on this hardware.~~ WITHDRAWN 2026-10-09 — the
arming path works.** This paragraph said *"`0x50` bit2 has never asserted on any boot of this
project, so the ROM's own trigger for its link-repair loop would not fire here either"*, and
concluded that anything a port does to run that loop has to be armed some other way. **Bit2
asserts.** It did so on attempt 30 and on every boot since where the link came up — four so
far, one assertion each, `status=0x00000004` with `0x04` bit0 set (§3.7.2). The silence through
attempt 29 was the absence of a link *change*, not a property of the part: the link had never
come up. **So the ROM's edge-driven arming is available to a port**, and the departure this
paragraph licensed is not required. Corrections log, `notes/mts-init-sequence.md`.

What a port must still supply for itself is the **other** producer — `if_init`'s interface-up
kick (`softc+0x31c8 = 0x10100`) — and the 3 s idle poll, because bit2 only ever fires on a
transition and a link that is already up at open produces none.

**The two doorbell re-rings above are new to this document, decoded 2026-09-27, and both were
omitted from every earlier revision.** They are not optimisations.

- **The ROM re-rings the TX doorbell after every completion it does not fully drain.** The
  `tx_free > 0xff` test is "is the ring now completely empty"; the *else* branch — the common
  one, whenever anything is still outstanding — writes `0x34 |= 4` again. **The ROM never
  relies on one doorbell surviving.** A port that rings once per transmit and then waits is
  assuming a property of this device that its own firmware does not assume.
- **The ROM re-rings the RX doorbell after every drain**, on both the quiet path and the bits
  1/5 path. This one our driver already did; it is recorded here so it stops looking optional.

**`0x50` bits 20|22 are a full TX restart, not a `0x9c` pulse.** This section previously
recorded only the pulse, which is the first of eight steps:

```
1.  rmw 0x9c &= ~0x40 ; rmw 0x9c |= 0x40     (pulse: clear then set)
2.  free every TX buffer still held
3.  for each of the 256 TX descriptors: word0 = 0x80000000 ; word2 |= 0xffff0000
4.  tx_prod = tx_cons = 0
5.  tx_free = 0x100                             (ROM value; walk 255 — §7.6)
6.  write 0x44 = tx_ring_busaddr             (base)
7.  write 0x3c = tx_ring_busaddr             (current)
8.  rmw   0x34 |= 1                          (RUN)
```

**There is no `0x34 = 2` engine reset anywhere in that sequence**, and that absence is the
interesting part — see §8 question 6. The ROM's recovery from a TX error is to rebuild the ring
and re-assert RUN on an engine it never stopped.

> ### ⚑ THE SEQUENCE IS THE ROM'S. THE REASON FOR RUNNING IT IS NOT — attempt 31, 2026-10-09
>
> **This device never asserts bits 20|22, so the condition that reaches the eight steps above
> in the ROM does not occur on this part.** `isr = 0x00000000` with `imr = 0x00001018` on
> **124 of 124** sample lines at attempt 31 — 62 `txwatch` plus 62 `watch` — **including all
> 36 taken while the transmitter was frozen at index 255 with `txctl` bit3 held set.** The
> mask was loaded throughout, so those bits were unmasked and clear: they are **not
> unserviced, they are not set.** Attempt 32 reproduces it on a boot with no stalls at all —
> `status=` takes only `0x04`, `0x80` and `0x000000c0` across 268 seconds and 1.5 GB, so
> nothing in `0x7be600` ever asserts in either boot.
>
> **Consequence for anyone reading this section.** The eight steps are ROM-faithful and they
> **work** — twelve for twelve at attempt 31, `txctl` back to `0x00000000`, `txcur` back to the
> ring base, `errctl` at `0x0000006f`, transmission resuming each time. But a port that runs
> them does so **from its own watchdog, for a fault this device does not report**, not as the
> ROM's designed response to an error it raises. **The steps are the ROM's; the trigger is
> ours.** Do not describe running this sequence as the designed recovery, and do not wait for
> bits 20|22 to arm it, because they never arrive.
>
> **And as of attempt 32 it is a safety net rather than the mechanism.** With the TX ring's
> logical length at 255 (§5.4) the stall does not occur and this path is never entered.
> **Keep it anyway** — the ROM ships a TX restart of its own, so the hardware is known to need
> one, and it is now the only sequence measured to recover this engine. A recovery that never
> fires on the fixed configuration is the correct state for a recovery to be in.

**`normal_mask` is `0x1018` — bits 3, 4 and 12 — and it is *not* the completion mask.**
(Corrected 2026-09-24; it was labelled "the RX and TX completion sources", which it is not.
TX completion is bit7 and RX completion is bit6, and neither is in `0x1018`.) It is the set of
sources whose only response anywhere in the ROM is "drain RX and return" — the sources that
need no register work. Anything outside it gets acknowledged and handled.

**`write 0x50 = status` is the acknowledge, and it is sufficient — including for bit6.**
Confirmed on hardware, attempt 15: `isr[1] status=0x00000040 after-w1c=0x00000000`, with two
completed descriptors still undrained, and the following poll read `0x00000000` again. §3.7
has the measurement and the three candidate mechanisms it refuted. **This paragraph used to
say the opposite** — "not sufficient for bit6", from attempt 14's `after-w1c=0x00000040` — and
that readback is now read as the source re-asserting between two register accesses while 18
completions were pending, not as a failure to clear.

**What a driver must not copy is the `0x54 = 0x007bfffe` write above.** `0x54` is a mask and a
set bit masks (§8 question 1), so that write **masks bit6**, and the ROM never restores
`normal_mask` except on the `0x204` timer-expiry path — which a port that never arms `0x204`
(§9) can never reach. The ROM gets away with it because its ISR is an internal drain loop
that needs no second interrupt. A one-shot ISR plus NAPI does: reproduce this write and RX
takes **one interrupt per boot**, which is what attempts 13, 14 and 15 each recorded. A port
should either not write `0x54` here at all, or write a value that leaves bit6 unmasked.

**What works, confirmed at attempt 16:** write the service mask on ISR entry as the ROM does,
and restore the **operating** mask (`0x1018`) on poll completion. That single change took the
interrupt count from one per boot to **535**, with `0x54` reading `0x00001018` between
interrupts on all 31 watchdog samples. Unmasking needs no acknowledge first — §3.7.1 has the
measurement and the full protocol.

**A timer fallback is worth keeping regardless, and it has now paid off twice, in opposite
ways.** A 5 s watchdog that calls `napi_schedule()` when an RX completion is asserted and
`rx_packets` has not moved delivered **436 frames in 358 seconds** on attempt 15's driver,
which took one hardware interrupt (§8 B) — that is what proved the datapath independently of
interrupt delivery. Attempt 16 kept it deliberately, as a safety net rather than as part of
the hypothesis, and it reported **`kick=0` on all 31 samples**. An instrument that fires is
evidence; the same instrument staying silent under a fix is what lets "reception is
interrupt-driven" be *stated* rather than inferred from a packet count. Keep it.

**`0x9c` bit6 is a pulse, not a clear.** The ROM clears it and then immediately sets it again.
The spec previously recorded only the clear — and then recorded the pulse as the *whole*
response to bits 20|22, which it is not; it is step 1 of the eight-step restart above.

**The `0x204` armed flag.** `softc+0x309c` is not a "first interrupt since start" latch. It is
an ARMED flag for the `0x204` coalescing timer, and the ROM cycles it continuously: armed on ISR
entry, disarmed when the timer expiry (status bit18) turns up as the **sole** source outside
`normal_mask`. The test is exact equality, not a mask test — bit18 arriving alongside anything
else leaves the timer armed. Arming once and never disarming leaves the timer free-running at
its programmed period, which is an interrupt per expiry with no work attached to it.

### 7.5 RX drain

```
sync RX ring and RX pool for CPU
idx = rx_idx ; n = 0
while n < 256:
        if desc[idx].word0 < 0x80000001: break     (hardware still owns it)
        build a packet from pool + idx*0x600, length word0[10:0]
        apply checksum and VLAN results from word0/word2
        batch[n++] = packet
        re-arm desc[idx] — word0, word1, WRAP if idx==255, then clear OWN (§6)
        idx = (idx + 1) & 0xff ; rx_idx = idx
sync RX ring for device
deliver the batch to the stack with the driver lock dropped
```

Up to 256 frames are batched per pass and delivered **outside** the driver lock.

**The caller re-rings `0x38 |= 4` after every drain**, on every path that reaches one — §7.4 has
it. It is the caller's write, not the drain's, which is why this section did not carry it; a
port that reproduces §7.5 alone will be missing it.

### 7.6 Transmit

```
if tx_free <= 63: return ENOBUFS
idx  = tx_prod
need = round_up(frame_len, 128)
if pool_head + need >= 0xa0000: pool_head = 0
if [pool_head, pool_head+need) would overrun pool_tail:     (see below)
        run transmit completion inline (§7.7) to advance pool_tail
        if it still would overrun: return ENOMEM
copy the whole frame into tx_pool + pool_head
desc[idx].word0 = 0x80000000
desc[idx].word1 = tx_pool_busaddr + pool_head
set length, FIRST, LAST, offload and (if idx == last) WRAP in word0   (§5.4)
pool_head += need
sw_tx[idx].buf = frame
tx_prod = next ; tx_free -= descriptors_used
if tx_free < 128: mark the queue stopped
clear word0 bit31 to hand the descriptor to the device
```

> **`last` is 254, not 255, and a port that reads only this section will hang.**
> The ROM puts WRAP on index 255 and this sequence is a faithful record of what
> it does. **On this silicon that descriptor is never transmitted.** The engine
> runs to index 255, parks there with its active bit still asserted, reports no
> error, and never wraps: measured twelve times on one boot, exactly 255 packets
> apart, with the descriptor byte-identical to what the ROM emits for that slot.
>
> Use **255 of the 256 descriptors**, so WRAP lands on index 254 and index 255 is
> never queued. That costs one descriptor and eliminates the hang across tens of
> thousands of packets. §5.4 has the evidence and §8 has what is and is not known
> about the cause.
>
> This is the one place in this document where following the ROM exactly produces
> a driver that does not work.

**The TX pool tail is an allocation-time guard, not bookkeeping.** This document used to
mention the tail only in §7.7, as something "recomputed on completion", which reads as a
statistic. It is not: the ROM keeps pool **head** at `softc+0x32b0` and pool **tail** at
`softc+0x32b4` (initialised `0` and `0xa0000`), and `mts_tx` **refuses to allocate into pool
bytes the device has not given back**, calling transmit completion inline to try to advance the
tail and failing the transmit outright if that does not free enough. The descriptor count
(`tx_free`) and the pool byte range are two independent resources and the ROM checks both.

A driver that tracks only a head can still be correct — ours was, by arithmetic: 193 maximum
outstanding frames × `ALIGN(1514, 128)` = 296,448 bytes, and 2 × 296,448 < the 655,360-byte
pool, so the head cannot lap a live buffer. But that safety is the product of three unrelated
constants happening to line up — the 63-descriptor stop threshold, the 1500-byte MTU cap and
the wrap rule — and nothing checks that they still do. Change any one and the pool silently
overwrites frames the device is still reading. **Implement the tail.**

### 7.7 Transmit completion

```
sync TX ring for CPU
idx = tx_cons
while desc[idx].word0 & 0x80000000:              (device handed it back)
        if (desc[idx].word2 >> 16) == 0xffff: break   (idle marker: already reclaimed)
        if sw_tx[idx].buf:
                count one transmitted packet
                free sw_tx[idx].buf ; sw_tx[idx].buf = NULL
        pool_tail = desc[idx].word1 - tx_pool_busaddr    (the §7.6 allocation guard)
        desc[idx].word2 |= 0xffff0000
        idx = (idx + 1) & 0xff
        tx_cons = idx ; tx_free++
```

**This function touches no registers at all, and it waits on nothing.** Verified at
instruction level 2026-09-27: `mts_txeof` makes **zero** register accesses and has no
precondition — no status poll, no enable check, no handshake. So "completion needs something
we do not set" is closed as a hypothesis; there is nothing to set.

**The doorbell re-ring lives in the caller** (§7.4): after this returns, the ROM's ISR rings
`0x34 |= 4` again unless the ring came out completely empty. Do not read this section's silence
as meaning one doorbell per transmit is enough — the ROM does not behave that way.

`pool_tail` is not a statistic. §7.6 blocks on it.

### 7.8 RX filter update

```
if promiscuous:
        rmw 0x10 |= 0x10
        return
rmw 0x10 &= ~0x10
if all-multicast:
        for i in 0..255: write 0x1c = 0x7000 | i
        return
write 0x1c = 0x80000000 ; poll until bit17 set        (clear the table)
for each multicast address:
        crc = ether_crc32_be(addr, 6)
        idx = ((crc >> 27) << 3) | (crc & 7)
        write 0x1c = 0x7000 | idx ; poll until bit14 clear
```

### 7.9 Control operations

| Operation | Action |
|---|---|
| interface up | start (§7.2) |
| interface down | stop (§7.3) |
| promisc / allmulti change only | RX filter update (§7.8), no restart |
| add / delete multicast | RX filter update (§7.8) |
| set MTU | accept **46..1500**, reject anything else |
| private link-mode get/set | forced-mode BMCR/ANAR/GTCR writes per §4.3 |

**Jumbo frames are not supported.**

---

## 8. Open questions

> ### ⚑ THE STANDING BLOCKER IS RESOLVED — 2026-10-04, attempt 30, build #46, banner `#49`
>
> **TX works and the link comes up.** Bit4 of Clause 45 `0x0147.1e` — the residue the ROM's
> GBE shutdown routine leaves in the PHY, inherited across every kexec because a kexec never
> power-cycles the part — **inhibits the transmit path**. Clearing it once at the top of PHY
> bring-up produced, in one boot: `tx=71/6014` with `prod==cons`, `free=256`, `drop=0 err=0
> stall=0` on all 66 `txwatch` samples; `reg04 = 0x00000b19` (bit0 **set**) on all 114 samples
> to t=362 s; `bmsr = 0x796d` (bit2 **set**, both reads) on all 113; `speed=2 fdx=1`
> throughout. Attempt 29 is the control — identical arrival state, identical PLL4 write, no
> residue clear, `tx=0/0` and `r04 = 0x00000b18` for its whole boot. The link partner, an
> OpenWrt switch, independently reports the port at `1 GbE` with live traffic both ways.
>
> **Questions 3 and 4 are answered. B's "nothing has been transmitted on any boot" is history.
> C — "why `BMSR` bit2 never sets" — is answered, and it is the same answer.** Those entries
> are rewritten below rather than struck, because the reasoning in them is why the answer was
> eventually found and the negatives they reached are all still correct.
>
> ⚠ **What is NOT resolved: end-to-end IP.** There is no DHCP exchange and no IP address in
> the boot log. The datapath works in both directions and the link negotiates gigabit; the
> console does **not** have demonstrated network connectivity. That needs a DHCP client
> actually run against `eth0` with a lease logged, or a static address plus a successful ping
> to the gateway. Nor is it known what bit4 does *mechanically*. Full scoping in
> `notes/hardware-bringup-log.md`, attempt 30.
>
> ### ⚑ AND THE LINK NOW RUNS AT LINE RATE — 2026-10-09, attempt 32, build #49, banner `#52`
>
> **Attempt 30 got TX working; attempts 31 and 32 found and fixed what it was working
> *through*.** The engine hung on the last descriptor of the 4 KB TX ring page — twelve times
> in one boot, exactly every 255 packets, at `txcur = base + 0xff0`, with the device reporting
> no error at all. **Walking 255 of the 256 descriptors with WRAP on index 254 eliminates it**:
> **zero stalls in 268 s, 4,589 clean ring laps, `tx = 1,170,378 / 1,519,887,431`, `drop=0
> err=0 stall=0 kick=0` across the whole boot, 847.6 Mbit/s from the driver's own byte counter
> and an external browser speed test of 711 Mbit/s down / 768 up** — against **0.37 / 0.26** on
> the same hardware and the same image at `txring=256`. The full evidence is **question 8**
> below, the descriptor side is **§5.4**, and what it does to §7.4's restart is in **§7.4**.
>
> **What this changes about the shape of this section.** Nothing on the critical path is now a
> question about this driver or this document. The one remaining TX item is **why the last
> descriptor hangs**, which is a question about silicon with a labelled inference against it
> (descriptor prefetch crossing the page boundary) and no instrument in this project able to
> test it. **End-to-end IP also moves**: the user's browser completed a speed test against a
> remote server over the public internet, so userspace had working connectivity — the DHCP
> *lease* is still not in any log, which is the narrow thing attempt 30's caveat was about.

**A. ~~EFUSE field packing is KNOWN WRONG — highest-value remaining decode.~~ CLOSED
2026-09-27, decoded and implemented in build #34; confirmed on hardware 2026-09-28.** The
packing is eighteen fields out of four words, **not** the fourteen byte lanes the driver
guessed (`word[i/4]`, shift `(i%4)*8`). The full table, the clamp, the two structural traps
and both trustworthiness arguments are in **§4.2**, "The field packing". Decoded from
`0xde1662c0`–`0xde166728`; the reader routine is `0xde1d9af0`, called with `0x5c`, `0x60`,
`0x64`, `0x68`, `0x6c`.

The question closes on evidence rather than on a decode alone: four of the destination
registers are read-modify-write, and on the part under test they already hold **exactly** the value
the decode computes — Sony's own programming, surviving the kexec. §4.2 (b) has the log lines.

The warning that made this the highest-value item stands and is now a warning about ports
rather than about us: wrong trim values mistune the PHY, and a mistuned PHY **links and then
errors under load**. It is a silent failure that no timeout announces and no boot that "looks
fine" rules out, which is why the read-back comparison is the check that matters.

**One thing does not close with it.** This decode was pursued as the explanation for a link
failure, on the strength of a misread `stat1000` — see §4.4 and the corrections log. That
motivation is retracted. **The decode is unaffected**: both of its arguments are independent
of it, and the packing defects it found were real.

**B. CLOSED — the RX datapath works (2026-09-27), and the TX datapath works (2026-10-04).**
This entry previously read "the device emits descriptor addresses with no ring base added"
and was the project's central blocker for ten boots; it then became "nothing has ever been
transmitted" and was the central blocker for thirteen more. **Both halves are now closed.**

> **The TX half is closed by attempt 30 and the cause was not in this entry's hypothesis
> space.** It was bit4 of Clause 45 `0x0147.1e` — the PHY shutdown residue — and the answer
> lives in question 4 and in §1.1, not here. `tx=71/6014`, `prod==cons` and `drop=0 err=0
> stall=0` on all 66 `txwatch` samples; §5.4 has the descriptor-side confirmation and §3.4 the
> `0x34` behaviour in a boot that transmits. **Everything below about the TX failure is kept as
> the record of twenty-nine boots' worth of correct negatives**, because those negatives are
> what narrowed the search until only inherited state was left. None of them is retracted:
> every elimination the TX narrative reached was right, and it was the hypothesis *space* that
> was short, not the measurements.

The RX half's resolution was two separate things, one a real fix and one a measurement error.

The real fix was the **31-bit DMA clamp**. With a 32-bit mask the rings landed at
`0xffffe000`/`0xfffff000`, every descriptor access faulted in the IOMMU, and the
fault addresses looked like bare `index × 16` with no base. Clamping to
`DMA_BIT_MASK(31)` moved the rings to `0x7fffe000`/`0x7ffff000` and the faults
stopped. Three consecutive boots since show **zero `ps4_gbe` IOMMU faults**; the
`0x10xxxxxx` faults that remain all belong to `xhci_aeolia` at `00:14.7` and are
Orbis-era IOVAs left by the kexec.

The measurement error was **"the current pointer advances one descriptor and
stops"**, repeated across several attempts. It was read microseconds after `DMA_RUN`,
which is simply too early. Attempt 13 shows `cur=0x7fffe010` at engine start and
`cur=0x7fffe190` — **25 descriptors** — later in the same boot, with the head
descriptor re-armed in between, so the ring is cycling.

Two things are deliberately *not* claimed here. The interval is **not** quantified:
this boot's printk timestamps come from a tick-clamped jiffies clocksource (`TSC found
unstable` at t=3.64, `sched_clock: Marking unstable`, `Switched to clocksource
refined-jiffies`) with a 1 ms grain at `CONFIG_HZ=1000`, and the dump lines are spaced
~999 µs apart, which is the clamp rather than a measurement. Any sub-10 ms interval in
this log is at or below the clock's noise floor. What survives is the **direction** of
the movement, which file order and descriptor contents fix independently of the clock.
And **attempt 13's** count is **descriptors, not frames**: both `goto next` paths in the
driver's receive loop (the length check and an skb allocation failure) advance `rx_idx`
without incrementing `done`, so *that* observation cannot distinguish 25 delivered frames
from 25 completed descriptors that were mostly dropped. Only descriptors 0–3 have observed
lengths. **The caution stands for attempt 13 — which had no counter log — and is answered
for attempt 14**, below.

**Completed descriptors do become frames the stack accepts. Confirmed 2026-09-27, attempt
14.**

```
[   98.245833] eth0: rxring[poll]: idx=0 rxctl=0x00000008 cur=0x7fffe120 base=0x7fffe000
[   98.338253] eth0: rxcnt: done=18 pkts=18 bytes=2283 drop=0 lenerr=0
```

Two independent counts agree. The device's own pointer stood `0x120` past the base —
`288 = 18 × 16`, exactly 18 descriptors pending at poll entry — and the drain reported
`done=18 pkts=18`, with `drop=0` and `lenerr=0` meaning **none** of the 18 was lost to the
length check or to an skb allocation failure. `rx_packets` and `rx_bytes` are observed
quantities now, not merely incremented ones.

**Scope this claim tightly.** It covers **18 frames in one poll on one boot**, averaging
~127 bytes — the size of the ARP and multicast-discovery chatter a quiet link carries. It
does **not** establish a sustained receive rate (reception stopped after that one poll —
§3.7), a full ring lap, WRAP handling at index 255, behaviour at any length near the
1536-byte buffer, or that the frame *contents* are correct. `drop=0 lenerr=0` means those
paths were never entered, which makes them untested rather than proven. A captured packet
with verifiable contents is still the stronger evidence, and none of this touches TX.

The related "`0x38` starts the engine and immediately suspends itself" reading is
also retracted. Bit0 never reads back at all (§3.4) and bit3 means active, not
suspended. The engine was running the whole time; the driver was asking the wrong
register bit whether it was.

**Sustained INTERRUPT-DRIVEN reception, confirmed 2026-09-27, attempt 16.** The strongest
honest claim this section can make is now:

> **535 frames were received and delivered to the network stack on 535 hardware interrupts,
> with the operating mask `0x00001018` intact on all 31 watchdog samples and the timer
> fallback reporting `kick=0` on every one of them.**

Note the wording carefully: **not** "zero drops". `drop=` and `lenerr=` are printed only by
the `rxcnt:` dump, which is bounded at 2 samples, so they describe frames 1 and 2 and say
nothing about frames 3 through 535. A bounded counter dump is not a property of the boot.

The paragraphs below are attempt **15**'s record — the timer-driven predecessor — and are
kept because the kick is what proved the datapath independently of interrupt delivery.

**Attempt 15's claim, superseded upward: 436 frames over 358 seconds, every one of them
because a 5-second timer kicked NAPI, in a boot that took exactly one hardware interrupt.**

Evidence. `rx_packets` read 2 at t=20.688247 and 436 at t=368.674276, sampled by a periodic
watchdog 70 times across that span. Two counter dumps:

```
[   20.688247] eth0: rxcnt: done=2  pkts=2  bytes=221  drop=0 lenerr=0
[   30.830902] eth0: rxcnt: done=64 pkts=66 bytes=4567 drop=0 lenerr=0
```

`done` is capped by the 64-descriptor NAPI budget, which is why `pkts=66` exceeds it — the
poll re-entered. `drop=0 lenerr=0` still means those paths were never *entered*, so they are
untested rather than proven.

The datapath was proved directly, by a kick, not inferred:

```
[   30.761219] eth0: watch[2]: isr=0x00000062 imr=0x007bfffe rxctl=0x00000000 cur=0x7fffe020 pkts=2   kick=1
[   35.874284] eth0: watch[3]: isr=0x00000062 imr=0x007bfffe rxctl=0x00000008 cur=0x7fffe520 pkts=337 kick=0
```

One `napi_schedule()` from a timer moved `rx_packets` from 2 to 337 in a single 5.1-second
window. **Polling recovers reception, so nothing downstream of the interrupt was ever
broken.** That was the discriminating test the watchdog was built to run, and it came back
positive.

**Do not read this as working networking, because the shape of it is wrong.** It is a
sawtooth. Across all 70 watchdog samples, `rx_packets` advances **only** in the interval
following a `kick=1` sample and is flat in every interval following a `kick=0` one — frames
arrive when the timer schedules a poll and at no other time. `imr` read `0x007bfffe` on all
70 samples: RX completion masked (§8 question 1), for the whole boot after the first
interrupt. A link that moves traffic in 5-second bursts because a timer remembers to ask has
a throughput ceiling set by the timer period, not by the wire.

**Still not established.** That received frame *contents* are correct — no capture with
verifiable payload exists on any boot. ~~A full ring lap and WRAP handling at index 255: the
highest `cur` observed is `0x7fffeb80`, 184 descriptors past the base, so index 255 was never
reached.~~ **WRAP at index 255 is CONFIRMED as of attempt 30** — the RX ring crossed it,
`cur` `0x7fffef60` (index 246) → `0x7fffe020` (index 2) with twelve frames counted against
twelve descriptors; §5.3 has the arithmetic. Any frame length near the 1536-byte buffer: 436
frames over ~4.5 KB of sampled bytes is ARP and multicast-discovery chatter, averaging ~69
bytes in the second dump. And ~~**anything whatsoever about TX — nothing has been transmitted
on any boot, ever.**~~ **TX is closed as of attempt 30; see the banner at the head of this
entry.** The following sentences describe the inhibited state and are the record of it:
`txctl` read `0x00000001` after RUN and no transmit had been attempted in sixteen boots. Attempt 16
saw `txctl` move `0x00000001` → `0x00000005` (bit2 is the `DMA_POLL` doorbell) with no
transmit attempted, so that is an observation and not evidence about TX.

**Attempt 17 (build #30, banner `#32`) narrowed that to something much sharper, and it is
still not a transmit.** Frames were queued this time — 70 of them over 388 seconds — and the
device took none: `txcur=0x7ffff000` on all 72 samples, exactly the ring base, `prod=70
cons=0`, the head descriptor still reading `w0=30000149` with OWN never handed back, no
`txrec[]` line in the whole boot and no `0x50` bit7 in the whole boot. So the claim above is
unchanged in substance — **nothing has gone out on any boot** — but the failure is now located:
the stack transmits, the descriptor is well-formed by §5.4, and the device does not look at
it. Question 6 above has what the ROM pass then ruled out, and what it did not.

**Attempts 20 and 21 reproduce that exactly and take away its leading explanation.** Build #35:
`prod=28 cons=0 free=228 tx=0/0 drop=0 err=0`, `txcur=0x7ffff000` — the ring base — on all 16
samples. Build #36: `prod=56 cons=0 free=200 tx=0/0`, `txcur` still the base. Two more boots,
same shape, nothing out.

What changed is the explanation. **"The MAC refuses to transmit because it believes the link is
down" is now untested rather than established.** It has been the working assumption since
attempt 17, and §4.4's anomaly removes its support: in the same boot, on the same MAC, with
`BMSR` bit2 clear and `reg04` bit0 clear, **reception works** — 528 frames on 497 hardware
interrupts in attempt 20, 50 on 49 in attempt 21. If a down link bit gated the datapath, it did
not gate this half of it. Either the bit gates TX and not RX, which nothing in the ROM or in
this document supports, or TX is blocked by something else entirely.

**So `0x34` bit0 latching while bit3 never asserts should be re-examined on its own terms**, as
a question about the TX engine's state machine, and not as a corollary of a link theory that no
longer has evidence behind it.

> **Footnote from 2026-10-04:** that instruction was followed and was useful, and the final
> answer went the other way. The link theory was right about the *correlation* — TX and the
> link bits have one common cause — and wrong about the *direction*: the MAC was not refusing
> to transmit because it saw the link down; **one PHY bit was holding both of them down at
> once.** Clearing it fixed them in the same 39 ms. A theory can be unsupported and still be
> pointing at the right pair of symptoms.

#### That re-examination happened — attempt 26, and the answer is that the gate is not in `0x34`

**Build #42 (banner `#45`, 2026-10-03) sampled `0x34`, `0x38`, `0x50`, `0x9c` and `0x1c8` at four
stage points around `gbe_start_engines()` and ten more times out to +4880 µs after RUN.** The
block is quoted verbatim in §3.4 and in the bring-up log, attempt 26. What it establishes:

- **`0x34` bit0 latches at the RUN write and never clears** — `0x00000001` at `post-run` and on
  **all ten** poll samples, with `txctl=0x00000005` hundreds of milliseconds later. The device
  accepts the RUN bit and keeps it.
- **`0x34` bit3 never asserts**, in any sample. **RX, in the same function microseconds earlier,
  goes `0x00` → `0x08` and works** — 54 packets on 52 interrupts, `kick=0`.
- **`0x50` never raises a TX error source.** `0x00000040` once at `post-run`, which is bit6, RX
  completion; zero on every sample after. **The device reports nothing wrong.**
- **`0x9c` and `0x1c8` never move across the sequence**, so neither participates in engine start
  and both come off the candidate list.

> **TX is gated by something outside `0x34`, and the device raises no error about it.**

That is a clean negative and a definitive one. It eliminates, by measurement rather than by
argument: **the RUN write itself** (it latches and persists), **doorbell mechanics** (bit2 reads
back set, and attempt 17 established the first edge was real), and **descriptor format** —
`txd w0=30000149` is valid field by field against §5.4, with OWN clear so hardware owns it,
FIRST and LAST both set, `[15:0] = 0x0149` = 329 bytes inside the `0xff80` cap, and every
offload bit and WRAP clear. Combined with question 6's finding that `mts_tx` makes zero register
accesses and returns success — confirmed from the device side by `prod=56 cons=0` against a head
descriptor the device still owns — **the failure is downstream of a correctly-rung doorbell**.

**What this does not name is a replacement.** The surviving candidates are §8 C's three (something
Orbis set that we never write, a sequencing requirement with no register trace, or state outside
BAR0 entirely), plus whatever gates a TX engine **specifically**, since RX passes through the same
MAC in the same state. The §3.4 reading — **RUN latched but unconsumed**, as against "started then
stalled" — is the first evidence that distinguishes those two states and is where the next
experiment should start.

> **Resolved 2026-10-04, and the headline conclusion of this subsection is the one that held.**
> "TX is gated by something outside `0x34`" was correct: it was gated by a PHY register. The
> replacement candidate was none of the three listed — it was a **fourth** shape nobody had
> enumerated, inherited teardown state, which question 4 opened three boots later. And the
> "RUN latched but unconsumed" reading turns out to be a *symptom* of the inhibit rather than a
> property of the register: in a boot that transmits, `0x34` bit0 does not latch at all (§3.4).
> **Every elimination in this subsection stands.** The descriptor format, the doorbell
> mechanics and the RUN write were all correctly cleared, and attempt 30 confirms each of them
> positively — the device accepted 71 of those exact descriptors.

**And one candidate class is now closed from the firmware side rather than the hardware side —
2026-10-03.** "The transmitter is waiting for a link handshake that this driver never performs"
was the last version of the link theory still standing. The ROM has two readers of `0x04` and
both have now been read end to end; neither writes `0x04`, and the link-state handler at
`0xde1681b0` touches no TX register, no ring, no producer/consumer index and no OACTIVE flag.
**There is no software link gate in this firmware to be missing.** §3.6.1 has the decode and —
importantly — the scope: a *hardware* gate inside the MAC is untouched by this result and is
unfalsifiable by reading firmware, so this closes a hypothesis family about **our software**, not
about the silicon. The candidates above are unaffected.

**~~One instrument gap, logged rather than closed.~~ CLOSED — attempt 27 (build #43, banner
`#46`), 2026-10-03.** Attempt 26's `txwatch` dumped **only `word0`**, so §5.4's
`word2 = 0xffff0000` idle marker had never been sampled on a queued TX descriptor on any boot
and nothing could state the marker was cleared. Build #43 dumps `word1` and `word2` too, and
reads `w0=30000149 w1=7fe00000 w2=00000000` — **the marker *is* cleared**, `word1` is a valid
buffer address inside the 31-bit-clamped aperture, and the descriptor is well-formed in ~~**all
four words**~~ **the three words it sampled. "All four" was one word ahead of the instrument:
`word3` was not in build #43's dump and was not sampled until attempt 29, where it reads
`w3=00000000` — so the claim is now true and was not when it was written** (corrections log,
2026-10-04). §5.4 carries the verbatim line and the field-by-field decode. **The "maybe the
device still sees the slot as idle" worry is answered negative**, and with it the last piece of
"something is wrong with what we hand the device".

**What attempt 16 added, and what it did not.** It made reception interrupt-driven and it
produced the project's **first observed ring lap** — 506 frames in one 5.1-second window with
`cur` running backwards `0x7fffe190` → `0x7fffe130` — but that lap is around a short active
window, so ~~**WRAP at index 255 is still unobserved**~~ **WRAP at index 255 was unobserved
until attempt 30, which crossed it on the RX ring (§5.3)**; attempt 16's highest `cur` is only
25 descriptors past the base. Frame contents, near-MTU lengths, the drop and length-check
paths, and ~~behaviour under load~~ all remain untested. ~~and TX remains wholly untouched.~~
**Load is tested as of attempt 32**: 1,027,818 frames received on 746,594 interrupts with
`kick=0` throughout, 1,170,378 transmitted and 1.5 GB moved in one boot, `drop=0 err=0` on
every sample. Near-MTU lengths are exercised on TX (mean 1,448 bytes) and frame *contents* are
still unread in both directions.

#### ⚑ THE TX HALF CLOSES — attempt 30, build #46, banner `#49`, 2026-10-04

**The cause was not in this entry, in §3.4, in §5.4, in the ring code or in the MAC at all.**
It was bit4 of Clause 45 `0x0147.1e`, the residue the ROM's GBE shutdown routine leaves in the
PHY, inherited because a kexec does not power-cycle the part. One read-modify-write, once,
before the rest of PHY bring-up.

| | Attempts 17–29 (residue present) | Attempt 30 (residue cleared) |
|---|---|---|
| `tx` | `0/0`, every boot | **`71/6014`** |
| `txcur` | `0x7ffff000`, the ring base, every sample | `0x7ffff1e0` → `0x7ffff470`, monotonic |
| `prod`/`cons` | `prod` climbing against `cons=0` | **`prod == cons`** on all 66 samples, `free=256` |
| head descriptor | `w0=30000149`, OWN never handed back | `w0=80000000` — **the device returned ownership** |
| `txrec[]` lines | none, in any boot | `cons` walking 2, 3, 5, 9, 10, 11, 12, 14 in the first 1.2 s |
| `0x50` bit7 | never, in any boot | four standalone `status=0x00000080` interrupts, each before a reclaim |
| `0x34` on doorbell | `0x1`→`0x5`, bit2 sticking, bit3 never set | **`0x00000000`→`0x00000008`** — bit3 |
| `drop` / `err` / `stall` | `0`/`0`/`0` (nothing was attempted) | `0`/`0`/`0` (71 frames went out) |

`txcur / 16 = 71` against `tx = 71` closes the arithmetic from the device side; §5.4 has the
field-by-field confirmation and §5.2's ownership convention is now confirmed on TX.

> **What the twenty-nine failures bought, stated plainly, because it is the useful part.** Every
> negative in this entry and in C below is **correct**. There is no register value this driver
> writes wrongly (class 1, attempt 27). No block is held in reset and no agent is fenced
> (class 3, attempts 28–29). There is no software link gate in the firmware (§3.6.1). The
> descriptor format, the doorbell and the RUN write were all cleanly eliminated and all three
> are now confirmed positively. **The search was sound and the hypothesis space was short by
> exactly one shape: state the firmware's *teardown* leaves behind.** See question 4 for why no
> comparison against a *running* Orbis could have contained it.

**And the caveat that belongs with the result rather than under it: this is not IP
connectivity.** The boot log has no DHCP exchange and no IP address. 71 frames of ~85 bytes
over 302 s is ARP and multicast discovery — a host with an interface up and no address. What is
demonstrated is that **the datapath works end to end and the link negotiates gigabit**, which
the link partner independently confirms. What is **not** demonstrated is that the console can
reach anything. That needs a DHCP client actually run with a lease logged, or a static address
and a successful ping to the gateway, and it is a userspace configuration step rather than a
driver one.

**C. THE LIVE QUESTION — why `BMSR` bit2 never sets. ~~Two causes eliminated, three candidates
standing, five ranked out.~~ ~~As of attempt 23: the entire "our PHY programming is wrong" class
is eliminated, and one unmatched configuration difference remains — VEND2 `0x0403`.~~ ~~As of
attempt 25 the MAC class is eliminated too, by the same method: there is no remaining
hypothesis of the form "this driver programs register X wrongly", on either half of the
device.~~ **As of attempt 27 the whole value-level category is empty, on both halves and on
both sides: `0x0403.1f` BIT(4) is written and changes nothing, and `0x08` bit27 — the last
"something we fail to write" candidate — is struck. Two hypothesis classes survive, and
neither is about a register value.~~ **ANSWERED 2026-10-04, attempt 30 — and the answer is
neither of the two surviving classes. `BMSR` bit2 never set because the PHY arrived carrying
bit4 of `0x0147.1e`, the residue of the ROM's own shutdown. Clearing it sets `BMSR` bit2 and
`reg04` bit0, and starts the transmitter, in the same 39 ms.** A, B and C are now all closed.

> **The shape of the answer, and why this entry's two "surviving classes" did not contain it.**
> They were "a sequencing requirement with no register trace" and "state outside BAR0 entirely".
> The real answer is a **third** thing: state **inside** the PHY's register space, in a register
> **neither init path writes**, left there by a teardown. It is not a sequencing problem — the
> write can go anywhere before the rest of bring-up — and it is not outside BAR0, since the PHY
> is reached through BAR0's MDIO window. Question 4 opened it three boots before it was tested,
> and this entry's enumeration never had a slot for it.
>
> **Everything eliminated below is still eliminated.** `BMCR_ISOLATE`, power-down, `TrFreeze`,
> `SlvDSPreadyTime`, the whole PHY-programming class, the whole MAC-programming class,
> `0x0403.1f` BIT(4), `0x08` bit27, the `0x00a2` gate, the firmware-side link gate, and
> everything in the ranked-out table. Not one of those negatives is retracted by attempt 30 —
> they are what left inherited state as the only place to look.

**The record below is the twenty-nine-boot search that got there. It is kept whole**, because
the negatives are the content and because the methodological lesson in question 4 only makes
sense against them.

~~A and B above are both closed; this is the one that is
open, and it is the project's critical path.~~

The shape of the failure, from §4.4: the 1000BASE-T PCS trains in both directions, frames
arrive, and both link-status bits — `BMSR` bit2 in the PHY and `reg04` bit0 in the MAC — read
down and stay down for ten minutes. The MediaTek identification (§1.1) turns this from a
search of an unknown register space into a comparison against a known driver, and that is what
produced the lists below.

#### Eliminated by direct observation — record as closed

**`BMCR_ISOLATE` (bit10) and power-down (bit11) are both clear, on 189 of 189 samples.**
`bmcr = 0x1040` — autoneg enable plus the 1000 Mb/s speed selector, and nothing else — in every
sample of `logs/uart/putty-20260928-attempt22-gate.log`, from the pre-touch read before this
driver has written anything through to the end of a 592-second capture.

Those two bits are *the* textbook causes of "the PCS trained but `BMSR` says link down": an
isolated PHY completes training and then refuses to present link to the MII, and a
powered-down PHY does the same. MediaTek's own MT7531 bring-up explicitly clears isolate,
which is what put them on the list. **Both are dead, and dead by direct measurement rather
than by argument** — the distinction §8 question 1 draws between positive observation and
inference from silence applies, and this is the good kind.

**`TrFreeze` and `SlvDSPreadyTime` are CLOSED — attempt 23, and they are Orbis's settings,
not ours.** Both were ranked 1 and 2 below on the strength of our values differing from
MediaTek's. Build #38 read all six Token Ring nodes **before this driver writes anything**,
and every one of them already read back bit-for-bit identical to the value we write (§4.1, the
pre-touch block). `postcal` is identical to `pretouch`, so our writes are idempotent no-ops.

```
tr[pretouch] 4 TrFreeze      read=0x00000004 ours=0x00000004
tr[pretouch] 5 DSPreadyTime  read=0x00060671 ours=0x00060671
```

So **`TrFreeze = 0x004` and `SlvDSPready = 0x0c` are Sony's own settings**, arriving in the
part from the Orbis ROM and surviving the kexec. We are not misprogramming them; we are
writing back what was already there. They die the right way — **measured before being
changed** — which is exactly what the read-only build was for.

> **Do not read "MediaTek says `0x5e`" as a to-do.** It is not one. `0x0c` is the value present
> in this part on a console that **demonstrably browses the internet** under Orbis with this
> silicon and this cable. Moving to `0x5e` would move *away* from the only configuration known
> to work on this board. The same argument applies to clearing `TrFreeze`. Anyone reaching for
> either write is proposing to deviate from Orbis on the basis of a different product's driver,
> and needs a reason better than the difference itself — which is now known to be Sony's
> choice, not our defect.

#### The whole "our PHY programming is wrong" class is eliminated

This is the most important consequence in the boot, and it is larger than the two candidates.

**Every part of the PHY's vendor configuration is already correct before our driver touches
it, and the link is still down.** Closed with evidence, each by direct observation rather than
argument: EFUSE trim (§4.2 — four read-modify-write destinations already hold the decoded
value), the paged vendor sequence, the MDIO latch, timing, EEE advertisement (§4.2), isolate
and power-down (above), master/slave resolution (§4.4), `TrFreeze`, and DSP-ready timing.

> **The `0x00a2` gate was on that list and comes off it — qualified, not closed.** Attempt 22
> found bit15 set in 189/189 samples over 592 s. Attempt 23 ran 1362 s and found **exactly one
> transition in 440 samples**: bit15 clears at t=385.2, at the *same sample* `stat1000` loses
> both receiver-OK bits (`0x3800` → `0x0800`). Two registers, one flip, no other transition in
> the boot — so **bit15 tracks 1000BASE-T receiver status**, and reading it as an unrelated
> gate that happens to be set was a window effect of the shorter capture. The *hypothesis*
> attempt 22 killed stays dead: bit15 was set throughout the window in which anything happened,
> so the ROM would have entered its `BMSR` poll exactly as we do. But `0x00a2` is **not** a
> closed question and must not be cited as one. Details in the bring-up log, attempt 23.
>
> **Attempt 24 (build #40, 1114 s, 363 gate samples) found bit15 set in every one and no
> transition at all.** So the correlation with receiver status rests on attempt 23's single
> flip and nothing has reproduced it. Two readings survive and this project cannot yet choose
> between them: bit15 genuinely tracks receiver status and attempt 24 simply never lost the
> receivers, or the attempt-23 flip was coincident with the `stat1000` change for a reason
> neither sample identifies. **`0x00a2` stays off the closed list, and now also off the
> "explained" list.** Details in the bring-up log, attempt 24.

**The link is already down at the moment Linux first looks** (§4.4, "The pre-touch read"). So
it went down at the **kexec boundary**, and the cause lies outside the PHY's register space.
That is where the search goes next. Hypotheses of the form "our PHY initialisation is missing
or mis-setting register X" are not merely unproven — they are answered, because the registers
were read before being written and they were already right.

#### And the MAC class dies the same way — attempt 25, and this is the headline

**`bar0[postinit]` is byte-identical to `bar0[pretouch]` on 31 of the 32 registers §3
documents.** The exception is `0x00`, the MDIO command and data port, and it changes only
because the driver did MDIO — a Clause 22 read *is* a write of `0x00`. Build #41 read all
thirty-two before `pcim_iomap_table()`'s value had been used for anything, and again after
`gbe_hw_init()` returns.

> **Our entire MAC initialisation is a no-op.** All fifteen of §7.1's register-value writes
> land on a register that already holds the value they write.

Worked through rather than asserted, because "idempotent by construction" is the kind of
claim that is wrong once:

| Register | What §7.1 writes | What it already reads | Result |
|---|---|---|---|
| `0x08` | `\|= 0x07597c00` | `0x0f597c00` | the OR adds nothing — Orbis additionally set bit27, which we never touch, **and attempt 27 shows bit27 is still set at `postinit`, so it is not something we fail to write; it survives our init untouched** |
| `0x10` | `(v & 0xffffff6e) \| 0x81` | `0x00000085` | `(0x85 & 0xffffff6e) \| 0x81 = 0x85` exactly |
| `0x14` / `0x18` | `swap16`/`swap32` of the MAC | `0x0000XXXX` / `0xXXXXXXXX` | already `xx:xx:xx:xx:xx:xx` |
| `0x7c` | `25000000` | `0x017d7840` | `= 25,000,000`; the reference clock is already loaded |
| `0x54` | `IMR_MASK_RUNNING` | `0x00001018` | **the operating interrupt mask is already loaded**, bit for bit |

The other ten — `0x200`, `0x50`, `0xac`, `0x04`, `0x78`, `0x0c`, `0x140`/`0x144`, `0x74`,
`0x1d4`, `0x30` — are each a write of a value the register already holds or a clear of a bit
already clear; the per-step table is in the bring-up log, attempt 25.

**`0x54` is the one to dwell on.** Its polarity is §8 question 1 — the most expensive
correction in this project, wrong for fifteen boots. Orbis leaves the register holding
`0x00001018`, which is the value we compute, *including* the decision to leave RX completion
(bit6) unmasked.

**So both halves of the device arrive from Orbis in the configuration Orbis used to pass
traffic, and the link still reports down.**

> **The failure is not in anything this driver writes. It is in what the kexec leaves behind
> that is not a register value.**

That is the single most important sentence in §8 as of 2026-10-02, and it is a narrowing
achieved by measurement, not an argument. It does **not** close the question — what it removes
is a family of answers. What survives — **one of the original three classes, as of 2026-10-04:
class 1 is empty (attempt 27) and class 3 is spent (question 7). Class 2 stands, and the
shutdown-residue question build #45 tests is a new shape rather than a member of any of the
three**:

1. ~~**Something we fail to write** that Orbis never had to, because its own earlier state
   already covered it. The dump produces exactly one candidate of this kind: `0x08` **bit27**,
   set by Orbis and never written by us. One bit.~~ **EMPTY — attempt 27 (build #43, banner
   `#46`), 2026-10-03.** That one candidate is struck. `0x08` reads `0x0f597c00` at
   **`pretouch`** and `0x0f597c00` at **`postinit`**, identical, **bit27 set in both** — so we
   do not fail to write it; it **survives our entire initialisation untouched** and was never a
   divergence. The class had exactly one member and it is gone. (`0x0403.1f` BIT(4), the other
   named straggler, was *written* in the same boot — see the subsection below.)
2. **A sequencing or timing requirement with no register trace** — satisfied by the order the
   ROM does things in, invisible in a before/after comparison of values.
3. ~~**State outside BAR0 entirely**: clocks, resets, the PCIe glue at `00:14.4`, IOMMU context,
   or anything the kexec tore down without leaving a mark in the MAC's thirty-two registers.~~
   §3 is closed as a register map (no second BAR, no hidden aperture), so this hypothesis is
   explicitly about state that is *not* in it. **Narrowed 2026-10-03 by the outside-BAR0 ROM
   sweep to one testable thing — reset and quiesce state in glue BAR 2** — see the subsection
   below. **SPENT 2026-10-04.** Both halves of that one testable thing are now closed by
   measurement: no Sonics initiator agent is fenced (attempt 28 result 3, and **attempt 29
   supplies the measured reference** — both SATA and both USB agents read, on a console using
   its disk, reading `quiesce=0 reset=0 busy=0` exactly as the GBE's does) and the **GBE's own
   reset handles are identified and read released** — low bank `0x20`, high bank `0x74`, agent
   `0x2000`, question 7 below. The four held reset-array indices are not the GBE's under either
   available reading of what they are, so they are no longer a candidate explanation for TX.
   The reset-array half also **reproduced bit-identically on three boots of #44** (attempt 28's
   reproducibility note), so neither half rests on a single sample.

   **What replaced it is not a third thing of the same kind.** The 2026-10-04 read turned up a
   previously unrecorded GBE **shutdown** routine (`0xde1653f0`, question 4 below), and with it
   a hypothesis of a new shape: **shutdown residue** — a bit the ROM's teardown sets on its way
   out, which is invisible to the whole "our programming matches Orbis's running values"
   comparison that emptied class 1, because the part this driver inherits across the kexec is
   post-shutdown and not post-init. That is not state the kexec tore down; it is state the
   kexec *preserved*. Build #45 reads `0x0147.1e` at the pre-touch point to test it.

#### The whole value-level category is now empty — attempt 27, and this is the headline

> **"A register value this driver programs wrongly or omits" is empty for the whole device.**

Attempt 23 closed the PHY half. Attempt 25 closed the MAC half. **Attempt 27 closes the two
named stragglers**: `0x0403.1f` BIT(4) is written (and changes nothing), and `0x08` bit27 is
struck as never having been a divergence at all. Combined with the firmware-side exhaustion
recorded in §3.6.1 and §8 question 6 — `mts_init_locked` makes exactly **seven** register writes
and there is no eighth, `mts_tx` and `mts_txeof` make **zero** register accesses, no MAC
transmitter enable exists, neither of the ROM's two `0x04` readers writes `0x04`, and the ROM
contains no software TX link gate — **there is no value-level hypothesis left on either side,
hardware or firmware.** Class 1 above is not merely unpromising; it has no members.

##### The standing candidate — and it is reasoning, not a finding

**Labelled explicitly: nothing measures this. It is a ranked direction, not a fact, and must
not be cited as one.**

The most informative asymmetry this project holds is that **RX works and TX does not**. A
receiver recovers its clock from the wire; a transmitter must source one. A trained receiver
passing frames indefinitely while the transmitter latches `DMA_RUN` (§3.4), never asserts its
active bit, and raises **no error source** in `0x50` is the shape of a transmit path **whose
clock never ticks** — no error, because by the device's lights nothing is wrong.

`0x7c` is already correct at 25,000,000 (§7.1), **but a *reference* clock is not a *transmit*
clock**, and §3 is closed as a register map, so **no TX clock enable exists in BAR0**. That
points at classes 2 and 3 above and specifically **outward**: the PCIe glue at `00:14.4` that
this project already reads EFUSE through, and Orbis code **outside** the `mtsc`/`mts` range —
every function disassembled so far lives inside it.

**Also measured, and it removes a second-order worry:** `0x04` reads `0x00000b18` on all 31
`stage` samples of attempt 25, from before the PHY identifier read to after the whole paged
sequence, so **PHY calibration changes no bit of `0x04`** and §7.1 step 9's hoisted-read
divergence from the ROM is unobservable on this part. See §7.1.

**New values, and two that were not new.** `0x34 = 0x38 = 0x00000000` at pre-touch: Orbis
**stopped both DMA engines** in its teardown, so this driver starts them from a genuinely
stopped state. `0x9c = 0x0000006f` — the error/link-handling register this document knows only
through a bit6 pulse in §7.4 — and `0x1c = 0x0003207b` are first-ever readings and are
**undecoded**. But `0x1c8 = 0x00a00000` and the ring **bases** `0x44 = 0x10000000` /
`0x48 = 0x10004000` were **already in §3's Live value column** from the phase-1 survey and are
*unchanged*, so they are a confirmation across ten days and eleven builds rather than a
discovery; the aperture was never inferred from IOMMU fault addresses. The ring **current**
pointers `0x3c`/`0x40` do differ from the recorded values, which §3.5 predicts because the
device advances them. Details and the corrections this supersedes: the bring-up log, attempt
25.

#### VEND2 `0x0403` answers, and it is the one difference not already matched

**`0x0403.1f` — `CORE_PLL_GROUP4` — reads `0x1000`.** Not `0000`, not `ffff`, and with
`mdio=0`. A live register block this project had never read or written:

```
cl45[pretouch]: 0189.1e=0110 0033.1e=0177 0403.1f=1000 pll4-bypass=0 pll4-poweroff=0
```

**`pll4-bypass = 0`, where MediaTek's MT7531 bring-up *sets* that bit.** This is the **first
configuration difference found that is not already matched** by Orbis's own programming —
everything else on the list above turned out to be a value we and Sony agree on.

##### VERIFIED against primary source — 2026-10-03

The sentence above **holds**, checked against `drivers/net/dsa/mt7530.{c,h}` fetched raw from
`git.kernel.org` and cross-read at **five revisions** — v5.12, v6.1, v6.6, v6.12 and master —
plus the local Linux 5.4.247 tree this project builds from. The bit layout, mainline master
`mt7530.h` lines 727–733:

```
CORE_PLL_GROUP4                  0x403
  RG_SYSPLL_DDSFBK_EN            BIT(12)
  RG_SYSPLL_BIAS_EN              BIT(11)
  RG_SYSPLL_BIAS_LPF_EN          BIT(10)
  MT7531_RG_SYSPLL_DMY2          BIT(6)
  MT7531_PHY_PLL_OFF             BIT(5)
  MT7531_PHY_PLL_BYPASS_MODE     BIT(4)
```

So **bypass is BIT(4) and power-off is BIT(5)** — the bit positions this driver already used
are correct and its macro names match mainline exactly. The bring-up write is in
`mt7531_setup()` (master `mt7530.c` lines 2810–2829), a hand-rolled read-modify-write that
does `val |= MT7531_RG_SYSPLL_DMY2 | MT7531_PHY_PLL_BYPASS_MODE` and then
`val &= ~MT7531_PHY_PLL_OFF`. **After MediaTek's bring-up: bit4 = 1, bit5 = 0, bit6 = 1.**
Bits 4 and 5 are stable across all five revisions; bit6 (`DMY2`) exists only in v6.12 and
master. There is **no `core_clear` against this register anywhere in the file**, which closes
the standing "maybe the polarity is backwards" worry.

Applied to our observed `0x1000`, MediaTek's sequence would produce **`0x1050`**. Build #43
writes **only BIT(4)**, deliberately: bit4 is present in every revision, whereas bit6 is named
"DMY2", appeared recently, and setting it would be speculative.

**The numeric decode of `0x1000` is solid; the *meaning* of bit12 on this silicon is not.**
`0x1000` is BIT(12) only, which mainline names `RG_SYSPLL_DDSFBK_EN` — but that macro carries
**no `MT7531_` prefix**, while every MT7531-specific bit in the block does. That implies bits
12/11/10 are MT7530-documented and their applicability to MT7531's version of `0x403` is not
established by the source. **Marked inferred:** "bit12 = `DDSFBK_EN` on this part" is
inference, not fact — recording it as fact would be the same shape of mistake as the EFUSE
inference (see the corrections log). What *can* be said positively: the only other write to
this register in mainline is the MT7530 TRGMII path in `mt7530_setup_port6()`
(`mt7530.c:425–426`), which writes `RG_SYSPLL_DDSFBK_EN | RG_SYSPLL_BIAS_EN |
RG_SYSPLL_BIAS_LPF_EN` = `0x1C00` — our `0x1000` plus the two bias bits. So bit12 set with the
rest clear is squarely inside the range MediaTek itself treats as valid.

**Two cautions, and they are load-bearing.** The first one's *mechanism* was recorded wrongly
until 2026-10-03 and is restated here; its conclusion survives for a stronger reason.

1. **This register is not reached the way we reach it, and the match of coordinates is partly
   definitional.** ~~MT7531's PLL group 4 handling is switch-level — part of bringing up the
   switch's core PLL, not a discrete PHY's.~~ **That was wrong.** MT7531's switch COREPLL is
   brought up in a separate function, `mt7531_pll_setup()`, which touches only
   `MT7531_PLLGP_EN` and `MT7531_PLLGP_CR0` and never goes near `0x403`. The `0x403` access is
   a distinct step that mainline's own comment calls the "**PHY core PLL**", so bits 4/5/6 are
   the PHY block's PLL, not the switch's. **The conclusion — that this may not apply to an
   on-die PHY reached through this MDIO window at all — stands, and now rests on the
   transport.** In mainline the register is reached on a dedicated **control pseudo-PHY** at
   MDIO address `MT753X_CTRL_PHY_ADDR(addr) = (addr + 1) & 0x1f` (`mt7530.h:735`) — adjacent
   to, not one of, the five user-facing gigabit PHYs. And it is not a host Clause 45 frame at
   all: `mt7531_ind_c45_phy_read/write` drive the **switch's own** `MT7531_PHY_IAC` register at
   switch offset `0x701C`, composing `MT7531_MDIO_CL45_ADDR`/`_READ`/`_WRITE` command words and
   polling `MT7531_PHY_ACS_ST`. It is a Clause 45 transaction generated internally by the
   switch's MDIO master, over the switch register interface, aimed at a pseudo-PHY on the
   switch's internal bus. **Consequence, and it tempers build #43's result in advance:** our
   read landing at the same `(0x403, devad 0x1f)` coordinates is *partly definitional, not
   corroboration* — `MDIO_MMD_VEND2 == 31 == 0x1f`, so the devad matches by definition of the
   vendor MMD rather than by any MediaTek-specific agreement. Nothing in mainline says anything
   about this register on an on-die PHY reached by host MDIO, so a readback showing the bit took
   is **not** by itself evidence that the PS4 part implements MediaTek's bit semantics there.
2. ~~**We have never written it.** Unlike the Token Ring rows, this is a register we have only
   read, so "bypass is clear" is an observation about the part's state and not yet a difference
   between our programming and Orbis's.~~ **Answered — we wrote it at attempt 27. See below.**

~~Ranked as the standing candidate because it is the only unmatched one left, not because the
mechanism is understood.~~

##### ANSWERED on silicon — attempt 27 (build #43, banner `#46`), 2026-10-03

**The bit is writable, and it changes nothing. A clean negative, and the caution above called
the result in advance.** From early boot — before `logs/uart/latest.log`'s window opens, so
quoted from the source capture `<the serial capture>`:

```
[    2.625235] cl45[pretouch]: 0189.1e=0110 0033.1e=0177 0403.1f=1000 pll4-bypass=0 pll4-poweroff=0 mdio=0
[    2.715237] pll4: 0x0403.1f 1000 -> 1010 bypass=1 poweroff=0 took=1 ret=0
```

`0x1000 → 0x1010`, `took=1`, `ret=0`, with the pretouch read immediately before still showing
`0x1000`. **The write landed.** As a positive fact about the silicon, at exactly its own
strength and no more:

> **`CORE_PLL_GROUP4` BIT(4) is writable at Clause-45 `0x0403` devad `0x1f` over a host
> Clause-45 frame.**

That was genuinely unknown before this boot — the register had only ever been read. **Note
carefully what it is not.** Caution 1 above stated in advance that a positive readback could
prove only this weaker claim, because mainline reaches `0x403` through the switch's own MDIO
master at a control pseudo-PHY and our `(0x403, devad 0x1f)` coordinate match is **partly
definitional** (`MDIO_MMD_VEND2 == 31 == 0x1f`). **The prediction was exact: the boot proved
precisely that and nothing stronger.** "Bit4 *means* PLL bypass on this part" remains
**unestablished** — writability is not semantics, and recording it as semantics would be the
same shape of error as the EFUSE inference (corrections log).

**And nothing downstream moved.** Counted over the whole 353 s capture:

```
reg04=0x00000b18    110 / 110 samples        up=0  on 111 / 111
bmsr=7969/7969      110 / 110                (autoneg complete bit5 set, link status bit2 clear, on BOTH reads of a latch-low bit)
stat1000=3800       109 / 110                (the one outlier is 38ff, at linkopen[0] — the init-window artefact of §4.4, as in attempt 24)
RX                  pkts=52  irqs=45  kick=0
TX                  prod=56  cons=0  tx=0/0  on all 64 txwatch samples
```

`0x04` read `0x00000b18` on every `stage` sample of init as well, so the PLL write changes no
bit of `0x04` either, and the `es[]` engine-start block is **byte-identical to attempt 26's**.
Full sample counts and the verbatim blocks are in the bring-up log, attempt 27.

**So the last unmatched configuration value in the device is now matched, and the link is still
down.** This is the second half of what empties class 1 above; `0x08` bit27 is the other.

#### Class 3 narrows to reset and quiesce state in glue BAR 2 — the outside-BAR0 sweep, 2026-10-03

Attempt 27 left two classes and pointed class 3 **outward**, at Orbis code beyond the
`mtsc`/`mts` range. That sweep has now been done over the whole 7.4 MB of Orbis 9.00 kernel
text. Facts, decode addresses and the proofs of completeness are in
[`mts-init-sequence.md`](mts-init-sequence.md) ("Outside BAR0") and in the
working record `outside-bar0-draft.md` (not published). What it
changes here:

**The external surface of this device is three things, and all three are already shipped.**
Outside BAR0, the entire Orbis ROM does exactly: five read-only EFUSE words from glue BAR 4
`+0xc000`; the `Max_Read_Request_Size` bump to 2048 (§9, already replicated via
`pcie_set_readrq()`); and bus-master enable plus one MSI vector. **There is no fourth thing.**
So class 3 cannot be "an outside-BAR0 *write* Orbis performs that we skip" — no such write
exists anywhere in the image.

**What is left in class 3 is state, not a step.** Glue **BAR 2** carries a per-southbridge-block
hardware **reset array** at `+0x180000` (`bit0 = 1` holds a block in reset, two handles per
block) paired 1:1 with a **Sonics Silicon Backplane initiator-agent array** at `+0x140000`
(`0x400` stride; control `+0x20` with bit4 quiesce and bit0 reset, status `+0x28` with bits 6:4
busy). The GBE must have an entry in both. ~~**Nothing in the ROM ever writes the GBE's**~~ —
**corrected 2026-10-04: the ROM's power-down walk asserts them (`0x20`, `0x74`), and nothing in
the ROM ever *releases* them. That correction is what identified the GBE's index; question 7
has it.** And
~~**this project has never mapped BAR 2 at all** — it maps only BAR 4, for EFUSE.~~ **It maps
BAR 2 as of build #44** — see attempt 28 below, which read both arrays. The sentence is struck
rather than deleted because the experiment it motivated is recorded underneath it.

**This makes class 3 an experiment rather than a hypothesis.** Build #44 dumps the whole
`+0x180000` array (32 dwords, both banks) and the nine decoded `+0x140000` agents' `+0x20`/`+0x28`
read-only at the pre-touch point, in the shape attempt 25 used for `bar0[pretouch]` /
`bar0[postinit]`.

- **It self-calibrates.** SATA's reset pair (`0x2c`/`0x6c`, `0x30`/`0x70`) and USB's
  (`0x24`/`0x64`, `0x28`/`0x68`) are decoded from those devices' own drivers, and this console
  boots from its disc drive and enumerates USB — so those four entries **must** read released.
  The dump therefore tells you what "released" looks like on this board *and* which of the four
  unidentified entries is anomalous, without needing a second reference console.
- **All-zero closes the family by measurement.** If every reset entry reads `bit0 = 0` and every
  agent reads `bit0 = bit4 = 0`, the whole clock/reset branch of class 3 is a clean final
  negative, established by reading hardware rather than by argument — the same standard that
  emptied class 1.
- **There is deliberately no BAR 2 *write* accessor, and that is a safety decision, not an
  omission.** The reset array is shared with SATA and USB. Releasing a reset that is already
  released is harmless, but asserting the wrong entry would reset a block the running kernel
  depends on. Read-only, and nothing in this document authorises a write to `+0x180000`.

**One hypothesis family dies outright in the same sweep: MAC↔PHY port mode and interface
clocking.** An image-wide string sweep for `rgmii` / `RGMII` / `sgmii` / `trgmii` / `gmii` /
`GMII` / `mdio` / `MDIO` / `txclk` / `refclk` / `clk_` / `clock_en` / `pwrgate` /
`power_domain` / `deassert` / `rstn` returns **zero hits relating to Ethernet**; the only `MII`
string in the image is FreeBSD's generic `"MII bus"` device description, which this driver's
ancestor does not use. **There is no MAC-to-PHY interface-mode or transmit-clocking step in this
firmware to be missing** — eliminated by measurement over the image, not by reasoning about what
a MAC ought to need.

**And nothing outside the driver reaches this device by PCI identity.** `0x90d8` occurs as an
instruction immediate **exactly once** in the whole text segment, in `mtsc_probe`. No second
translation unit knows the GBE exists.

**If the BAR 2 dump comes back clean**, the remaining space holds no software step this firmware
performs, and the next move is not more disassembly — it is the instrument gap §8 B already
names: sampling `word1`, `word2` and `word3` of a **queued** TX descriptor, which has never been
done on any boot.

**It did not come back clean. It came back half clean — attempt 28, below.** The agent half met
the all-zero condition; the reset array did not. The prediction is left standing above, unedited,
because it set the pass/fail criterion in advance and the value of having done that survives the
result: **the agents close negative by measurement and the reset array becomes §8 question 7.**
(`word1` and `word2` were separately sampled at attempt 27 and are no longer the gap; ~~`word3`
still is — it is added to the TX dump in build #45.~~ **`word3` was sampled at attempt 29:
`w3=00000000` on all 22 `txwatch` samples. The gap is closed and the queued TX descriptor is
now sampled complete in all four words.** It confirms the driver's own `d->word3 = 0` write
landed and says nothing about the device, which is the strength it is recorded at.)

**And the reset half closed too, one identification pass later — 2026-10-04.** The pass did not
name the four held entries; it found the **GBE's** entries instead, from the one device-specific
callback spliced into the `0xddff8760` walk, and they read released. So the all-zero criterion
above was the wrong census but the right question, and the answer it was reaching for is yes:
**the whole clock/reset branch of class 3 is a clean final negative, established by reading
hardware.** Question 7 has it.

#### The experiment ran — attempt 28, build #44, banner `#47`, 2026-10-04

The dump above was taken. **It came back half clean, so class 3 does not close**, and what
replaces it is a numbered open question rather than an answer. Full figures, the verbatim
blocks, the index arithmetic and the counter-argument are in the bring-up log, attempt 28.

**BAR 2 is mapped and answering, and that is load-bearing for everything below.** The extra
probe read exists only to separate a live window from a dead one, and it reports
`+0x10a000 = 0x00001249` — non-zero — with `ret=0` on **all thirty-four** reads in the dump.

> **Every zero in the arrays below is a read zero, not a dead read.**

This project had never mapped glue BAR 2 before this boot.

**Result 1 — four of thirty-two reset entries are held.** `bit0` set, and no entry anywhere
in the array holds any other bit. The held entries are array indices **0, 15, 17 and 19**, byte
offsets `+0x180000 + 0x00`, `+0x3c`, `+0x44`, `+0x4c`. Each `glue2[rst]` line covers eight
dwords, so `+000` is indices 0–7, `+020` is 8–15, `+040` is 16–23 and `+060` is 24–31.

**Result 2 — the calibration passed, which is what makes result 1 trustworthy.** USB's
`0x24 + 4k` and SATA's `0x2c + 4k` — indices **9+k** and **11+k** — and their high-bank
partners at `0x64 + 4k` and `0x6c + 4k` (indices 25+k, 27+k) all read **released**, all eight
of them, on a console that boots and reads its own disk and enumerates USB. They had to, or
the decode was wrong. **So the array decode holds and the four non-zero entries are a real
reading rather than a misread array.**

**Result 3 — no initiator agent is fenced, and this half *is* a clean negative.**
`quiesce = 0`, `reset = 0`, `busy = 0` on all nine agents read; `status` bits 6:4, the busy
field the ROM polls to `(v & 0x70) == 0`, are zero on every one. The `ctrl` values are
**structured rather than random** — three groups, `0x00000000` ×1 / `0x02000400` ×2 /
`0x06040400` ×6 — which is a cross-check that the window is being read correctly and **not** a
decode; `bit4` and `bit0` remain the only two bits of `+0x20` this project has ROM evidence
for. The varying bits in `status` sit at **15:12** (`7 9 e 4 b 5 a 0 5`), every other nibble
zero. **Inference, labelled as such:** that **looks like** a per-agent ID or revision field
rather than a status condition. Nothing measures it; it is recorded so nobody mistakes those
bits for the busy indication, which is 6:4 and is zero everywhere.

**Result 4 — the read-only instrument perturbed nothing.** `bar0[postinit]` is byte-identical
to `bar0[pretouch]` on 31 of the 32 registers §3 documents, the exception being `0x00` because
a Clause 22 read writes the MDIO port; link state and every counter are unchanged from attempt
27. Thirty-four BAR 2 reads at the pre-touch point changed no bit of BAR0. Stated as a
confirmation rather than assumed, because a dump that disturbed what it samples would have
invalidated the whole measurement and nothing else in the boot would have shown it.

**One instrument defect, and it bounds result 3.** `gbe_glue2_dump()` walks
`base = 0x140000 + i * 0x400` for `i` in 0–8 — the first nine slots at stride — not the nine
decoded agent bases this section asked for. So **four** decoded agents were read (`0x0c00`,
`0x1000`, `0x1400`, `0x2000`, the partners of reset handles `0x14`/`0x18`/`0x1c`/`0x20`) and
**five were not**: `0x3800` and `0x4800` (SATA), `0x4000` and `0x4400` (USB), and `0x3c00`.
Consequences: the **agent half of the dump does not self-calibrate**, because the USB and SATA
agents were the control for what an un-fenced agent looks like; and `0x3c00`, one of the five
GBE candidate agents, is unread (its reset handle `0x34`, index 13, was read and is released).
Result 3 stands on nine real agents and the ROM's bit decode, not on a measured reference. The
fix is to walk the decoded bases; the reset-array half reads all thirty-two words and is
unaffected.

#### The agent half NOW SELF-CALIBRATES — attempt 29, build #45, banner `#48`, 2026-10-04

Build #45 replaces the stride with a `static const u32 gbe_ssb_base[9]` holding the nine
decoded bases, and the dump was retaken on hardware:

| Agent | Offset | `ctrl` | quiesce | reset | busy |
|---|---|---|---|---|---|
| `ssb0` | `+140c00` | `0x06040400` | 0 | 0 | 0 |
| `ssb1` | `+141000` | `0x06040400` | 0 | 0 | 0 |
| `ssb2` | `+141400` | `0x06040400` | 0 | 0 | 0 |
| `ssb3` | `+142000` — **the GBE's** | `0x06040400` | 0 | 0 | 0 |
| `ssb4` | `+143800` — **SATA 0** | `0x06040400` | 0 | 0 | 0 |
| `ssb5` | `+143c00` | `0x06040400` | 0 | 0 | 0 |
| `ssb6` | `+144000` — **USB 0** | `0x06040400` | 0 | 0 | 0 |
| `ssb7` | `+144400` — **USB 1** | `0x06040400` | 0 | 0 | 0 |
| `ssb8` | `+144800` — **SATA 1** | `0x06040400` | 0 | 0 | 0 |

> **The Sonics surface is closed with a MEASURED REFERENCE, not on the ROM's bit decode
> alone.** Both SATA and both USB agents — the controls that **must** be unfenced on a console
> that is reading its own disk and enumerating USB while the line prints — are now in the set,
> and they read `quiesce=0 reset=0 busy=0` exactly as the GBE's own agent `+142000` does. This
> is the calibration result 3 lacked, and it is the same standard the reset-array half met at
> attempt 28 and the same standard that emptied class 1.

`ctrl` is **uniform `0x06040400`** across all nine real agents, where #44's stride produced
three groups. `status` bits 15:12 vary (`3 b 7 f 3 b f 7 c`) with every other nibble zero,
consistent with attempt 28's inference that those bits are a per-agent ID or revision field
and not a status condition. Still inferred; still nothing measures it.

> ### ⚠ COMPARE THESE DUMPS BY ADDRESS, NEVER BY LABEL — and here is the concrete example
>
> The fix renumbers every `ssbN` label: **not one of the nine means the same address in #44
> and #45.** `0x0c00` was `ssb3` and is now `ssb0`, `0x1000` was `ssb4` → `ssb1`, `0x1400`
> was `ssb5` → `ssb2`, `0x2000` was `ssb8` → `ssb3`; the other five addresses each build read
> are unique to it. Both builds print the absolute `+%06x` on every line.
>
> **The example, and it was waiting to be misread.** `ssb0` reads `ctrl=0x00000000` in #44's
> dump and `ctrl=0x06040400` in #45's, and `ssb1`/`ssb2` go `0x02000400` → `0x06040400`.
> **None of that is a state change between boots.** #44's `ssb0`/`ssb1`/`ssb2` were
> `BAR2+0x140000`/`0x140400`/`0x140800`, which are **not agents at all**; #45's are
> `+0x140c00`/`+0x141000`/`+0x141400`, which are. Read by label, this looks like a fabric
> control register that changed between two boots of one console — a plausible, checkable,
> entirely fictitious lead. Read by address, nothing moved: the four addresses both builds
> read agree on `quiesce`, `reset` and `busy`, and `ctrl` differs only where the addresses do.
> **#44's three `ctrl` groups were an artefact of reading non-agent addresses, and the real
> array is uniform.**

#### Ranked out, with the reason — so nobody re-raises them

These came out of the same comparison and are **not** on the critical path. The reasons matter
more than the verdicts, because each is a different argument.

| Candidate | Why it cannot be the cause |
|---|---|
| Auto-downshift not configured | Downshift is a **fallback** — it drops a failing gigabit link to 100. A fallback cannot hold a link down; at worst its absence means we do not get the consolation prize. |
| Link-down power saving not enabled | MediaTek **enables** it. Not enabling a power saving is not a mechanism by which link is suppressed. |
| MCC near-echo (`0xa6`) | Our **receive path works** — frames arrive on real interrupts. Whatever the echo cancellation is doing, it is not deaf. |
| `100_mse_threshold` (`0x123`) | That is the 100BASE-TX equivalent of `0x122`. We write the **1000** one, which is the speed we are negotiating. |
| TX pair delay, `0x13`/`0x14` = `0x404` | **Low for link, and still untested territory for TX.** ~~Nothing has ever been transmitted on any boot (§8 B)~~ — attempt 30 transmits 71 frames with these values unchanged, so **this is now ruled out as a transmit blocker** and remains untested as an analog *tuning* question. Parked with the TX work, not with this question. |
| `0x0268`.1f — MT7531 writes `0x3aa`, we write `0x07f4` | Same bucket. "Adjust TX class AB driver 1" (§1.1) is a transmit analog setting, and ~~on a device that has never transmitted~~ **attempt 30 transmits with `0x07f4` in place**, so it is not a blocker. A real divergence, filed against TX as a tuning question. |

1. **CLOSED 2026-09-27 (attempt 15) — reg `0x54` is a MASK, and a SET bit MASKS its source.**
   Every reading of this register as an *enable* anywhere in this repository was wrong, and
   the driver's `IMR_*` constant names still encode the inverted reading (renamed in build
   #29). The values are unchanged — `0x1018` at start, `0x007bfffe` on first interrupt,
   `0x7ffffa` at stop, bit12 cleared on suspend — only their meaning is inverted.

   **Three independent lines of evidence, and a fourth corroboration added 2026-10-04.**

   **(a) A delivered interrupt for a bit that is not in the loaded value.** Attempt 15 took
   exactly one hardware interrupt in its whole boot:

   ```
   [   20.595718] eth0: engines: txctl=0x00000001 rxctl=0x00000008 imr=0x007bfffe | ...
   [   20.595723] eth0: isr[1]: status=0x00000040 after-w1c=0x00000000 imr=0x007bfffe ...
   ```

   `status=0x00000040` is **bit6**, RX completion. The driver writes `0x54 = 0x1018`
   immediately before `DMA_RUN` (a deliberate reordering of §7.2 step 15 — see that section),
   so `0x1018` was the value loaded when the source asserted. **`0x1018` is bits 3, 4 and 12.
   Bit6 is not in it.** Under "1 = enabled", bit6 was therefore disabled and the interrupt
   could not have been delivered. It was delivered. The `engines:` line is the corroboration
   from the other side: it reads `0x54` back *after* that same `0x1018` write and finds
   `0x007bfffe`, which is the value `gbe_isr()` writes on entry — so the ISR had already run
   by then, in the `0x1018` window, exactly as the timestamps say.

   **(b) The mask that the ISR and the poll path both write masks RX completion, and nothing
   restores the start value.** `0x007bfffe` sets bits 1–17 and 19–22 — everything the register uses except bit0, bit2 and bit18 (the `0x204` timer, §7.4) — and that includes bit6. `gbe_isr()` writes
   it on entry and `gbe_poll()` re-writes it on `napi_complete_done()`; the only write of
   `0x1018` is in the start path, which runs once. So from the first interrupt onward, bit6 is
   masked for the remainder of the boot. That predicts **exactly one RX interrupt per boot**,
   which is what attempts 13, 14 and 15 each recorded — three boots, three different builds,
   one interrupt each. Under "1 = enabled" there is no account of why the second interrupt
   never comes; under "1 = masked" it is the arithmetic.

   **(c) §7.3 step 1 makes sense only under "1 = masked".** Interface-down writes
   `0x54 = 0x7ffffa` as its *first* action, before resetting either engine. Under "1 =
   enabled" that enables nearly every source in the register at the exact moment the driver
   is tearing the device down and freeing the buffers those sources refer to, which no
   teardown sequence would do. Under "1 = masked" it is the obvious thing: mask almost
   everything, then reset. Within the bits the register uses, the only ones `0x7ffffa` leaves **clear** are bit0 and bit2; it differs from `0x007bfffe` by setting bit18 and clearing bit2.

   **(d) A second teardown, in a second function, writes the same value in the same position —
   added 2026-10-04.** The GBE quiesce routine at `0xde1653f0` (question 4, and the only caller
   of it is the ROM's southbridge power-down walk) opens with `0x54 = 0x7ffffa` *before* it
   touches either DMA engine, then resets TX and RX exactly as §7.3 does. This is **line (c)'s
   argument occurring a second time in code that was not read when (c) was written** — it is a
   new instance, not a new kind of argument, and it is recorded at that strength. What it adds
   is that the "mask almost everything, then reset" ordering is the ROM's teardown idiom rather
   than one function's habit: `0x7ffffa` is what this device's firmware writes to `0x54`
   whenever it is shutting the part down, in both places it does so.

   **Why this time is different, and it is not a matter of having more data.** This question
   looked settled twice before *in the other direction* — build #18, then attempt 13 — and
   both times the argument had the same shape: **a source did not fire, therefore the mask
   disabled it.** That is an inference from silence, and silence has other causes. Build #18's
   died when the timestamps showed the storm ran at a flat ~1 ms under both mask values (and
   the decode then showed bit18 is the `0x204` timer, which `0x54` does not gate at all).
   Attempt 13's died when attempt 14 showed a source enabled *and* asserted for 123 seconds
   delivering nothing, which broke the premise that an enabled source would have fired.

   Attempt 15's evidence is of the opposite kind: **a positive observation.** An interrupt was
   *delivered*, for a bit *absent* from the value in the register. That cannot be explained by
   any amount of missing traffic, any stuck level, or any property of MSI delivery — those all
   explain why something did **not** happen, and nothing here rests on something not
   happening. Line (a) alone settles the polarity; (b) and (c) are consistent independent
   confirmations rather than the load-bearing argument.

   **~~What this does not settle.~~ SETTLED 2026-09-27 by attempt 16 (build #29, banner
   `#31`), and settled by a positive experiment rather than by argument.** The remaining
   doubt was whether unmasking bit6 re-delivers a *level that is already asserted* under
   edge-triggered MSI (`IR-Baikal-MSI`, per `/proc/interrupts`). Build #29 changed exactly one
   behaviour — `gbe_poll()`'s completion path restores `0x1018` instead of leaving `0x007bfffe`
   loaded — and the result is unambiguous:

   | | attempts 13, 14, 15 | attempt 16 |
   |---|---|---|
   | `0x54` between interrupts | `0x007bfffe` | **`0x00001018`**, on 31 of 31 samples |
   | hardware interrupts in the boot | **1**, each time | **535** |
   | reception driven by | a 5 s timer kick | the interrupt path (`kick=0`, 31 of 31) |

   **That is the closing evidence for this question, and it is of a different kind from
   everything above.** (a), (b) and (c) are readings of what the device did on its own.
   Attempt 16 *changed the register and the device changed with it*, in the direction "1 =
   masked" predicts and the opposite reading forbids. A polarity claim that survives an
   intervention is closed in the way an observational one is not.

   **The delivery rule, as a fact about this part.** Unmasking a source whose status is
   **clear** re-delivers on the next assertion — 535 times in this boot. The
   acknowledge-then-unmask ordering that was feared to be mandatory is **not needed**, because
   §7.4 has the ISR acknowledge on entry, so nothing is standing by the time the poll path
   unmasks. See §3.7 for the measurement. Whether unmasking a source that *is* still asserted
   re-delivers remains unobserved and is now uninteresting: no correct driver reaches that
   state, because the ISR clears first.

2. **Reg `0x9c` bit6** is pulsed in response to interrupt status `0x500000`; the full error
   taxonomy of `0x50` is not yet enumerated.
3. ~~**TX `word0` field boundaries** are now decoded at instruction level (§5.4) rather than
   inferred from RX, but unlike RX they have not yet been confirmed against silicon. A boot
   that transmits is the confirmation.~~ **ANSWERED — attempt 30, build #46, banner `#49`,
   2026-10-04. That boot happened.** The device accepted, consumed and returned **71**
   descriptors written to §5.4's layout (`w0=30000149`: `[15:0]` length, bit28 LAST, bit29
   FIRST, OWN clear, all offload bits and WRAP clear), handing ownership back on each
   (`w0=80000000`), with `txcur / 16 = 71` matching `tx = 71`. §5.4 has the block.
   **Scope: the layout is confirmed as accepted and consumed, not as correct on the wire** —
   nothing reads a transmitted frame back, so the length field is confirmed to sit where the
   device expects a length rather than to carry the length the peer received. Every offload
   bit, `word3`/TSO and `word2`'s VLAN encoding remain unexercised — still true at attempt 32,
   which reached 847.6 Mbit/s with every offload bit clear. **TX WRAP at index 255 no longer
   does:** attempt 31 reached it twelve times and the engine hung on it every time, and attempt
   32 traverses WRAP cleanly at index 254. See question 8 and §5.4.
4. **PHY quiesce on shutdown — ANSWERED 2026-10-04, attempt 30, and it is the question that
   unblocked the project.** The shutdown path performs a Clause 45 write of `0x0010` to
   register `0x0147` devad `0x1e`, then delays 1000 us.

   > ### ⚑ BIT4 OF `0x0147.1e` INHIBITS THE TRANSMIT PATH. MEASURED, WITH A CONTROL.
   >
   > ```
   > [    2.615915] cl45[residue]: 0147.1e=0010 shutdown-bit4=1 | ...        (pre-touch, before any write)
   > [    2.876262] residue: 0x0147.1e 0010 -> 0000 bit4 1 -> 0 cleared=1 ret=0
   > ```
   >
   > In the same boot: `reg04 = 0x00000b19` (bit0 set) on all 114 samples to t=362 s,
   > `bmsr = 0x796d` (bit2 set on both reads) on all 113, `speed=2 fdx=1` throughout, and
   > **`tx=71/6014`** with `prod==cons free=256 drop=0 err=0 stall=0` on all 66 `txwatch`
   > samples. Twenty-nine previous boots read `tx=0/0`, `reg04=0x00000b18` and `bmsr=0x7969`.
   >
   > **The control, and it is what makes this a measurement rather than a correlation.**
   > Attempt 29 (build #45) differs from attempt 30 (build #46) by this one read-modify-write.
   > It reads the same `0x0147.1e = 0x0010` on arrival, performs the same `0x0403.1f` PLL4
   > bypass write in the same 15 ms window, and reads `r04 = 0x00000b18` on **all 31** `stage`
   > samples with `tx=0/0` on all 22 `txwatch` samples. **The PLL4 write is excluded as the
   > cause by direct observation.** The only PHY write that differs between a boot that
   > transmits with the link up and a boot that does neither is the bit4 clear.
   >
   > **It is also not this register's reset default.** That was the pre-registered risk when
   > build #46 was specified: a reset default that nothing depends on clears to no effect. It
   > cleared and the part started working.

   **What the answer is, split by status.**

   **FACT.** The ROM's shutdown routine writes `0x0010` to `0x0147.1e` after both DMA engines
   have stopped. The register reads `0x0010` before this driver touches it, on both boots that
   have looked. Clearing bit4 makes this part link and transmit; not clearing it leaves both
   dead. All three halves are measured.

   **INFERENCE, labelled as such.** Orbis's shutdown is followed by a real power cycle, which
   resets the PHY and clears the bit, so the firmware never meets its own residue. We arrive by
   **kexec**: no power cycle, the PHY's register space intact — attempt 23 established exactly
   that — so the bit persisted into every boot this project ever took. Nothing measures that
   chain; it is the reading that makes the fact actionable.

   > ⚠ **UNKNOWN, and to stay recorded as unknown: what bit4 does mechanically.** Not a
   > power-down, not an isolate, not a lane quiesce, not a loopback — the register is unnamed in
   > every MediaTek source searched (§1.1's open list) and nothing in this project has looked
   > inside the behaviour. "Bit4 inhibits transmit on this part" is the whole of what is known.
   > A port must clear it; a document must not explain it.

   **The original framing of this question was wrong in a way worth keeping.** It read "Not
   needed to bring a port up; likely needed to leave the PHY in a state Orbis can resume from."
   The first clause is exactly backwards: **undoing it is the one thing a port on this hardware
   cannot skip.** The second clause is probably still right and is untested.

   **The routine is `0xde1653f0`, and it is a complete sequence — located 2026-10-04.** It sits
   *below* `mtsc_probe` and so outside the `0xde165530`–`0xde16c000` window every earlier ROM
   sweep covered, which is why it took until now to find; its **only caller in the whole image**
   is the southbridge power-down walk at `0xddff88de`, immediately before that walk resets the
   GBE's block (question 7). It performs **§7.3 steps 1–3 exactly** — `0x54 = 0x7ffffa`, then
   `0x34 = 2` polled on bit1, then `0x38 = 2` polled on bit1 — then clears
   `IFF_DRV_RUNNING | IFF_DRV_OACTIVE`, **then** issues the PHY write above and the `DELAY(1000)`.
   So the PHY write happens **after both DMA engines have stopped**, and the `0x54` write is
   line (d) of question 1's polarity evidence. The packing `0x0147001e` is decoded from
   `smi_cl45_write`'s own body (`0xde167ab0`–`0xde167bd0`), not guessed: register in bits 31:16,
   devad in `(esi & 0x1f) << 8`.

   **This is why build #45 exists, and the reasoning is the part to keep.** Attempt 23 closed
   "our PHY programming is wrong" and attempt 25 closed the MAC equivalent, both by comparing
   against Orbis's **running** values. **The part this driver inherits across the kexec is
   post-shutdown, not post-init** — so a bit this teardown sets on its way out is invisible to
   that entire class of comparison. It is not a value programmed wrongly; it is a value nobody
   programs at all. ~~`0x0147.1e` has **never been read on any boot**; build #45 reads it at the
   pre-touch point and is built, archived and awaiting a boot.~~ The criterion was set in
   advance: **`0x0010` read back there would make it a candidate; `0x0000` would empty the
   theory.** See §1.1's open list and `build-history/BUILDS.md` #45.

   > ### ⚑ MEASURED 2026-10-04 — attempt 29, build #45, banner `#48`: `0x0147.1e` READS `0x0010`
   >
   > ```
   > [    2.626014] cl45[residue]: 0147.1e=0010 shutdown-bit4=1 | 0037.1e=0033 0039.1e=000f 0096.1e=8000 0107.1f=4000 0171.1e=0065 mdio=0
   > ```
   >
   > **The pre-registered positive criterion is met.** `0x0147.1e` = `0x0010`, bit4 set, at the
   > pre-touch point, before this driver has written anything to the PHY — the first reading
   > of this register on any boot, and **exactly** the value the shutdown routine above writes
   > there.

   **The mechanism, in three parts, because only the first is established.**

   **FACT.** The ROM's shutdown routine writes `0x0010` to `0x0147.1e` after both DMA engines
   have stopped. The register reads `0x0010` before we touch it. Both halves are measured —
   the first from the ROM, the second from attempt 29. Nothing in the ROM's init path and
   nothing in this driver writes that register on any path.

   **INFERENCE — labelled as such, and not established.** Orbis's shutdown is followed by a
   real power cycle, which resets the PHY and clears the bit, so the firmware never meets its
   own residue. We arrive by **kexec**: there is no power cycle, the PHY's register space
   survives intact — attempt 23 established exactly that, finding the whole vendor
   configuration already correct at pre-touch — and the bit persists into our boot. Nothing
   measures that chain. It is the reading that makes the fact interesting, not part of the
   fact.

   > ~~⚠ **CAVEAT, and it belongs with the finding rather than under it: `0x0010` could be this
   > register's reset default.** If it is, the shutdown write stores a value that was already
   > there, the register has never held anything else on this part, and **the whole lead is a
   > clean negative.** Nothing in this project separates the two readings — the register is
   > unnamed in every MediaTek source searched (§1.1), it has been read exactly once ever, and
   > **what bit4 does is not known**: not a power-down, not an isolate, not a lane quiesce, not
   > anything. `0x0010` is consistent with residue and equally consistent with a part that
   > always reads `0x0010` there.~~
   >
   > **SEPARATED at attempt 30 — see the banner at the head of this question.** The caveat was
   > correct to raise and correct to put above the finding rather than below it; the write that
   > resolved it was specified in the same breath. The half that survives is the last clause:
   > **what bit4 does is still not known.**

   **Why no earlier pass could have caught it, and this is the part worth keeping.** Attempts
   23 and 25 both compared against Orbis's *running* state. **A bit set by a teardown is
   invisible to that entire class of comparison**: it is not a value we program wrongly (the
   comparison would have shown that), not a value we fail to program (likewise), but **a value
   nobody programs at all**, in a register neither init path mentions. No further comparison
   against a running Orbis could have surfaced it, however long it ran or however many
   registers it covered. **That is a genuinely new hypothesis shape and it is the only
   candidate of it** — class 1 is empty (attempt 27) and class 3 is spent (question 7); this
   is neither, because it is inherited state that is *correct* by every comparison this
   project knows how to make.

   **The successor experiment was build #46**, which clears bit4 with a read-modify-write,
   default on, and prints `residue=0` as the in-image control. **The measurement could not
   separate residue from reset default and a write could**: if bit4 is residue and bit4 matters,
   clearing it moves something downstream; if `0x0010` is the reset default, clearing it moves
   nothing and the lead closes negative. Same cheap-discriminating shape as build #31's
   four-point `0x34` sample and build #43's `0x0403` readback — **both of which came back
   negative**, which was the honest prior.

   **It came back positive.** Attempt 30 is in `notes/hardware-bringup-log.md`, and the result
   is at the head of this question. Recorded with the prior still visible: this was the third
   cheap discriminating experiment of the same shape in five builds, and the first two were
   negatives. **The method is what produced the answer, not the hit rate** — two negatives and
   a positive out of three one-bit experiments is a working search, and all three cost one boot
   each.

   **The transferable lesson, and it is the one to keep.** Attempts 23 and 25 eliminated whole
   classes by comparing this driver's values against Orbis's — and both compared against Orbis
   **running**. A bit set by a *teardown* is invisible to that entire class of comparison: not a
   value programmed wrongly, not a value omitted, but **a value nobody programs at all**, in a
   register neither init path mentions. Every negative those passes reached was correct; the
   hypothesis space they searched did not contain this. **So when a class-eliminating comparison
   comes back clean, ask what *state* its reference was in — not only which registers it
   covered.** This project inherited a device that had been shut down and compared it against
   one that was running.
5. **Suspend / wake.** The suspend path brings the interface up, waits for `0x04` bit0, then
   transmits a fixed 34-byte frame built from the *secondary* MAC with a rolling sequence
   counter, before clearing `0x54` bit12 — consistent with arming remote wake for Rest Mode.
   Now that `0x54` is known to be a mask (question 1), *clearing* bit12 **unmasks** whatever
   bit12 is, which fits that reading better than it did before: the suspend path arms a source
   rather than switching one off.
   Not required for a basic port.
6. **CLOSED 2026-09-27 — there is no MAC transmitter enable in this device's programming
   model, and the one remaining TX divergence is ours.**

   A pass over the ROM went looking for the register write that makes the transmitter go, on
   the standing assumption that TX failed because we had missed one. **It does not exist.**
   Enumerated at instruction level, not searched for:

   | Hypothesis | Finding |
   |---|---|
   | a write was dropped from §7.2 | `mts_init_locked` makes **exactly seven** register writes, and they are §7.2 steps 9–15. There is no eighth. |
   | the transmit path pokes something | `mts_tx` makes **zero** register accesses. Its only device-visible act is the descriptor store; the doorbell is the caller's. |
   | completion has a precondition | `mts_txeof` makes **zero** register accesses and waits on nothing (§7.7). |
   | `0x34`/`0x38` are set up earlier | `mts_init` never touches either register. |
   | a MAC-level TX enable exists somewhere | **None anywhere.** The TX engine's entire control surface is `0x34` bits 0/1/2 plus the `0x44`/`0x3c` base/current pair. |

   **So the failure is not a missing ROM step, and that is a genuine negative result** — the
   enumeration is of every store in each function, not a search for a particular offset, so
   this does not have the shape of the inferences-from-silence this project has been burned by
   (§8 question 1, "why this time is different").

   **What survived: we reset the TX engine and the ROM never does.** `0x34 = 2` appears in
   exactly one ROM function — `mts_stop`, a teardown. `mts_init_locked` does not reset before
   RUN, and the ROM's own post-error TX restart (§7.4, bits 20|22) rebuilds the ring and
   re-asserts RUN with **no reset at all**. Our start path opens with a **bare full-word store**
   of `DMA_RESET`, which clears every other bit in the register as a side effect.

   **This is a candidate, not a finding, and it is untested.** `0x34` has never been sampled
   before RUN on any boot — sixteen boots sampled `0x38` and none sampled `0x34` — so nothing
   this project holds either confirms or refutes it. The reset is also **not** removable
   blind: it is there because without it the engine kept walking its Orbis-era ring after
   kexec, DMA'ing to `0x0`/`0x10`/`0x20`/… and faulting every access. Removing it on a
   hypothesis resurrects a solved bug.

   Build #31 measures instead, sampling `0x34` and `0x38` at four points — at probe, after
   reset, after the base writes, after RUN — with **nothing branching on the samples**:

   | Reading | Conclusion |
   |---|---|
   | `probe` non-zero, `after-reset` `0x00000000` | the bare store destroyed configuration the ROM leaves standing; stop resetting TX here |
   | `probe` **and** `after-reset` both `0x00000000` | there was nothing to destroy — **the candidate is dead and must not be pursued further** |
   | `after-base` ≠ `after-reset` | writing `0x44`/`0x3c` has a side effect on `0x34` |
   | `after-run` without bit0 | RUN does not latch — a finding of its own, and consistent with §3.4 |

   Note what the third and fourth rows cost: nothing. A four-point sample of two registers is
   the cheapest possible experiment and it discriminates four outcomes, two of which nobody
   predicted. #31 is deployed and unbooted as of this writing.

   **CLOSED NEGATIVE 2026-10-03 — attempt 26 (build #42, banner `#45`) filled in every row of
   that table, and the candidate is dead.** This is the second row, the one written in advance to
   mean *"there was nothing to destroy — the candidate is dead and must not be pursued further"*:

   | Predicted reading | Actual | Verdict |
   |---|---|---|
   | `probe` non-zero, `after-reset` zero → "stop resetting TX here" | `pre-rst` = `0x00000000` | **not this row** |
   | `probe` **and** `after-reset` both zero → nothing to destroy, **candidate dead** | both `0x00000000` | **THIS ROW** |
   | `after-base` ≠ `after-reset` → base writes have a side effect on `0x34` | both `0x00000000`, equal | **no side effect** |
   | `after-run` without bit0 → RUN does not latch | `0x00000001` | RUN **does** latch — see §3.4 |

   > **The fourth row fired too, one boot later, and it was a different boot's answer.** Attempt
   > 30 reads `after-run = 0x00000000` — **RUN does not latch** — in the boot that transmits.
   > Both readings are correct for the state they were taken in: the latch is a property of the
   > *inhibited* part, not of the register. §3.4 carries both. The verdict on this question is
   > unaffected: rows 1–3 read the same in attempt 30 as in attempt 26, so the reset still
   > destroys nothing and still must not be removed.

   So the bare `DMA_RESET` full-word store **destroys nothing**, because `0x34` already reads zero
   when `gbe_start_engines()` is entered — attempt 25 supplied that value at probe, attempt 26
   supplies it at the function's own entry, which is the point the candidate was about.

   > **The TX reset must not be removed on this hypothesis.** The reset is still load-bearing: it
   > exists because without it the engine kept walking its Orbis-era ring after kexec and faulted
   > every access. It now has **no case against it at all**, so removing it would resurrect a
   > solved bug in exchange for nothing.

   The fourth row also inverts what §3.4 said it would: `after-run` came back **with** bit0, not
   without it, which is the `0x34` asymmetry §3.4 now records and the corrections log carries.

7. **ANSWERED FOR THE GBE, STILL OPEN FOR THE FOUR INDICES — 2026-10-04 — which blocks are
   reset-array indices 0, 15, 17 and 19, and is any of them the GBE?**

   > **The GBE's reset-array entry is FOUND and it is RELEASED. Low bank `0x20` (index 8),
   > high bank `0x74` (index 29), Sonics agent `0x2000` — all three read `0` in attempt 28's
   > dump. None of the four held indices is the GBE's, and the reset/quiesce surface is closed
   > by measurement.** Which blocks indices 0, 15, 17 and 19 *are* remains **could not
   > determine**, and stays open — but they are no longer a candidate explanation for TX.

   The identification pass that answers this is the ROM read of 2026-10-04; the detailed
   working record, with the fact/inference separation intact, is
   `reset-array-draft.md` (not published), and the merged facts are
   in [`mts-init-sequence.md`](mts-init-sequence.md). **The original form of
   the question, and attempt 28's own account of what it could and could not say, are left
   standing unedited below the answer**, because the measurement was recorded before the
   interpretation and the value of having done it in that order survives the result.

   **How the index was found, and why every earlier sweep missed it.** The nine-block reset
   walk at `0xddff8760` splices **exactly one** device-specific callback into nine otherwise
   identical block iterations, at `0xddff88de`. That callback is a **GBE BAR0 quiesce routine
   at `0xde1653f0`** — a function *below* `mtsc_probe` (`0xde165530`) and therefore outside the
   `0xde165530`–`0xde16c000` window every prior pass swept, with exactly **one caller in the
   whole image**. It is the GBE beyond argument: it takes `softc+0x3068` as its `struct
   resource *`, the same BAR0 resource slot `mts_init` uses; it calls `smi_cl45_write`
   (`0xde167ab0`); it clears `IFF_DRV_RUNNING | IFF_DRV_OACTIVE`; and it sits immediately
   before the walk's `reset_helper(0x2000)` and the `0x20`/`0x74` asserts at `0xddff88f7` /
   `0xddff8906`. A quiesce is exactly what has to precede a block reset, and the GBE is the one
   block in the walk whose own driver never asserts its own handle — which is why the ROM has
   to stop it from inside the walk.

   **Two supporting arguments are inferred, and the conclusion does not rest on them.** Only
   blocks 1–5 of the walk have a high-bank handle; blocks 1–4 are the four decoded PHY devices
   (USB ×2, SATA ×2), making the two-handle set exactly the five blocks that have a PHY and the
   low/high split **controller reset / PHY reset**. Block 5 is the fifth. That is positional,
   not a name in the ROM, and it is labelled inferred. **It does not matter to the verdict: all
   four fallback candidates — `0x34`/`0x3c00`, `0x14`/`0x0c00`, `0x18`/`0x1000`,
   `0x1c`/`0x1400` — also read `0`.** Every candidate for the GBE is released.

   **The apparent discrepancy in this question dissolves, and the premise was the wrong half.**
   The walk's own indices are **5–13 and 25–29**; the intersection with {0, 15, 17, 19} is
   **empty**. And the walk cannot have executed on any boot: the chain is `0xde72acc0` →
   `0xde3bf4b0` → `0xde23b2c0` → `0xddff8760`, where `0xde72acc0` prints `"mDBG: Power
   down...\n"`, waits one second, calls the southbridge power-down and ends in `hlt ; jmp .`,
   with its own caller following the call with `ud2`. All four functions occur **zero** times as
   a little-endian 64-bit pointer anywhere in the image, so there is no method-table or
   event-handler back door — the chain is rel32 only and it terminates in a halt. So nothing
   released five blocks afterwards; there was never anything to release. (The related hazard —
   that *"nine blocks are held in reset at exit"* is a fact about the power-**down** path and
   reads misleadingly as machine state — is already filed in the corrections log.)

   **The ROM never touches the four held indices, and that is a complete clean negative.** All
   **26** `+0x180000` access sites in the image were recovered and back-disassembled — **22
   writes, 4 reads**: 4 SATA (`0x2c + 4u`, `0x6c + 4u`), 8 USB (`0x24 + 4u`, `0x64 + 4u`, with
   `u ∈ {0,1}` materialised by a `setne` in both drivers), and 14 belonging to the walk. **No
   instruction anywhere in the ROM references offsets `0x00`, `0x3c`, `0x44` or `0x4c`.** The
   raw-`struct resource *` hole was checked rather than assumed: `0xde1d9bf0`'s three callers
   use `0x109000`, `0x108000` and the UART, none of them in range.

   **So indices 0, 15, 17 and 19 are COULD NOT DETERMINE, and that is the honest answer.** Two
   readings cannot be separated by anything this project holds. Either they are unreleased
   resets for blocks this firmware never power-manages, held since cold boot because nothing
   ever released them; **or they are not reset bits at all** — the `bit0 = reset` semantic is
   decoded only from the four handles that two PHY drivers bracket their own init with, and a
   dword at `+0x00` reading `1` is equally consistent with a version, capability or strap
   register. Under **both** readings they are not the GBE's.

   **And they cannot be named by analogy with the other southbridge — checked, and the
   near-miss is recorded so it never becomes a corrections-log entry.** Aeolia/Belize's
   `+0x180000` is a **bit-per-block read-modify-write** with dwords shared between blocks and
   per-block masks; Baikal's is one dword per block written with a literal `1` or `0` and never
   an RMW. **Different structure**, so `0x3c` and `0x4c` appearing in both chips' offset lists
   is a coincidence between two register files and is **not** evidence about the Baikal entries.
   Detail in `notes/mts-init-sequence.md`.

   **Would a block held in reset still answer BAR0 and DMA? — settled by argument, and it kills
   the lead independently.** No ROM code reads a block's BAR0 while that block is under reset,
   so firmware gives no direct evidence either way. But BAR0 register reads, descriptor DMA
   reads and frame DMA writes all traverse the Sonics fabric to the block itself, and a block in
   reset cannot serve them — only PCI configuration space on `00:14.x` is answered by the glue
   and the root complex. **The NIC's own observed behaviour is therefore itself proof that its
   block is released**, independently of and consistently with `0x20`/`0x74` reading `0`. The one
   shape that could have rescued the lead — two handles, with the kexec leaving one of the pair
   asserted so that RX works and TX does not — is dead: both read `0`, as do all seven other
   candidate handles.

   **What this does to class 3.** The reset and quiesce surface that class 3 had narrowed to is
   **closed by measurement** on both halves: no agent is fenced (attempt 28, result 3) and the
   GBE's block is released (this answer). **Class 3 is spent.**

   **And as of attempt 29 both halves have their control.** The reset-array half always
   self-calibrated — USB's and SATA's eight decoded handles read released on a console using
   its disk — and it **reproduced bit-identically on three boots of #44**. The agent half did
   not, because #44's stride missed both SATA and both USB agents; **#45 reads all nine decoded
   bases and the four device agents read `quiesce=0 reset=0 busy=0` exactly as the GBE's
   `+142000` does**, with `ctrl` uniform `0x06040400` across the array. So the negative no
   longer rests on the ROM's `bit4`/`bit0` decode alone. See §8 C's agent-calibration
   subsection and attempt 29.

   What survives is **class 2** — a sequencing or timing requirement with no register trace —
   **plus** the shutdown-residue question build #45 was built to test, which is a new and
   different shape: not a value this driver programs wrongly, but a value nobody programs at
   all. **That question now has a positive reading against it**: `0x0147.1e` = `0x0010` at
   pre-touch, attempt 29 — with the reset-default caveat that could still make it a negative
   (question 4 below, and `0x0147.1e` in §1.1's open list).

   ---

   *Attempt 28's original record, left standing:*

   **This is the question attempt 28 opens, and it is deliberately not answered here.** The
   all-zero condition this section set in advance — *"if every reset entry reads `bit0 = 0` and
   every agent reads `bit0 = bit4 = 0`, the whole clock/reset branch of class 3 is a clean final
   negative"* — is **met on the agents and not met on the reset array**. Four entries are held, so
   the branch stays open and the measurement stands on its own until the blocks are identified.

   **What is established.**

   1. Four entries are held, at indices 0, 15, 17, 19. The values are exact, the reads are good,
      and the array decode is calibrated against four working blocks.
   2. **None of the four is a decoded block.** The nine low-bank handles `0x14`–`0x34` are indices
      5–13 and the high bank `0x64`–`0x74` is indices 25–29; indices 0, 15, 17 and 19 lie outside
      both. **The array is live in regions the ROM sweep never reached, in both directions** —
      index 0 is below the decoded low bank's first handle.
   3. **All five GBE *candidate* entries read released.** `notes/mts-init-sequence.md` named
      `0x14`/`0x18`/`0x1c`/`0x20`/`0x34` — indices 5, 6, 7, 8, 13 — as the unidentified blocks the
      GBE could be. Every one reads `0x00000000`. **If the GBE is one of the nine blocks the ICC
      power walk knows about, its block is not held in reset.**
   4. No agent is fenced, within the scope the instrument defect above sets.

   **What is not established — and the objection carries real weight.** Our GBE answers PCI
   configuration space, answers BAR0 reads **and** writes, passes RX traffic interrupt-driven, and
   performs DMA in **both** directions: it writes received frames into host memory and it reads RX
   descriptors out of host memory to find out where to put them. **A block held in reset should be
   far more broken than that.** So the most likely reading is that **the GBE is not one of the four
   held blocks**, and that four held entries are four genuinely unused blocks on a Slim, held by
   design on a part whose southbridge carries functions this board does not populate.

   > **Nothing in attempt 28 shows the GBE held in reset, and nothing in this document says it
   > does.** Four held entries and an unidentified GBE handle are two separate facts, and joining
   > them is not something this measurement licenses.

   **What would settle it.** Identifying the blocks behind indices 0, 15, 17 and 19 from the ROM —
   the read that was in flight when attempt 28 was recorded. If one of them is the GBE, this
   becomes a finding on the critical path. If all four are unpopulated blocks, the clock/reset
   branch of class 3 closes negative after all. The number does not change either way, which is
   why the measurement is recorded separately from its interpretation.

   **That read landed, and it split the question.** It did not identify the four — they are
   untouched by the ROM and cannot be identified from it — but it found the GBE's own index by
   another route entirely and it reads released. So the clock/reset branch closes negative
   without the four ever being named. See the answer above.

   **Still no write accessor, and still a safety decision rather than an omission.** The reset array
   is shared with SATA and USB; asserting the wrong entry would reset a block the running kernel
   depends on. **Nothing in this document authorises a write to `+0x180000`**, and attempt 28's
   result does not change that — four held entries whose identity is unknown is the strongest
   reason yet not to write one. The 2026-10-04 read strengthens this rather than relaxing it:
   every handle this device could own already reads released, so there is nothing here for a
   write to achieve and a block SATA or USB depends on for it to hit by mistake.

8. **ANSWERED AS A WORKAROUND, OPEN AS A MECHANISM — 2026-10-09 — why does the TX engine hang
   on the last descriptor of the 4 KB ring page, and nowhere else?**

   > **The fault is POSITIONAL and it is worked around by never using index 255.** Walk 255 of
   > the 256 allocated descriptors with WRAP on index **254** and the hang does not occur:
   > **zero stalls in 268 s and 4,589 clean ring laps at attempt 32**, against twelve stalls in
   > 321 s at attempt 31 with the same image at `txring=256`. **What is still open is why** —
   > and it is a question about silicon, not about this driver or this document.

   **The evidence, and it is one-sided in both directions.**

   *That it hangs, from attempt 31 (build #48, banner `#51`):*

   - **Twelve stalls, `tx` = 255, 510, 765, 1020, 1275, 1530, 1785, 2040, 2295, 2550, 2805,
     3060** — eleven consecutive differences of **exactly 255**, zero jitter. The ring is 256
     and one descriptor goes per frame, so the engine fails on the **first** descriptor of
     every lap that carries WRAP and on no other descriptor at all.
   - **`txcur = 0x7ffffff0` and `txctl = 0x00000008` on all twelve**, and those two fields take
     no other value in any `txstuck[]` line. The base is `0x7ffff000`, so that is
     `base + 0xff0 = base + 255 × 16` — index 255, the last descriptor in the page.
   - **The descriptor is correct, field by field.** Twelve distinct `(word0, word1)` pairs, one
     per stall, three samples each: `word0` ∈ {`0x70000042`, `0x7000004d`, `0x7000005e`,
     `0x70000135`, `0x7000050c`, `0x700005ea`} — OWN clear, WRAP set, FIRST and LAST set, every
     offload bit clear — `word2 = 0`, `word3 = 0`, `word1` a valid pool address.
     `notes/tx-wrap-draft.md` derived `0x70000042` from the ROM independently as what `mts_tx`
     emits for a 66-byte frame at `tx_prod == 0xff`. **§5.4 is not wrong and the driver is not
     wrong.**
   - **It is not length-dependent.** 66 bytes and 1,514 bytes hang identically.
   - **The device reports nothing.** `isr = 0x00000000` with the operating mask loaded on all
     36 stalled samples, `drop=0 err=0 carrier=1` throughout. §7.4 has what that does to the
     restart's meaning.
   - **`0x34` bit3 is set and held** on 48 observations while `txcur` and `tx_packets` are both
     frozen — so the engine is persistently *active* and never completing, not idle and
     declining. §3.4.

   *That it stops when index 255 is not used, from attempt 32 (build #49, banner `#52`):*

   - **Zero `txstuck[]`, zero `txrst[]`, and `0x7ffffff0` appears zero times in the whole
     boot.** Highest `txcur` sampled is `0x7fffffd0` (index 253).
   - **`tx = 1,170,378 / 1,519,887,431`** — **4,589 complete laps**, each crossing WRAP at
     index 254.
   - **847.6 Mbit/s** peak from the driver's own byte counter and **711 / 768 Mbit/s** from an
     external browser speed test, against attempt 31's **0.37 / 0.26**.
   - **Zero errors of any kind:** `drop=`, `err=`, `stall=` and `kick=` take the single value
     `0` across the boot, and `status=` takes only `0x04`, `0x80` and `0x000000c0`.

   **The candidate this eliminates.** Outcome (b) — "the engine mishandles bit30 wherever it
   appears" — predicted in advance that a 255-entry ring would park at index **254** with the
   Δ`tx` = 255 cadence intact. It did not. **Bit30 is handled correctly**, and §5.4's decode now
   has three independent confirmations: the device reads it (attempt 30, RX), our word matches
   the ROM's (attempt 31), and the engine traverses it at index 254 (attempt 32).

   **The mechanism is an inference and is labelled as one.** The shape fits a **descriptor
   prefetch that crosses the ring's page boundary**: the engine reads ahead past the descriptor
   it is working on, and at index 255 that read-ahead leaves the 4 KB page. That would account
   for the one thing nothing else does — **the complete absence of any error report** — because
   from the device's point of view nothing failed; it is waiting on a fetch that will not
   complete. **This project cannot test it.** It would need either a ring deliberately placed
   so the page boundary falls elsewhere, or documentation this part does not have. Recorded as
   the standing hypothesis, not as the answer.

   **What is NOT established, so nobody over-reads the workaround:**

   - **Not that index 255 is safe under any other arrangement.** No boot has tested a
     differently-aligned ring, an allocation larger than one page, or a ring that is not one
     4 KB page. The configuration measured to work is: **256 descriptors in one 4 KB page, 255
     of them walked, WRAP on index 254.**
   - **Not anything about RX.** The RX ring is still 256 with WRAP on index 255, it crossed that
     boundary cleanly at attempt 30, and it received 1,027,818 frames at attempt 32 with
     `kick=0` throughout. **Whatever this is, it is specific to the transmit engine.**
   - **Not that the ROM avoids it.** `notes/tx-wrap-draft.md` (d) argues the ROM's `tx_prod`
     laps the ring every 256 frames regardless of how aggressively it reclaims, so the vendor's
     own driver reaches index 255 on any real workload — which is the strongest reason to keep
     looking for a state-dependent explanation rather than closing at "the silicon is broken".
     That argument is offered there as the file's weakest claim and is not resolved here.

---

## 9. Port notes

- **No firmware blob required.** Everything is in this document plus the per-console EFUSE
  trim, read from the southbridge at runtime.
- **The EFUSE window is on southbridge function 4, BAR4** — a different PCI function from the
  GBE. The driver needs a path to it; on the existing Linux fork that means the `bpcie` glue.
- **Allocate rings and both pools with a 32-bit DMA mask.** The ring pointer registers are
  32-bit, and Orbis asks for exactly that (`lowaddr = 0xffffffff` on all five DMA tags).
  Anything narrower — our driver currently clamps to 31 bits, as the other PS4 southbridge
  drivers do — is a *platform* choice, not a device requirement.
- **Give the MDIO path its own lock, separate from the core driver lock.** The vendor driver
  does: `softc+0x3180` ("gbe:phy") guards `phy_read`/`phy_write`, while `softc+0x30b0`
  ("network driver") guards the init and stop paths. This is a Sony addition — FreeBSD's
  `msk` has one mutex for everything — so it is a deliberate choice, not inherited
  boilerplate. It matters because PHY access is slow (a bounded ~10 ms poll per phase, §4)
  and happens from a kernel thread as well as from the interface paths; sharing one lock
  would hold the core lock across those polls. Structural detail in
  [`mts-init-sequence.md`](mts-init-sequence.md).
- **Raise Max Read Request Size to 2048.** `mts_init` reads the PCIe Device Control field
  and, if it is 512, writes 2048. It is the only PCI config write the vendor driver makes.
- **Do not program the `0x204` coalescing timer.** §7.4 describes what the ROM does with it;
  a Linux port should not copy it. The ROM's ISR is an internal loop that services
  everything in one entry, so its arm-on-entry / disarm-on-expiry cycle terminates. In a
  one-shot ISR plus NAPI it does not: every interrupt re-arms the timer, which fires the
  next interrupt. That produced **629,736 interrupts moving zero packets**; never writing
  `0x204` brought it to 1–2 per boot. Nothing in the ROM depends on the timer being armed.
- **`iommu=pt` is not usable on this platform.** ⚠ With the AMD IOMMU in passthrough, PS4
  devices fall back to SWIOTLB and the kernel `BUG`s in `swiotlb_tbl_map_single` during USB
  hub enumeration, ~3.6 s into boot — before the NIC is ever opened. Build #21 panics on
  boot for this reason; it is kept only as a record. The IOMMU must stay **on and
  translating** on this hardware.

Of the numbered questions in §8, most are refinements: a driver can bring a link
up and pass traffic without resolving them. **Two are still open.** Question 2
is the `0x50` error taxonomy, of which only the bit6 pulse and the bits 20|22
restart are decoded. Question 5 is suspend and wake, which this project has
never exercised. Questions 1 and 6 are closed, 4 is answered, and 7 and 8 are
answered in part with the open remainder stated at each. **A and B are
closed too, and this paragraph described them as live for several revisions after
they were not.** B is resolved: the link comes up and the device passes traffic at
line rate once the shutdown residue is cleared, so it no longer "blocks all
traffic". A is resolved in the sense that mattered: the PHY is not silently
mistuned, because the calibration this driver writes was shown to match the values
the part already holds. **Question 7 is answered for this
device and is off the critical path**: the GBE's reset handles are identified and read released,
so no reset or quiesce state gates this port. What is left of it — which blocks hold indices 0,
15, 17 and 19 — is open, is **could not determine** from the ROM, and is **not** on the critical
path, because under both available readings of those entries none of them is the GBE's.
**Question 4 is the one to watch instead**: it is still formally a refinement, but the
shutdown-residue hypothesis built on it is the live successor to class 3. It was
tested and it is **settled**: `0x0147.1e` reads `0x0010` at pre-touch, the value the
ROM's shutdown routine writes, in a register no init path touches; clearing bit4
brings the link up, with `reg04` bit0 setting 39 ms later after twenty-nine boots
of reading down. The two candidate explanations the test was built to separate,
residue versus the register's reset default, are separated: it is residue.

**What remains unknown is the mechanism, not the effect.** Bit4's function has no
name in any public source, and clearing it is a measured cause with an undescribed
middle. That is a smaller gap than it was, and it is the gap.

