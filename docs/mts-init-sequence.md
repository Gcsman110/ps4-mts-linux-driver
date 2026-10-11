# Baikal GBE — analysis record

> **A note on references.** This repository publishes the specification, the
> driver and the corrections log. It does not publish the per-boot serial
> captures, the per-build records, or the working drafts those documents cite
> as evidence, because they are specific to one console and one machine.
> Citations below name them (`the bring-up log`, `attempt 30`, `build #46`)
> and the reasoning stands on its own; the raw logs are not part of this tree.


Bookkeeping for the `mtsc`/`mts` reverse engineering: where things live in the Orbis 9.00
kernel, what the softc looks like, and what we got wrong along the way.

**The hardware description lives in [`../docs/baikal-gbe-register-map.md`](baikal-gbe-register-map.md).**
That is the specification the Linux driver is written from — register map, PHY access,
calibration, descriptor formats, and all six operating sequences. This file deliberately
holds only what that document excludes: kernel addresses, FreeBSD-side structure, and the
correction history. Nothing here should restate it; if the two ever disagree, the doc wins.

---

## Function inventory

```
0xde1653f0                      GBE BAR0 quiesce / shutdown; one caller in the
                                whole image, 0xddff88de in the ICC power walk
mtsc_probe        0xde165530    entry; 0xde1655ef is the vendor compare, mid-function
mtsc_attach       0xde165650    resources, 5 DMA tags, 2 rings, bus_setup_intr
mts_init          0xde166180    reset, PHY calibration, MAC, autoneg   (4434 bytes)
mts_intr          0xde167310    ISR                                    (1777 bytes)
smi_cl45_write    0xde167ab0
smi_cl45_read     0xde167c60
phy_read          0xde167e20    Clause 22
phy_write         0xde167f30    Clause 22
mts_rx_drain      0xde168030                                           (373 bytes)
mts_txeof         0xde1682c0    transmit completion                    (232 bytes)
mts_rxeof         0xde1683b0                                           (649 bytes)
mts_stop          0xde168640                                           (386 bytes)
mts_init_locked   0xde1687d0    ring setup + DMA engine start          (629 bytes)
mts_probe         0xde168a60    the child device
mts_ioctl         0xde168df0                                           (1493 bytes)
SceGbeMtsPhyCtrl  0xde169aa0    PHY/link-repair kthread                (1153 bytes)
mts_rxfilter      0xde169f30                                           (1284 bytes)
mts_tx            0xde16b0c0                                           (940 bytes)
mts_suspend       0xde16b4a0    wake-frame arming                      (957 bytes)
```

Named 2026-09-27, all previously unnamed in this inventory. Two of the six names are
**inferred** and are flagged as such — they are not decoded identifications:

```
if_init           0xde165f60    calls mts_init then mts_init_locked    (400 bytes)
mts_start         0xde1693d0    the real transmit path                 (896 bytes)   INFERRED
0xde1681b0                      link-state handler: reads 0x04 twice   (272 bytes)
                                and publishes via 0xde1d5be0; three
                                call sites, 0x50 bit2 one of them
0xde16aeb0                      loopback self-test                     (528 bytes)   INFERRED
if_stop           0xde1660f0    wrapper                                (144 bytes)
0xde169750                      lock-wrap around mts_init_locked        (96 bytes)
```

**`0xde1653f0` is new as of 2026-10-04 and it is why the inventory now starts below
`mtsc_probe`.** Every earlier sweep of this driver swept `0xde165530`–`0xde16c000` — from
`mtsc_probe` upward — so a GBE function *below* the probe was outside the window by
construction, which is exactly where this one was. It is identified by what it touches, not by
a name: `softc+0x3068` as its `struct resource *` (the same BAR0 slot `mts_init` uses), a call
to `smi_cl45_write` at `0xde167ab0`, and an
`if_drv_flags &= ~(IFF_DRV_RUNNING | IFF_DRV_OACTIVE)` clear of `0x40 | 0x400` at
`[*(softc+0x30a0)] + 0x1c8`. Its register sequence is doc §8 question 4; the BSS softc pointer
it loads is at rip target `0xdfb5a9f8`, past segment 4's `filesz`, and it returns immediately
if that is NULL. **Two functions outside the driver range also belong to this device's story**
and live in the "Outside BAR0" section below rather than here: `0xddff8760`, the nine-block
southbridge reset walk in `icc_power.c` that is `0xde1653f0`'s only caller, and `0xddff8a70`,
the Sonics agent reset helper it calls per block.

`mts_start`'s name comes from recognising the FreeBSD `IFQ_DRV_DEQUEUE` idiom in its loop
shape, not from a string or a symbol — the function is the `if_start` half of the driver and
`mts_tx` is the per-frame worker it calls. Confidence is high but it is pattern-matching, and
the distinction is recorded because this inventory has mislabelled a function before
(`0xde167e20`, in the corrections log below). `0xde16aeb0` is inferred the same way and more
weakly; treat "loopback self-test" as a working label.

The four unnamed entries are left unnamed deliberately. What each one *does* is decoded; what
the vendor calls it is not, and inventing a plausible name is how `smi_cl45_read` ended up on
the wrong address.

The ISR was missed by the initial call-graph walk because `bus_setup_intr` takes it as a
function pointer rather than calling it:

```c
bus_setup_intr(dev, irq_res, 0x204 /* INTR_TYPE_NET | INTR_MPSAFE */,
               NULL, mts_intr, softc, &softc[0x611]);
```

## FreeBSD kernel symbols — identified 2026-10-02 by the type import

Importing the FreeBSD 9 kernel types into the Ghidra project
(method) resolved
six of the surrounding kernel symbols. All six are **stock FreeBSD**, nothing Sony-specific.

| Address | Symbol | Deciding evidence |
|---|---|---|
| `0xde353610` | `device_set_desc(device_t, const char *)` | `+0x60` loaded and null-tested, `+0x74` tested `& 0x8`, `free(desc, M_BUS)`, `+0x74 &= 0xf7`, `+0x60 = arg2`, then the generation `INC`. Reads RDI/RSI only. |
| `0xde3524e0` | `device_get_softc` | — |
| `0xde2c5ce0` | `free(void *, struct malloc_type *)` | null-test, then a 16 KB slab-base mask, then the slab-from-address lookup at `0xde0f13b0`. |
| `0xde2b2eb0` | `_mtx_lock_flags(struct mtx *, int, const char *, int)` | 4 params — see the corrections log; the callee alone gives the wrong answer. |
| `0xdf52ba20` | `M_BUS` | the `malloc_type *` passed by `device_set_desc`'s free. |
| `0xdf52bb98` | `bus_data_generation` | 20 inlined `INC` sites across `0xde351da0`–`0xde3543xx`, one read at `0xde357bca`, and a `CMP` against a caller-supplied generation at `0xde356ee6` — `sysctl_devices()`'s `EINVAL` check. |

`device_set_desc` is the `copy == 0` entry point. FreeBSD routes both setters through
`device_set_desc_internal(dev, desc, copy)`; with `copy` constant-folded to zero the
`malloc`/`strcpy` arm vanishes, which is why the body has a free but no allocation. The
absence is not a discrepancy with the source.

The real symbol is `_mtx_lock_flags` with the leading underscore — `mtx_lock_flags` is the
macro wrapper in `sys/sys/mutex.h`.

### `struct device`, and why `desc` lands at `+0x60`

The structure carried as `astruct` in the disassembly is `struct device`, length **`0xa8`**.
Two fields were mistyped and both now decode:

```
+0x60   char *  desc            (was read as ulong)
+0x74   u_int   flags           (was read as a byte)
                bit3 = DF_DESCMALLOCED = 0x08
```

`KOBJ_FIELDS` expanding to a single `kobj_ops_t` is what puts `desc` at `0x60`; get that
wrong and every offset in the structure shifts.

### One datum that does not reconcile — recorded, not smoothed over

`bus_data_generation` holds **`0x53F6D9C5`** in this dump. FreeBSD's static initialiser is
`static int bus_data_generation = 1;` (`sys/kern/subr_bus.c`), and the variable is only ever
incremented, so a value that large is not reachable by normal operation either.

The behavioural identification does not depend on it — the 20 increment sites and the
`sysctl_devices()` comparison are conclusive on their own. But **anyone treating this dump's
`.data` as a pristine image should know it is not.** No explanation is offered here; the
likeliest candidates are that the dumper caught the page mid-use or that the segment is not
a faithful at-rest image, which is the same suspicion already standing against the resource
specs at `0xdf503d20`/`0xdf503d40` under *Still open*. Two independent oddities in the same
segment is worth noticing.

## Driver extent

Using embedded `if_mts.c` assertion paths as the discriminator, the child driver spans
`0xde168030` to `0xde16bbc0`; `0xde16cbe0` is `link_elf_obj.c` and unrelated.

```
mtsc (controller)   ~0xde165530 - 0xde168000    ~10 KB
mts  (port/MAC)     ~0xde168000 - 0xde16c000    ~16 KB
                                                 ~26 KB total
```

## Driver registration names

Strings immediately after the PHY calibration table at `0xde761c60`:

```
"pci"  "ether"  "kernel"  "mtsc_pci"  "pci/mtsc"
```

Matching the Orbis boot log — the driver is two-layer, mirroring `mskc`/`msk`:

```
mtsc0: <Baikal GBE controller> on pci0
mts0: <Baikal GBE> on mtsc0
```

### The vendor build host — from the `__FILE__` string, 2026-10-02

The `const char *file` argument the four `_mtx_lock_flags` call sites pass points at
`0xde761d27`:

```
W:\Build\J02541110\sys\freebsd\sys\dev\mts\if_mts.c
```

The *source path* — `sys/dev/mts/if_mts.c` — was already known from the assertion paths used
to bound the driver extent. What is new is the rest: a `W:\` drive and backslashes mean the
vendor driver was built on a **Windows host**, under build job **`J02541110`**. Of no use to
the port; recorded because it dates and locates the build, and because build-host identity is
the kind of thing that is cheap to note now and impossible to recover later.

---

## `mtsc_attach` — resources and DMA

Controller attach, `0xde165650`, 1732 bytes. Allocates the rings and **then** calls
`mts_init`, which is why `mts_init` itself contains no ring setup.

```c
softc = device_get_softc(dev);
softc+0x314c = (*(u8 *)0xdfb14be8 != 0xff);        /* config flag */
softc+0x30dc = revision_table_A[rev];              /* {0,1,1} */
softc+0x30e0 = revision_table_B[rev];              /* {0,1,0} */
orbis_get_mac(0, softc+0x30d0);                    /* primary MAC   */
orbis_get_mac(1, softc+0x30d6);                    /* secondary MAC */

mtx_init(softc+0x30b0, "network driver");
mtx_init(softc+0x3158, "gbe:ctrl");
mtx_init(softc+0x3180, "gbe:phy");
mtx_init(softc+0x31a8, "gbe:phy_ctrl");
/* softc+0x30e8 "gbe:rmu" via a different init call */

softc[0x60e] = &resource_spec_regs;                /* 0xdf503d20 */
bus_alloc_resources(dev, &resource_spec_regs, softc+0x60d);
softc[0x610] = &resource_spec_irq;                 /* 0xdf503d40 */
bus_alloc_resources(dev, softc[0x610], softc+0x60f);
```

### DMA tags

FreeBSD `bus_dma_tag_create(parent, align, boundary, lowaddr, highaddr, filt, filtarg,
maxsize, nseg, maxsegsz, flags, lock, lockarg, *tag)`:

| Stored in | parent | align | lowaddr | maxsize | nseg | maxsegsz | Purpose |
|---|---|---|---|---|---|---|---|
| `softc[0]` | bus tag | 1 | `0xffffffff` | `0xffffffff` | 0 | `0xffffffff` | parent tag |
| `softc[1]` | `softc[0]` | `0x4000` | `0xffffffff` | `0x4000` | 1 | `0x4000` | TX descriptor ring |
| `softc[2]` | `softc[0]` | `0x4000` | `0xffffffff` | `0x4000` | 1 | `0x4000` | RX descriptor ring |
| `softc[3]` | `softc[0]` | 1 | `0xffffffff` | `0xa0000` | 1 | `0xa0000` | TX buffer pool, 640 KB |
| `softc[4]` | `softc[0]` | 1 | `0xffffffff` | `0x60000` | 1 | `0x60000` | RX buffer pool, 384 KB |

**`lowaddr = 0xffffffff`** — mts uses a **32-bit** DMA mask. Note this differs from
`xhci-aeolia`, which declares `DMA_BIT_MASK(31)`. Relevant to the 6.15 DMA investigation
(task #6) — but read `kernels.md` first: the 31-bit mask is unchanged across
every branch, so it is not established as the cause, and an MSI-domain fault fits the
symptom equally well. The discriminating log line is recorded there.

### Ring allocation

```c
bus_dmamem_alloc(softc[1], &softc[7], 0xd, &softc[5]);   /* WAITOK|COHERENT|ZERO */
bus_dmamap_load (softc[1], softc[5], softc[7], 0x4000, cb, &busaddr, 0);
softc[8]  = busaddr;

bus_dmamem_alloc(softc[2], &softc[9], 0xd, &softc[6]);
bus_dmamap_load (softc[2], softc[6], softc[9], 0x4000, cb, &busaddr, 0);
softc[10] = busaddr;
```

Callback at `0xde167a90` — the usual one-segment "store the bus address" helper.

---

## Softc map

```
0x0000  bus_dma_tag parent
0x0008  dma tag, TX descriptor ring
0x0010  dma tag, RX descriptor ring
0x0018  dma tag, TX buffer pool (640 KB)
0x0020  dma tag, RX buffer pool (384 KB)
0x0028  dmamap, TX ring
0x0030  dmamap, RX ring
0x0038  TX ring vaddr
0x0040  TX ring bus address
0x0048  RX ring vaddr
0x0050  RX ring bus address
0x0058  TX software ring, 256 x 24 bytes   (ends 0x1858)
0x1858  RX software ring, 256 x 24 bytes   (ends 0x3058)
0x3058  TX producer index
0x305c  TX consumer index
0x3060  TX free descriptor count           (init 0x100)
0x3064  RX index
0x3068  struct resource * (registers)      (+0x08 tag, +0x10 handle)
0x3070  resource spec pointer (regs)
0x3080  resource spec pointer (IRQ)
0x3090  device handle
0x3098  normal interrupt mask = 0x1018
0x3099  flag byte (bit4 cleared on suspend)
0x309c  ARMED flag for the 0x204 coalescing timer (arm/disarm, doc §7.4)
0x30a0  pointer to the mts child struct { ifnet *, device_t, softc * } — settled, see below
0x30a8  child device ("mts")
0x30b0  struct mtx, "network driver" — the driver/core mutex
0x30d0  primary MAC (6 bytes)
0x30d6  secondary MAC (6 bytes)
0x30dc  per-revision flag A
0x30e0  per-revision flag B
0x30e8  mutex "gbe:rmu"
0x3108  wake-frame sequence counter
0x310a  wake-frame reply bytes
0x314c  config flag
0x3158  mutex "gbe:ctrl"
0x3180  struct mtx, "gbe:phy" — the MDIO/PHY mutex
0x31a0  struct thread * of the SceGbeMtsPhyCtrl thread (stored from gs:[0] at entry)
0x31a8  mutex "gbe:phy_ctrl"
0x31c8  u32 PhyCtrl event word, and its own _sleep/wakeup channel (&softc+0x31c8)
0x31c9  the byte holding bits 15:8 of 0x31c8 — byte-OR'd with 1 to set event bit8
0x31cc  u32 quiesce/abort request flag
0x32a0  TX pool vaddr
0x32a8  TX pool bus address
0x32b0  TX pool head offset
0x32b4  TX pool tail offset
0x32c0  RX pool vaddr
0x32c8  RX pool bus address
```

Software ring entry, 24 bytes:

```c
struct mts_swdesc {
        struct mbuf *mbuf;      /* +0x00 */
        u64          unused;    /* +0x08 */
        void        *desc;      /* +0x10  pointer to the hardware descriptor */
};
```

The ifnet's `if_drv_flags` sits at `ifp+0x1c8`; `mts_init_locked` guards on bit6
(`IFF_DRV_RUNNING`) and sets it on exit, `mts_stop` clears it along with `IFF_DRV_OACTIVE`.
Unrelated to BAR0 register `0x1c8`, which is a pattern-match filter length register.

### `softc+0x30b0` and `softc+0x3180` are both `struct mtx` — settled

Each is `LEA`'d and passed as RDI to `_mtx_lock_flags`, and in the callee `[RDI+0x18]` is the
word the `CMPXCHG` operates on. That matches `offsetof(struct mtx, mtx_lock) == 0x18`
exactly — `struct lock_object` is `{const char *, u_int, u_int, struct witness *}` = 24
bytes, then `mtx_lock` — with `MTX_UNOWNED == 0x4` as the compare value. Airtight, and the
two roles separate cleanly by call site:

| Site | Caller | Mutex | `__LINE__` |
|---|---|---|---|
| `0xde165f6f` | `if_init` | `softc+0x30b0` | `0x12df` |
| `0xde1660fc` | `if_stop` wrapper | `softc+0x30b0` | `0x130f` |
| `0xde167e2e` | `phy_read` | `softc+0x3180` | `0x56c` |
| `0xde167f3e` | `phy_write` | `softc+0x3180` | `0x589` |

**A dedicated MDIO mutex is a Sony addition.** FreeBSD's `msk`, the closest structural
relative, has exactly one `mtx_init` call in the whole driver. The programming-model
consequence is recorded in doc §9.

### `softc+0x30a0` — **SETTLED 2026-10-03, and it is not a `struct ifnet *`**

This section read "typed, not discriminated — **flagged, not settled**" until the `mts` child's
attach function was decoded. It is settled now, and the type the import gave it was **wrong**.
The attach at `0xde168ada` builds a three-field struct and stores *that*:

```
0xde168afc  S          = device_get_softc(dev)        (the mts child's own softc)
0xde168b07  parent     = device_get_parent(dev)
0xde168b0f  big_softc  = device_get_softc(parent)
0xde168b14  S[0x08]    = dev                          (device_t, the mts child)
0xde168b18  S[0x10]    = big_softc
0xde168b1c  big_softc[0x30a0] = S
0xde168c42  ifp        = 0xde1d3210(6)                (if_alloc(IFT_ETHER), inferred)
0xde168c4a  S[0x00]    = ifp
0xde168c67  ifp[0x00]  = S                            (ifp->if_softc = S)
0xde168c6a  ifp[0xa8]  = 0x5dc                        (if_data.ifi_mtu = 1500)
```

So `softc+0x30a0` is a pointer to

```c
struct mts_child {
        struct ifnet *ifp;      /* +0x00 */
        device_t      dev;      /* +0x08 */
        void         *sc;       /* +0x10  the big mtsc softc */
};
```

and **`ifp->if_softc` is this struct, not the big softc.**

**The behaviour this section recorded was right; only the type was wrong.** Every existing use
in these notes dereferences `0x30a0` at offset 0 and gets "an ifnet" — and it does get one,
because `S->ifp` is at offset 0. Nothing that was derived from those sites needs revisiting.
What does change is anything that would walk `0x30a0` as an `ifnet` at a *non-zero* offset:
`+0x08` is the child `device_t` and `+0x10` is the big softc, not `if_link`/`if_addrhead`.

**And `ifp[0xa8] = 0x5dc` is what pins the `ifnet` layout.** 1500 is `if_data.ifi_mtu`, which
places `if_data` at `ifnet+0xa0` with the stock FreeBSD 9 field order — the thing the previous
version of this section said was missing ("any site touching a field that only `struct ifnet`
has"). That is what makes the `ifnet+0xa4` = `if_data.ifi_link_state` identification in the
link-state path below safe rather than assumed.

Corrections log has the row.

---

## The link-state path and the `softc+0x31c8` event handshake — decoded 2026-10-03

Two reads, one machine: `SceGbeMtsPhyCtrl` @ `0xde169aa0`
(draft) is the consumer, and the link-state handler @ `0xde1681b0`
(draft) sits in the ISR path that feeds it. Neither half makes
sense alone. **The hardware-facing half of this — the `0x04` decode, the three per-revision
outcomes, and the TX negative — is in the specification, §3.6, §3.6.1 and §7.4. This section
holds only the kernel-side structure.**

### Call sites of `0xde1681b0`

| Site | In | Context |
|---|---|---|
| `0xde167853` | `mts_intr` | the `0x50` bit2 branch (`test r12b, 4` at `0xde167822`) |
| `0xde169220` | `mts_ioctl` region | immediately after `phy_write(sc, 0, 0x800)` — BMCR bit11, PHY power-down |
| `0xde16b600` | `mts_suspend` region | under `softc+0x30b0`, `if_mts.c:4544` |

Arg0 is the big softc, there are no other arguments, and both publishing exits are **tail
jumps** to `0xde1d5be0` (`0xde16825d`, `0xde1682b2`); the revision-1 exit at `0xde1681db`
returns having done nothing. So this is the single "recompute and publish link status" entry
point: a link interrupt, a software PHY power-down, and suspend.

### The publisher, `0xde1d5be0(struct ifnet *ifp, u64 word)`

Decoded body, in order: return immediately if `ifp[0xc0] == word`; store `word` at `ifp[0xc0]`;
then on a transition only — `ifp[0xa4] == 2 && !(word & 1)` → `ifp[0xa4] = 1`, arg 2, or
`ifp[0xa4] != 2 && (word & 1)` → `ifp[0xa4] = 2`, arg 1 — call `0xde2a81a0(ifp, arg, 0, 0)`;
finally tail-call `0xde07e6c0(*(0xdfb15340), ifp + 0x4a0)`.

`ifnet+0xa4` is `if_data.ifi_link_state` on the layout pinned by `ifp[0xa8] = 1500` above, and
1/2 are `LINK_STATE_DOWN`/`LINK_STATE_UP`. **So word bit0 — `0x04` bit0, and nothing else — is
what becomes the kernel's link state**; the rest of the word is carried as an opaque status
cache at `ifnet+0xc0`.

| Address | Identity | Confidence |
|---|---|---|
| `0xde1d5be0` | kernel-side link-status publisher, `(ifp, word)` | body decoded; **vendor name unknown, deliberately not guessed** |
| `0xde2a81a0` | called `(ifp, 1\|2, 0, 0)` on an up/down transition | not decoded |
| `0xde07e6c0` | called `(*0xdfb15340, ifp+0x4a0)`; the shape of `taskqueue_enqueue(tq, &task)` | **inferred** |
| `0xde1d3210` | `if_alloc()`, arg `6` = `IFT_ETHER` | inferred, strongly |
| `0xde417ec0` | `wakeup(ident)` — one arg, `sleepq_lock` `0xde11e0b0`, `sleepq_broadcast(id,0,0,0)` `0xde11efc0`, `sleepq_release` `0xde11e140`, then a conditional scheduler kick | decoded, high confidence |
| `0xde353400` | `device_get_parent` | inferred, strongly |

`0xde417ec0` is **not** `0xde417a10`, which is the `_sleep` side of the same pair.

### Producer inventory for `softc+0x31c8` — complete over the driver range

This is the question `notes/phyctrl-de169aa0-draft.md` §2.4 left open. Every reference in
`0xde165530`–`0xde16c030`:

| Address | In | What it writes |
|---|---|---|
| `0xde165e10` | `mtsc` detach/quiesce | `softc+0x31cc = 1`, then `wakeup(&0x31c8)` at `0xde165e1b` — **no event bit set** |
| `0xde166072` | `if_init` (`0xde165f60`) | **`softc+0x31c8 = 0x10100`**, `wakeup` at `0xde166083` — the bit16 and bit8 producer |
| `0xde1673e3`/`0xde1678a9` | `mts_intr`, `0x50` bit2 | **`softc+0x31c8 \|= (reg04 & 1) ? 0x100 : 0x1`**, `wakeup` at `0xde1678ab` — the bit8 and bit0 producer |
| `0xde169b12`, `0xde169df9`, `0xde169e58` | `SceGbeMtsPhyCtrl` | the thread's own writes (its init to `0x100`, its re-arm, its `ev&1` re-arm) |

**The handshake is closed: link up → event bit8, link down → event bit0.** The ISR's sequence
is exact — `call 0xde1681b0`, re-read `0x04`, `_mtx_lock_flags(&softc+0x31a8, 0, "if_mts.c",
4357)`, the OR, `wakeup`, unlock at `:4365` — so the register is read three times on one
interrupt, and the ISR's own test is the third read.

**Clean negative: nothing in the driver range ever writes `0x20000`.** Event bit17 is the
quiesce handler inside `SceGbeMtsPhyCtrl`, and the PhyCtrl draft found the *handler*; there is
no **producer** in `mtsc` or `mts`. Either something outside the driver sets it, or that handler
is dead code in this ROM. Recorded as a negative rather than left as a gap, because the handler
existing is otherwise an invitation to assume a producer does too.

### `softc+0x30e0` does two unrelated things

The notes record it as the RX-tag-steering flag. It is also, on revision index 1, what
suppresses link-state publication entirely (doc §3.6.1), and it is what decides whether
`SceGbeMtsPhyCtrl` is started at all. Both uses are consistent with one per-revision "this
silicon is wired differently" flag; the behaviours themselves have nothing to do with each
other, and reading one off the other is not safe.

---

## Corrections log

Every one of these was asserted wrongly in this repo first. Kept so the same wrong turn
isn't taken twice.

**Four rows are not corrections**, and each is marked as such. The provenance entry filed
2026-09-28, where a true claim was held out of the documentation until it could be sourced
rather than recalled, then promoted once it was. The build-#39 entry filed 2026-10-02,
where an agent recognised its own draft comment as this project's characteristic error, against
evidence that had landed while it was working, and spent a whole build rather than ship it.
The sweep-window entry filed 2026-10-04, where nothing was asserted wrongly and the defect was
an unlabelled artefact in a method. And the attempt-30 `word2` entry filed 2026-10-04, where a
scribe caught its own draft claiming a silicon confirmation for a value the driver had written
itself — on the boot that finally worked, which is the hardest day to be careful on.
All four are in this log deliberately. A log that only ever records the rule being broken makes
the rule look like a tax; these four are what it looks like when it pays.

| Claim | Correction |
|---|---|
| `mts_init` does ring setup | It does not. `mtsc_attach` allocates the rings before calling it; `mts_init_locked` programs them. |
| Software ring entry has `desc` at `+0x08` | It is at `+0x10`. `mts_tx` writes the mbuf at `softc+0x58 + i*0x18` and the descriptor via `softc+0x68 + i*0x18`. |
| `softc+0x3060` is "free count / pool head advance" | Purely the free **descriptor** count: init `0x100`, `-= n` per transmit, `+= 1` per reclaim. The pool head is `softc+0x32b0`. |
| `0xde167e20` is `smi_cl45_read` | It is `phy_read` (Clause 22). `0xde167c60` is `smi_cl45_read`. The four PHY functions separate by argument shape, matching their four error strings. |
| `mts_init_locked` is `mts_ring_init` | It is the interface-start path, not a ring allocator: it programs the ring bases and starts both DMA engines. |
| Descriptors are 64 bytes, contents unknown | 16 bytes, fully decoded. See the doc, §5. |
| TX completion stops when `word2` high half is **not** `0xffff` | Backwards. Orbis terminates the scan on `word2` above `0xfffeffff`, i.e. it stops when the high half **is** `0xffff` — the idle marker meaning the slot is already reclaimed. Introduced as a transcription error when spec §7.7 was written, then inherited by the driver, where it wedged TX permanently after ~193 frames **with no log line**. Caught by cross-check, 2026-09-23. |
| Reg `0x00` bit15 is **busy, poll until clear**; `0x8000` is written to clear a *stuck* transaction; the Clause 22 register number goes in bits 31:16; the PHY answers at *PHY addresses* `0x1e`/`0x1f` | All four wrong. Bit 15 is **DONE**: write-1-to-clear, and the poll waits for it to be **set**. `0x8000` is written unconditionally at the start of **every** phase as the W1C of that done flag — not error recovery, and not a go bit. The Clause 22 register number is in bits **12:8**; bits 31:16 are data only. There is **no PHY/port address field at all** — the on-die PHY is implicit and `0x1e`/`0x1f` are Clause 45 **devads** (MMD 30/31, vendor-specific 1/2). Spec §4 rewritten with the decoded layout, the per-phase clear/issue/poll procedure and the composed command words. **Why it mattered:** the driver, written from the wrong spec, set bit 15 in the command word believing it was a go bit — which write-1-cleared the done flag, so its poll-until-clear passed instantly and it reported PHY writes as succeeding when no transaction had occurred. Boot 2 appeared to show Clause 45 trim writes 0-2 succeeding and only write 3 failing; **none of them had actually run**. The second documentation error to propagate into the driver, after the TX idle-marker inversion above. Decoded at instruction level, 2026-09-23. |
| Reg `0x38` "starts the engine and immediately suspends itself" — bit0 latches, then drops, leaving `0x8` | Bit0 **never reads back at all**. `0x38` reads `0x0` before RUN and `0x8` after, and bit3 means **engine active**, not suspended. Attempt 13 (2026-09-27) shows three post-RUN reads — the `engines:` line, `isr[2]` and dump 2 — all `0x00000008` and never `0x00000009`, while the ring advanced 25 descriptors. The engine was running for twelve boots; the driver was asking a write-only bit whether it was. Spec §3.4 rewritten. (Citation corrected 2026-09-27: this row first cited dump 1's `0x00000000`, which was sampled **pre-RUN**, and claimed "25 frames"; see the two rows below.) |
| "The RX current pointer advances one descriptor and stops" | A measurement error, repeated across several attempts, not a hardware behaviour. It was read microseconds after `DMA_RUN`. The same boot that shows `cur=0x7fffe010` at engine start shows `cur=0x7fffe190` — **25 descriptors** — later in the same dump sequence (no interval is claimed; see the clock row below). Any negative result about a DMA engine needs a deadline before it is believed; this is the second time that lesson has been recorded. |
| "The console **stopped mid-line** at t=97.911, the log did not grow afterwards, and the descriptor dump is the prime suspect" — recorded in the attempt 13 section and as a **Next:** action, and acted on in build #26 | **There was no console halt, and no hardware event of any kind.** `<the serial capture>` was copied **while PuTTY still held it open and was appending to it**, so the repo's copy ended wherever the copy caught it, mid-line, exactly as a partially-written file does. The re-copied log is 962,639 bytes / 10,554 lines, reaches **t=1266.176806**, contains one `Linux version` banner, and shows **zero panics, zero RCU stalls, zero lockups**; dump 2 completes cleanly with all four descriptors back at `00000600`. Recorded as a **method failure**, because the reusable lesson is not about this device: *a capture copied out from under a process that is still writing it is not a capture*, and "the log ends mid-line" is evidence about the copy, never about the machine. Cost: build #26 shrank a working diagnostic — four descriptors to two, three calls to two — to chase a problem that did not exist. Retracted 2026-09-27. |
| "Userspace retried DHCP and never got it" — offered as an observation from attempt 13's log while dispatching this correction pass | **Never recorded, because it is not in the log.** `dhcp`, `udhcpc` and `dhclient` appear **zero times** case-insensitively in the whole file. It was an inference about what userspace must have been doing, stated in the grammar of an observation — the **same failure as the console halt above, twice in one day**, which is why it gets a row despite never reaching a document. What the log does support: `NetworkManager` 24 times, all `audit:` records of `NetworkManager-dispatcher` starting and stopping from t=107.9 to t=1199.2, so NetworkManager *was* running and cycling; and no `eth0` line from anything but `ps4_gbe` after the banner. **A caution for reading the older entries:** "no DHCP" is phrased as an observation in several earlier attempts too, and it rests on the same absence-of-evidence — a DHCP client's traffic need not reach a console log at all, so the missing string shows nothing was logged, not that nothing was attempted. Caught before publication, 2026-09-27. |
| "25 frames were completed and consumed" (spec §8 B, and the two rows above) | **25 descriptors, not 25 frames.** Both `goto next` paths in `gbe_rx()` — the length check and an skb allocation failure — advance `rx_idx` **without** incrementing `done`, so the observation cannot distinguish 25 delivered frames from 25 completed descriptors that were mostly dropped. Only `rxd[0..3]` have observed lengths. The attempt 13 "honest limit" paragraph had this right from the start and the other two places contradicted it — the error was letting a stronger phrasing survive in the spec than in the note it came from. Build #26's `rxcnt:` counter log exists to close exactly this gap. Corrected 2026-09-27. |
| "Seven milliseconds later" — used in spec §8 B, the attempt 13 section and a corrections row as the interval between the two dumps | **This log cannot measure that interval.** `TSC found unstable` at t=3.64, then `sched_clock: Marking unstable` and `Switched to clocksource refined-jiffies`, so printk timestamps come from a per-CPU tick-clamped clock quantised at 1 ms with `CONFIG_HZ=1000`. Dump 2's four lines land 999 µs apart (97.911414 / 97.912403 / 97.913403 / 97.914402) — that is the clamp reporting itself, not the spacing of the events. **Any sub-10 ms interval in this log is at or below the clock's noise floor** and must not be quoted as a measurement. The *direction* of the pointer movement (`0x010` → `0x190`) survives, because file order and descriptor contents fix it independently of the clock. All such figures struck 2026-09-27. |
| "Dump 1 read `0x00000000`, dump 2 read `0x00000008`, and the engine was receiving throughout" — the bit0 evidence in spec §3.4 and the attempt 13 section | The conclusion is right, the citation was wrong. Dump 1's header was sampled **before** `DMA_RUN`, so it is the pre-RUN value §3.4 predicts anyway and is **not** evidence about bit0 under load — and "receiving throughout" contradicted the ordering claim in the same sentence. The three genuinely post-RUN reads are the `engines:` line (t=97.903428), `isr[2]` (t=97.905416) and dump 2's header (t=97.910482), all `0x00000008`, never `0x00000009`. Also recorded: **dump 1 is not a coherent snapshot.** Its header was read pre-RUN and its four descriptor words post-RUN, because `gbe_up` and `gbe_rx` were running on different CPUs — the log even prints `isr[1]` (t=97.901699) ahead of the `rings:` line (t=97.901112) in file order. A diagnostic that straddles two CPUs is not a point-in-time read of the device. Corrected 2026-09-27. |
| IOMMU faults in attempt 13 span "between t=2.73 and t=2.85" | **t=2.698052 to t=2.851872.** Rounded inward on both ends, so the span was reported narrower than it was. The owner attribution was right: all 15 faults are `00:14.7`, ten carrying the `xhci_aeolia` prefix and five in the bare `device=00:14.7` form, and **none** mentions `14.1`. Corrected 2026-09-27. |
| The RX stall would be explained by the engine setting `OWN` with a **zero length** | Refuted by the first unconditional dump. The four completions read `b0040052`, `b0440057`, `b0440087`, `b0440137` — OWN, FIRST and LAST set, lengths 82/87/135/311. It was a reasonable hypothesis, and worth noting that the diagnostic built to test it had been **gated on the hypothesis being true**, so as first written it could only ever confirm it. The gate is what produced zero output in attempt 12. |
| "`ps4_gbe` still faults in the IOMMU" / the device emits `index × 16` with no ring base | Cured by the **31-bit DMA clamp**. At a 32-bit mask the rings sat at `0xffffe000`/`0xfffff000` and every descriptor access faulted; at `DMA_BIT_MASK(31)` they sit at `0x7fffe000`/`0x7ffff000` and three consecutive boots show **zero `ps4_gbe` faults**. The remaining `0x10xxxxxx` faults are all `xhci_aeolia` at `00:14.7`, Orbis-era IOVAs left by the kexec. Spec §8 B closed. |
| `softc+0x309c` is a "first interrupt since start" latch | It is an **ARMED flag for the `0x204` coalescing timer**, and the ROM cycles it continuously: arm on ISR entry (`0x204 = 0x10001388`, `0x54 = 0x007bfffe`), then **disarm** when that timer's expiry — status bit18 — turns up as the sole source outside the normal mask (`0x204 = 0`, `0x54 = 0x1018`). The test is exact equality, not a mask test. The driver, written from the "first interrupt" reading, armed the timer once and never disarmed it, so the timer free-ran: **629,736 interrupts that moved zero packets**. The storm was not a shared-line artefact, a polarity problem or a level-triggered latch — it was self-inflicted, and it began at the instruction that armed `0x204`. Decoded at instruction level, 2026-09-24; doc §7.4. |
| `0x1018` is "the RX and TX completion sources" | It is not. Completion is bit7 (TX) and bit6 (RX), **neither of which is in `0x1018`** — which is bits 3, 4 and 12. `0x1018` is the set of sources whose only response anywhere in the ROM is "drain RX and return". The label mattered because it made the mask look like the datapath mask, which is how "everything outside it is an error" became the reading. Doc §7.4 relabelled, 2026-09-24. |
| "A DMA engine reset clears register `0x30`" | Wrong, and it was used to justify a change in build #17. The ROM's own down/up cycle resets **both** engines and restarts them with only the ring bases, RUN and `0x54` — it never rewrites `0x30`. So a reset plainly does not require the rewrite. The write has been **kept** (harmless, and `0x30` is genuinely DMA configuration), but the reasoning was invented to fit an observation, and the fault reduction measured in build #17 — 12 faults instead of 21–35, first one 30 s later rather than 164 µs — has some other cause that is still unidentified. The error is the same shape as the EFUSE one: a mechanism assembled backwards from a result. |
| Spec §6: "there is no RX refill path to write" | There is, and it is required. The ROM re-arms every RX descriptor as it consumes it — `word0 = 0x80000600`, `word1 = rx_pool_busaddr + idx*0x600`, WRAP if `idx == 255`, then clear OWN — not just hand it back. The buffer address never changes, but the device consumes the length field, so a descriptor returned without a full re-arm is not usable again and the ring **stalls after one lap**. Doc §6 corrected, 2026-09-24. |
| TX `word0` is the RX layout with a TSO field added | Substantially richer, and two bits were missing outright. Length is **16 bits** on TX (`[15:0]`, segments capped at `0xff80`), not the 11 RX uses; bit18 LSO/TSO, bit23 UDP / bit24 TCP / bit25 IP checksum offload, bit26 VLAN insert with `word2 = 0x81000000 \| (vid & 0xfff)`, **bit28 LAST**, **bit29 FIRST**, bit30 WRAP, bit31 OWN; `word3 = (total_len & 0x1fffff) \| (mss << 21)` under TSO and `0` otherwise. WRAP is set at *transmit* time on whichever descriptor lands at index 255, not at ring init. **Why it mattered:** the driver set neither bit28 nor bit29, so every descriptor claimed to be neither the start nor the end of a frame. That matches the observed "TX 0 packets, 0 errors" exactly — the device declined them and had nothing to report. The third documentation error to propagate into the driver. Doc §5.4 rewritten, 2026-09-24. |
| The `gbe:rmu` mutex name means "Marvell Remote Management Unit", so the silicon is Marvell-derived — and the `mtsc`/`mts` split is "the Yukon-2 family shape, which raises the odds that parts of sky2 are reusable" | Both wrong, and both were inferences from *names* rather than from the device. The register map shares nothing with Yukon-2, and the PHY identifier read from live silicon is **`03a2:9441`** — not Marvell (`0x0141`), not Realtek (`~0x001c`), not Broadcom (`0x0020`/`0x0362`), and `lspci` shows no Realtek device on this console. This also refutes the community claim that PS4 ethernet is Marvell or an RTL8111H/G. The OUI decode of `03a2:9441` is provisional (PHY-ID OUI encoding is bit-reversed); the raw pair is the fact. `notes/gbe-findings.md` corrected, doc §1 records the identifier, 2026-09-26. **Completed 2026-09-28 — see the MediaTek row below: the part is an MT7531-family PHY.** The three negatives here all stand; they were not the whole answer. |
| "Seven register values were predicted and confirmed against live silicon" | **Nine.** The enumeration that accompanied the claim in `notes/kernel-re.md` — `0x08 0x10 0x14 0x18 0x30 0x74 0x7c 0xac 0x1d4` — has always listed nine, and so has the ✓ column of doc §3. A count written once and copied into three documents without being counted. Cosmetic in effect, but it is a factual claim about how much of the map was verified. Corrected 2026-09-26. |
| Doc §9: "the open questions are refinements, not gaps — a driver written from this document can bring the link up and pass traffic" | Not true of questions A and B. B (the device emits descriptor addresses with no ring base added) blocks **all** traffic today, and A (EFUSE field packing) mistunes the PHY silently. The sentence was written when both were believed small; ten boots later, one of them is the entire remaining project. Corrected 2026-09-26. |
| `notes/kernels.md`: "the DMA addresses that fault are far above the 31-bit mask, so something is not honouring it" — recorded as the diagnosis of the 6.15.4 failure | Not established. `DMA_BIT_MASK(31)` is **unchanged on every branch** from 5.4.213 to 6.15.4, so the code requesting it on 6.15.4 is the code that works on 5.4. USB, SATA and SD/MMC also share the `Baikal-MSI` interrupt domain, which *did* change across those branches, so "all three die together" fits an MSI-domain fault at least as well. The original hypothesis is kept, with the alternative and a one-line log discriminator beside it. `notes/kernels.md` amended, 2026-09-26. |
| `notes/hardware-bringup-log.md` attempt 5 left standing as "PENDING … `UNTESTED` until then" | Build #16 had booted and its result was recorded in `build-history/BUILDS.md` the same day; only the bring-up log was never updated. `UNTESTED` is the honest default, but it stops being honest once the log exists. Filled in 2026-09-26. |
| "The build agent should commit its own build record" | It should not. `AGENTS.md` gives documentation and git to the **scribe alone** — "Nobody else commits" — and a build agent was dispatched with instructions to commit and push before that rule was re-read. Caught and corrected mid-flight by messaging the agent; no bad commit was made. Recorded because the rule is not bureaucracy: it exists so the written record has one author and one commit-message style. And because this failure mode is **silent** — a build agent that commits tidily looks fine, nothing breaks, and the first symptom is two roles writing history in different voices. Not a hardware fact; this log covers process errors too, for the same reason it covers the others. 2026-09-27. |
| `logs/uart/latest.log` is the newest capture because a capture was taken | Not automatically. The capture script wrote **only** a timestamped file, so `latest.log` still pointed at **attempt 9** while attempt 13 was the live boot. `AGENTS.md` names `latest.log` as where agents look for the newest capture, so an agent dispatched to analyse "the latest log" would have silently analysed a four-attempt-old boot and reported it as current. Fixed by copying attempt 13's capture across. The copy, not the capture, is what makes the convention true. 2026-09-27. |
| "25 descriptors, not 25 frames" — carried deliberately as the honest limit since attempt 13, and the reason spec §8 B would not say the RX datapath delivered anything | **Resolved in the affirmative, 2026-09-27, attempt 14.** `rxcnt: done=18 pkts=18 bytes=2283 drop=0 lenerr=0` at t=98.338253, cross-checked against the device's own pointer: `cur=0x7fffe120 base=0x7fffe000`, and `0x120 = 288 = 18 × 16`, so the hardware said 18 descriptors were pending and the drain produced 18 frames. `drop=0 lenerr=0` means none of the 18 was lost to the length check or an skb allocation failure. **Completed descriptors do become skbs the stack accepts.** Recorded here as a resolution rather than a correction — the caution was right when it was written and was never overstated; it is the first entry in this log that closes *upward*. Scoped tightly in spec §8 B: 18 frames averaging ~127 bytes in **one** poll on one boot, with no ring lap, no WRAP at index 255, no length near the 1536-byte buffer, and no check of frame *contents* — `drop=0 lenerr=0` means those paths were never entered, which is untested rather than proven. Attempt 13's own figure stays at 25 **descriptors**, because that boot had no counter log. |
| "Third and final interrupt of the boot" — the framing the `isr[1]` line at t=98.245820 was handed over with | It is **`isr[1]`: the first and only interrupt of attempt 14's boot.** Third only if you count attempt 13's two in the same file, which is a property of the capture, not of the boot. The log bound is 8 (`if (gp->isr_logged < 8)`) and one slot was used, so the count is exact rather than clipped. **The corrected reading is strictly worse news and a stronger fact:** "the third interrupt failed to clear" describes an intermittent fault, while "the only interrupt of a 531-second boot failed to clear, on its first attempt" describes a protocol the driver does not understand. Caught while verifying the figures against the log, 2026-09-27. |
| "The rootfs on `sdb2` produced 5,482 `EXT4-fs` error lines and 1,672 `Buffer I/O error`s **in this boot** and is deteriorating across boots" — offered for the attempt 14 section | **Wrong boot, and nearly the opposite of the truth.** Those are near-enough the **whole-file** totals (5,483 / 1,672), and the file holds three segments. Attempt 14's own segment contains **zero `EXT4-fs error` lines** and **ten `Buffer I/O error`s, none against `sdb2`** — all ten are `async page read` failures on `sda19/25/27/29/31`, the console's Orbis partitions, during the t=93.8–94.1 partition scan, which is expected. `sdb2` mounted cleanly at t=94.135431. The storm belongs to **attempt 13's** segment: 5,370 / 1,662. So the media was *healthier* on the newer boot, and "deteriorating across boots" would have been filed as a trend from data pointing the other way. **The reasoning error is the one this log already has a row for and is why that row exists:** a whole-file `grep -c` on an append-only PuTTY log counts every boot in the file. Row below on `latest.log` for how the segments got confused in the first place. Caught before publication, 2026-09-27. |
| Attempt 13's media figures, "4,065 `EXT4-fs` / 1,232 `__ext4_find_entry` / 1,255 `Buffer I/O error`s, continuing to the end of the capture" | **All three reproduce exactly on the capture they were taken from, and all three were floors.** That capture ended at **t=1266** while the machine was still running; attempt 14's capture contains the same boot in full, to **t=1473.869246**, and the real figures are **5,370 / 1,641 / 1,662**. Not a counting mistake — the phrase "to the end of the capture" was doing work it could not do, because the end of a capture is a fact about when copying stopped. **An error count taken from a log of a live machine is a rate sampled once, not a total**, and should be quoted with the timestamp it runs to. Revised in place, 2026-09-27. |
| `logs/uart/latest.log` was refreshed for attempt 13, so the convention holds now | **It broke again on the very next boot, the same way.** Dispatched to read attempt 14 from `latest.log`, a file turned up **byte-identical to `putty-20260927-attempt13-rxdump-uncond.log`** — same 962,639 bytes, one `Linux version` banner reading `#27`, one probe line, no `rxcnt:` line and no `watch[` line anywhere in it, and `after-w1c=0x00000000` on both interrupts: attempt 13's boot, presented as attempt 14's. Nothing in `logs/uart/` contained the newer boot at all; it existed only in the live `<the serial capture>`, and was copied in as `putty-20260927-attempt14-imr-fix.log`. **This is the second occurrence of the row above it, and the first fix did not generalise** — copying attempt 13's capture across made the convention true for one file, not for the mechanism, because `scripts/poll-putty-log.sh` only writes `latest.log` on its normal exit and a boot captured any other way leaves the pointer stale. The failure is silent and the stale file is *plausible*: it names the right device, it has a `#27` banner that reads like a build number, and it is full of `ps4_gbe` output. What caught it was the **absence** of instrumentation the build was known to add. **Check the banner against the build and grep for that boot's new log lines before analysing any capture.** 2026-09-27. |
| Spec §8 question 1: attempt 13 is "strong evidence for 1 = enabled" on `0x54`, needing only a register poll to close | **Weakened, and the register poll did not close it.** The argument was "bit6 clear → the source stopped firing", and its force came from assuming an enabled-and-asserted source *would* fire. Attempt 14 refutes that assumption on this part: `imr=0x007bfffe` (bit6 **enabled**) and `0x50 = 0x00000040` (bit6 **asserted**) on all 24 register samples across 123 seconds, with **no interrupt delivered**. Nothing here argues for 1 = masked — it is all consistent with 1 = enabled plus an edge-triggered MSI and a level that never deasserts — but **enablement and delivery are now confounded**, so a stuck level explains attempt 13's silence just as well as a disabled mask, with no reference to polarity. The question is not closer to closed; it needs a source whose assertion is *transient*, and it is now blocked behind §3.7 rather than behind one more boot. **Third time this question has looked settled and not been** (build #18, attempt 13, and the poll built to settle it), which is the argument for leaving it open. 2026-09-27. **Closed by attempt 15 on the fourth attempt — and in the direction opposite to both earlier closures. See the `0x54` polarity row below.** The instinct to leave it open was right; what was wrong was expecting the answer to come from a better version of the same experiment. |
| `notes/hardware-bringup-log.md`: "three of the **nine** defects above were the document's" | The table above that sentence has listed **ten** rows since attempt 13 and eleven since attempt 14. A count written once and not recounted when rows were added — the same shape as the "seven register values / nine" row above, and cosmetic in the same way, but it is a claim about the ratio of our errors to the specification's. Corrected to eleven, 2026-09-27. |
| Reg `0x54` is an interrupt **enable** — "1 = enabled" — the reading behind every mention of this register in the repo, the `IMR_*` names in the driver, spec §3's register table, §7.2's step-15 hazard note, §7.4's mask discussion, and two earlier attempts to close spec §8 question 1 | **Inverted. `0x54` is a MASK: a set bit MASKS its source.** Closed 2026-09-27 by attempt 15 (build #28, banner `#30`). The boot's single interrupt read `status=0x00000040` — **bit6**, RX completion — and it was delivered while `0x54` held **`0x00001018`**, which is bits **3, 4 and 12**. Bit6 is not in it. Under "1 = enabled" that source was disabled and the interrupt was impossible; it arrived. Corroborated from the other side by the `engines:` line, which reads `0x54` back *after* the `0x1018` write and finds `0x007bfffe` — the value `gbe_isr()` writes on entry — so the ISR had already run inside the `0x1018` window. Two independent confirmations: `IMR_ARMED = 0x007bfffe` sets bit6 and is re-written by both the ISR and the poll path with nothing restoring `0x1018`, which predicts exactly **one RX interrupt per boot** and is what attempts 13, 14 and 15 each recorded; and §7.3 step 1's `0x54 = 0x7ffffa` at interface-down is coherent as "mask almost everything, then reset" and absurd as "enable almost everything while freeing the buffers those sources refer to". **Why the earlier reasoning failed, which is the part worth keeping.** Build #18 and attempt 13 both closed this question in the *other* direction, and both used the same argument shape: *a source did not fire, therefore the mask disabled it.* That is an inference from silence, and silence has other causes — #18's died when the timestamps showed the storm ran at a flat ~1 ms under **both** mask values (and the decode then showed bit18 is the `0x204` timer, which `0x54` does not gate at all), and attempt 13's died when attempt 14 showed a source enabled *and* asserted for 123 s delivering nothing. Each refutation was treated as "that experiment was flawed, run a better one", when the repeated failure of the *same argument shape* was the signal. Attempt 15's evidence is a **positive observation** — an interrupt *delivered* for a bit *absent* from the register — and nothing in it rests on something failing to happen, which is why no amount of missing traffic, no stuck level and no property of MSI delivery can explain it away. **A methodological rule this log has now paid for twice:** a claim about a control register that rests only on a device *not* doing something is not closable by more of the same; it needs an observation in which the device *does* something the wrong reading forbids. Spec §3 table, §3.7, §7.2, §7.3, §7.4 and §8 question 1 rewritten. |
| The driver's constant names — `IMR_NORMAL`, `IMR_ARMED`, `IMR_STOPPED`, and the `GBE_IMR` comment "enable/mask; polarity open" | **The names encoded the inverted reading. ~~Being changed in build #29~~ — DONE in build #29, committed and on the boot media**: `IMR_MASK_RUNNING` / `IMR_MASK_SERVICING` / `IMR_MASK_DOWN`, values unchanged, and the register comment now reads "interrupt MASK: 1 = MASKED". Recorded here rather than left to the diff, because this is the failure mode the corrections log exists for: the *values* were always right — the driver wrote Orbis's values in Orbis's places, deliberately, rather than deriving a mask, which is why the bug never produced a wrong register write — but `ARMED` names the value that **masks** RX completion, and `NORMAL` names the one that leaves it live. A future reader correcting the spec would have hit those names, found them contradicting the corrected document, and had no way to tell which was stale. **Names are documentation with no corrections log of their own**, so an inverted one re-teaches the bug to everyone who reads the source before the spec. The deviation itself stays: keep writing Orbis's values in Orbis's places, and never derive a mask. |
| "`0x50` bit6 will not clear / is not cleared by writing it back, so the source stays asserted and an edge-triggered MSI never re-fires" — recorded as a **defect of the device** in spec §3.7, §7.4, the `0x50` row of the §3 register table, the attempt 14 section, the defect table in `notes/hardware-bringup-log.md`, `README.md` and `driver/README.md`, and stated twice on the way in | **There is no acknowledge defect. Write-1-to-clear clears bit6.** Attempt 15: `isr[1] status=0x00000040 after-w1c=0x00000000`, read with **two completed, undrained** descriptors in the ring (`rxd[0] w0=b0440087`, `rxd[1] w0=b0040056`, both OWN\|FIRST\|LAST), and `ack[1]` read `0x00000000` again from a different function 100 ms later. Attempt 14's `after-w1c=0x00000040` is two separate register accesses with 18 completions pending and the engine still receiving, so the source had every opportunity to re-assert in between — the readback never distinguished "did not clear" from "cleared and re-asserted", and it was read as the first without that being noticed. What kept `0x62` standing across all 70 of attempt 15's watchdog samples is that **nothing ever wrote `0x50` again**: `gbe_isr()` is the driver's only writer and it ran once. The persistence was a **consequence** of the interrupt famine, not its cause. The cause was the mask, row above. **The reasoning error, stated plainly:** a register that is not being written is not a register that refuses to be written, and the difference is invisible if you only ever sample it. The claim was asserted twice — once in the attempt 14 write-up and again when dispatching attempt 15 — and both times "stuck" was doing work that "unacknowledged" would have done without implicating the silicon. **The three-point `ack[]` probe built to chase it answered by refuting both candidates it was built to separate**: `ack[1]` kills "bit6 is a level meaning completions pending" (a level cannot read zero with completions pending), `ack[2]`–`ack[4]` kill "the `DMA_POLL` doorbell is the acknowledge" (`pre=postdrain=postbell=0x00000062` after a 64-descriptor drain), and `isr[1]` kills "the status write is not the acknowledge". **That double negative, not any positive result, is what redirected attention to `0x54`.** Worth keeping on both counts: a probe that empties its own hypothesis space has done its job, and the second candidate it refuted was the repo's own driver behaviour rather than the device's. `0x62` is bits 1, 5 and 6; **bits 1 and 5 stay undecoded** and are not to be guessed (spec §8 question 2). |
| "18 frames were received and delivered in a single poll, and then reception stopped" — spec §8 B's strongest claim since attempt 14, and the resolution of the descriptor-versus-frame question | **Superseded upward, and the descriptor-versus-frame question is now closed at scale.** Attempt 15: `rx_packets` 2 → **436** over **358 s** (t=20.688247 to t=368.674276, 70 watchdog samples), `drop=0 lenerr=0` on both counter dumps. The question this closes has a history in this log — it opened as "25 descriptors, not 25 frames" at attempt 13 (both `goto next` paths advance `rx_idx` without incrementing `done`, so a pointer count cannot distinguish delivered frames from dropped ones), and was answered for 18 frames in one poll at attempt 14. It is now answered for 436 frames across a whole boot, and `done=64 pkts=66` on the second dump is the budget capping `done` rather than any disagreement. **But the scope moved sideways, not just up, and spec §8 B says so.** Every one of the 436 arrived because a 5 s timer kicked NAPI: `rx_packets` advances only in the interval following a `kick=1` sample and is flat in every interval following a `kick=0` one, and the boot took **one** hardware interrupt. So the honest claim is "436 frames received and delivered over 358 s, but only under timer-driven polling" — a sawtooth whose ceiling is the timer period. Still untested: frame *contents* (no capture with a verifiable payload on any boot), a ring lap and WRAP at index 255 (highest `cur` is `0x7fffeb80`, 184 descriptors past base), any length near the 1536-byte buffer (~69 bytes average), and the drop and length-check paths, which `drop=0 lenerr=0` means were never *entered*. And **nothing has been transmitted on any boot, ever** — fifteen boots, `txctl=0x00000001` after RUN, no transmit attempted. |
| "~370 seconds" for attempt 15's receive span, and "`drop=0 lenerr=0` throughout", and "`ack[1..4]` read `pre=0x62 postdrain=0x62 postbell=0x62`" — the figures handed over with attempt 15 | Three small ones, caught against the log. The span is **358 s** (`pkts=2` at t=20.688247, `pkts=436` at t=368.674276; the watchdog itself spans 25.634210–378.914276, 353 s). "Throughout" is **two counter samples**, both `drop=0 lenerr=0`, because `rxcnt` is bounded at 2 — the property is not sampled across the boot and the watchdog does not carry it. And **`ack[1]` read `0x00000000`, not `0x62`**, on all three of its points; only `ack[2]`–`ack[4]` read `0x62`. The last of these is not cosmetic: `ack[1]`'s zero **is** the refutation of the level hypothesis and the proof that write-1-to-clear works, so quoting all four as `0x62` would have discarded the single most load-bearing sample in the boot while appearing to strengthen the case. Also filed: the `kick` alternation is not perfect — `watch[44]`–`[46]` and `watch[56]`–`[57]` read `kick=1` consecutively with `pkts` and `cur` both flat, which is an idle link rather than a failed kick (36 `kick=1` to 33 `kick=0` over 69 complete samples), and the watchdog period is **~5.12 s**, not 5.00 s. Checked 2026-09-27. |
| The `ack[]` probe "returned `pre=0x62 postdrain=0x62 postbell=0x62`", therefore `0x50` "does not need to clear" — stated when attempt 15 was reported | **The figure is already corrected in the row above; this row records the *shape* of the error and the fact that it is a repeat.** `ack[1]` read **all zeros** on all three of its points, and it is the load-bearing sample of that boot: a level meaning "completions pending" cannot read zero with completions pending, so `ack[1]` is what kills the level hypothesis and proves the status write is the acknowledge. `ack[2]`–`ack[4]` read `0x62` and agreed with each other. **What was done with them was to generalise from the three samples that agreed and drop the one that did not** — and the dropped one was not noise or an outlier, it was the only sample taken while the hypothesis was still falsifiable. The conclusion drawn (`0x50` "does not need to clear") even came out *directionally right by luck*, which is the dangerous part: a correct-sounding claim reached by discarding the disconfirming evidence gets filed as settled and stops anyone re-reading the probe. **This is the second time `0x50` was mis-stated in this same shape.** The first is three rows above — "bit6 will not clear", read off attempt 14's single `after-w1c=0x00000040` while ignoring that 18 completions were pending and the register had every chance to re-assert between the two accesses. Both times the error was **treating a set of samples as a population and letting the majority speak for it**, when a register probe's samples are not interchangeable: each is taken in a different device state, and the one taken in the state that discriminates is worth more than the rest combined. **The rule: when a bounded probe's samples disagree, the disagreement is the result.** Quote every line or quote none; never summarise a probe by its modal value. Caught by checking the claim against the log, 2026-09-27. |
| "535 frames in **182 s**, zero drops" — the figures handed over with attempt 16 | **Both wrong, and the second one is a repeat of the row above's second item, one boot later.** 182.283875 is where the *capture* is cut off mid-line, not where reception ended: the interface came up at t=18.335814, the first frame landed t=18.332803 and the 535th between `watch[26]` (t=151.568545, `pkts=533`) and `watch[27]` (t=156.687766, `pkts=535`). It is **535 frames over ~138 s of reception**, inside a ~159 s interface-up window. And **"zero drops" is not in the log at all**: `drop=` appears exactly **twice** in 83,368 bytes, both from `rxcnt:`, which is bounded at 2 — at `pkts=1` and `pkts=2`. Frames 3 through 535 have no drop or length-error observation whatsoever, and the `watch[]` line does not carry those counters. **The standing trap, now named twice:** a bounded counter dump says nothing about the boot outside its bound, and `drop=0` printed twice reads like a property of the run when it is a property of two frames. The log's *end timestamp* is likewise not the *event's* duration — a truncated capture makes the run look longer than the thing being measured. Verified figures that did hold: `imr=0x00001018`, `isr=0x00000000` and `kick=0` on **31 of 31** watchdog samples; `pkts == irqs` at all 31; `irqs` reaching 535; `isr[1..8]` all `status=0x00000040 after-w1c=0x00000000` against a bound of 8; `txctl` `0x00000001` → `0x00000005` from `isr[2]`; and one `Linux version` banner reading `#31`. Caught by checking the claim against the log, 2026-09-27. |
| The bounded `ack[]` and `unmask[]` probes report a trustworthy sample index | **They do not, in build #29 — both emitted one line more than their bound with index 1 printed twice** (`ack[]` 5 lines for a bound of 4, `unmask[]` 9 for a bound of 8). Not a second `gbe_open()` and not a mixed log: the cause is ordering inside the single open. `gbe_start_engines()` takes the first interrupt from *inside itself* — `isr[1]`, `rxcnt`, `ack[1]` and `unmask[1]` are timestamped t=18.325820–18.334810, all **before** the `engines:` line at t=18.335814 — and `gbe_open()` then zeroes `ack_logged` and `unmask_logged` underneath the poll that already used them. **The comment immediately below those two resets reasons this exact hazard out for `irq_count` and declines to reset it**, so the trap was understood, written down, and then missed three lines higher in the same function by the same reader. Harmless to attempt 16's result — it costs one extra sample, not a wrong one — but a duplicated index in a bounded probe is precisely the thing that makes a later reader suspect two boots were mixed. Fix in #30: move both resets above `gbe_start_engines()`, or drop them as `irq_count`'s was. Filed 2026-09-27. |
| **Build #30's source is what is in the repo, and its image can be rebuilt from it** — the project's central provenance claim, applied to every build in `build-history/BUILDS.md` | **False for #30, and unrecoverably so.** `driver/ps4_gbe.c`, the in-tree copy and `/tmp/ps4_gbe_30.c.bak` in WSL all hold **#31's** `f298a789…4d0a6526`; the `.bak` was taken *after* the overwrite, so it is misnamed and useless. Both filesystems searched; no other copy exists. What survives is the **image** (`<kernels>\bzImage-5.4.247-ps4gbe-txbringup`, `5434e0b4…0f84ab`), the **object** (`/tmp/ps4_gbe_30.o.bak`, `4dbd867f…`, matching the hash `BUILDS.md` recorded at build time) and the **boot** (`logs/uart/putty-20260927-attempt17-txbringup.log`, banner `#32`). Nothing functional is lost — #31 is a superset of #30 and attempt 17's findings are captured — but **#30's exact image can never be rebuilt**, and no future claim may say otherwise. Cause: the build agent was told to wait for the tree to go clean, waited ~50 minutes, and installed its change by atomic rename instead. The rename was reasonable in itself — it removes the torn-read hazard by construction rather than by waiting — and it is **not** the error. The error is upstream of it: **#30's source had never been committed in the ~4 hours it was the live driver**, so there was nothing for the rename to preserve. The rule this argues for: **a build's source is committed before the next build starts**, not batched at the end of a session. Filed 2026-09-27. |
| Doc §7.6 / §7.7: the TX pool tail is "recomputed on completion" | It is **enforced at allocation**. `mts_tx` compares pool head against pool tail (`softc+0x32b0` / `softc+0x32b4`, init `0` / `0xa0000`), calls `mts_txeof` inline to advance the tail, and returns **ENOMEM** if that does not free enough. Recording it only in the completion path made it read as a statistic, so the driver tracked a head and no tail for four builds. That was safe by arithmetic — 193 × `ALIGN(1514,128)` < half the 655,360-byte pool — and the arithmetic was correct, but it was load-bearing by accident: three unrelated constants (the 63-descriptor stop threshold, the 1500-byte MTU cap, the wrap rule) happened to line up and nothing checked that they still did. Fixed in doc §7.6 and structurally in build #31. Filed 2026-09-27. |
| Doc §7.1 step 9 reads as an in-place `rmw 0x04 &= 0x7fffcfff` | The read is **hoisted**. `mts_init` reads `0x04` at `0xde1668a9` **before** PHY calibration, holds it across ~900 bytes of Clause 22 work, and writes `saved & 0x7fffcfff` back at `0xde166c2d`. Our driver does a fresh read-modify-write at step 9, so for any bit inside `0x7fffcfff` that calibration changes, the ROM restores the **pre**-calibration value and we keep the **post**-calibration one. Which bits those are is unknown — `0x04` has never been read either side of calibration on any boot — so this is a divergence to measure, not a known defect. Filed 2026-09-27. |
| Doc §7.4: `0x50` bits 20\|22 are answered by pulsing `0x9c` bit6 | The pulse is **step 1 of eight**. The full response is a TX restart: pulse `0x9c` bit6 low then high, free every TX buffer, reset all 256 descriptors to `word0 = 0x80000000` with `word2 \|= 0xffff0000`, zero prod/cons, `tx_free = 0x100`, re-write `0x44` and `0x3c`, then `0x34 \|= 1`. **With no `0x34 = 2` reset** — which is the load-bearing part, since it is a second ROM site that re-asserts RUN on an engine it never reset (doc §8 question 6). Filed 2026-09-27. |
| Doc §7.7: transmit completion is the end of the TX story | It omitted the **doorbell re-ring**. The ROM's ISR, on `0x50` bit7, runs `mts_txeof` and then tests `tx_free > 0xff`: fully drained clears OACTIVE, **otherwise it writes `0x34 \|= 4` again**. The ROM never relies on one doorbell surviving, and the document implied a port could. Filed 2026-09-27. |
| Doc §7.4 / §7.5: the RX drain ends at the drain | It does not. The ROM rings `0x38 \|= 4` after **every** RX drain, on the quiet path and on the `0x50` bits 1 or 5 path alike. Our driver already did this, so nothing was broken — but it was doing it without the document saying to, which is the same exposure as doing the opposite. Filed 2026-09-27. |
| Doc §3.4: bit3 = active and bit0 does not read back, with the ROM as a possible second source | **The ROM cannot corroborate either, and it never will.** Every `0x34`/`0x38` access in the ROM was enumerated: it reads **only bit1**, and only in the stop path, waiting for the reset to self-clear. It never reads bit0 or bit3 anywhere. Both decodes therefore rest **entirely on hardware observation**, which is strong here (twenty-seven post-RUN reads across two boots) but is one source, not two. Filed 2026-09-27. |
| Doc §3.6: `reg0x04=0x00000b18` reads as link **down** | The decode is right; **its validity window is not established**, and the reading is inconsistent with the same boots. `0x00000b18` is the value at **probe**, bit0 clear — yet attempts 16 and 17 went on to receive 535 and 542 frames on that link. So either bit0 is not meaningful that early or `0x04` needs re-reading after §7.1 step 9. The ROM's link handler (`0xde1681b0`, on `0x50` bit2) trusts bit0 directly with no settling delay, so it offers no guidance. A probe-time `0x00000b18` must not be read as evidence the link is down. Filed 2026-09-27. |
| `build-history/BUILDS.md`: build #30 "**Boot result:** UNTESTED" | **It booted**, as attempt 17, banner `#32`, ~388 seconds, and it is the boot that located the TX failure. The record said UNTESTED for the honest reason — no capture had been filed — but the capture existed the whole time inside the live PuTTY log and was never extracted, so the bring-up log skipped from attempt 16 to nothing while `build-history/BUILDS.md`'s **#31** entry quoted attempt 17's output as established fact. Two documents disagreeing about whether a boot happened. Extracted to `logs/uart/putty-20260927-attempt17-txbringup.log` (2,663,978 bytes, exactly one `Linux version` banner) and written up. This is the **third** time `logs/uart/latest.log` has gone stale across a boot; see the two rows above. Filed 2026-09-27. |
| mtsc is ~86 KB / 120 functions | That address range contained unrelated FreeBSD USB stack code. |
| mtsc is 7.8 KB | Attach-reachable only. The real figure is ~26 KB across both layers. |
| The mskc power sequence applies to Baikal | It is Aeolia/Belize only. `0xe00`–`0xfff` reads all zeroes on Baikal. |
| The crashniels 6.15 hang is TSC calibration | `calibrate.c` differs from working 5.4 by five lines (`ioremap`→`early_ioremap`). Not the cause. |
| **`stat1000` (MII register 10) was misread by two bit positions, and four builds of work were built on it** — bit10 read as local-receiver-OK and bit11 as remote-receiver-OK, so `stat1000 = 0x7800` was reported repeatedly as *"remote receiver OK, local receiver never OK"* | **Wrong, and checkable against the header it came from the whole time.** `include/uapi/linux/mii.h`: `LPA_1000LOCALRXOK` is `0x2000` (**bit13**), `LPA_1000REMRXOK` is `0x1000` (**bit12**), `LPA_1000FULL` is `0x0800` (bit11), `LPA_1000HALF` is `0x0400` (bit10). So `0x7800` decodes as: master/slave resolved with no fault, **local receiver OK SET**, **remote receiver OK SET**, link partner 1000BASE-T full-duplex capable, LP half-duplex not advertised (normal for every switch), and — the low byte being the idle error count — **zero idle errors**. **The 1000BASE-T PCS is fully trained in both directions. There is no stuck receiver and there never was.** Corrected in build #36's source, which prints the fields decoded through the kernel constants so the misreading cannot recur, and in `build-history/BUILDS.md`. **The shape of the error, which is the reusable part: a bit position asserted from memory rather than read out of the header it is named after, and then used as the foundation for four builds of reasoning.** This log already has rows for *inference stated as observation* (the console halt, the DHCP retries); this is the adjacent failure — **a decode stated as observation**. It reads exactly like a measurement, it survives every sanity check that asks "is the number right", and nothing catches it but opening the header. Corrected 2026-09-28. |
| "They hear us, we cannot hear them" — the narrative carried across builds #33, #34 and #35, and the frame for every link investigation in that stretch | **Retracted in full.** It is the bit10/bit11 swap above restated in prose. Both receivers report OK on every sample of every boot in which `stat1000` has ever been read — `0x7800` (attempt 20), `0x3800` (attempt 21), and the `0x4000`/`0x4800`/`0x3800` transients of attempt 19 that carry bit13 or bit12 — and the idle error count is zero. The asymmetry the sentence describes does not exist in the data. Retracted 2026-09-28. |
| "Signature of a mistuned analog front end" — offered as the **motivation** for the EFUSE trim hunt | **The motivation was fiction; the work is sound, and the two must not be confused.** There is no mistuned front end: both receivers train and the idle error count is zero. But the 18-row EFUSE decode that the fiction sent us after was **validated independently of it** — against Sony's own programmed values, read back from the part — and it **found and fixed real packing defects**. A correct result reached for a wrong reason is still a correct result. This row exists so that a future reader who finds the motivation retracted does not also throw out the decode. What the retraction does cost is the *priority*: the EFUSE hunt was run ahead of other work because it was believed to be on the critical path to first link, and that belief was unfounded. Corrected 2026-09-28. |
| "Forcing the PHY to SLAVE may break an asymmetric training failure" — raised as build #36's rescue stage 2 | **Deprived of its rationale, then eliminated by experiment.** Its rationale was "our local receiver never locks", which is the misreading above. It was shipped anyway because it was cheap and bounded, and attempt 21 ran it: `linkfix[1]` printed `master=0` **before** stage 2, so the PHY had already resolved as slave on its own and the write asked it to keep doing what it was doing; `linkfix[2]` reports `lnk=0 s1000=3800 localrx=1 remoterx=1 master=0` after. **Both of build #36's rescue candidates are eliminated.** Full write-up in `notes/hardware-bringup-log.md`, attempt 21. Closed 2026-09-28. |
| "The idle error count is **saturated at `0xff` in several samples**", offered as evidence of a mistuned receive path, with "observed `stat1000` values: `0800`, `08ff`, `3800`, `38ff`, `4000`, `4800`, `48ff`, `7800`" | **Two samples out of 288, and two of the eight listed values were never observed at all.** Every capture in the project was re-tallied 2026-09-28. `0x78ff` appears **once**, in attempt 19; `0x38ff` appears **once**, in attempt 20's early stage-boundary reads. **`0x08ff` and `0x48ff` appear in no capture in this repository or in the live PuTTY log** — they are in the list of observed values and were never observed. Both real `ff` samples land during PHY initialisation or an autoneg restart, which is not a measurement of a settled link. Every settled reading has an idle error count of **zero**: `0x7800` ×46 and `0x4000` ×98 in attempt 19, `0x7800` ×31 in attempt 20, `0x3800` ×60 in attempt 21. **The receive path is not erroring.** Two failures in one table: a count of "several" written from an impression rather than from `grep`, and a list of observed values that had grown two entries by symmetry — `08ff` and `48ff` are exactly what you get if you assume every base value must have an `ff` variant. Corrected 2026-09-28. |
| Build #36's rescue success test, as specified: "`lnk=1` **or** `stat1000` bit10 set" | **Caught before it shipped, by the build agent, and it would have been silent.** Under the correct mapping the bit that test names is `LPA_1000LOCALRXOK`, which is **already set on every sample**. Stage 1 would therefore have declared success on its first verdict and **swallowed stage 2 without a log line saying it had been skipped** — a boot that reports a win, changes nothing, and never runs the other half of the experiment it was built for. The shipped test is `BMSR` bit2 and nothing else. Recorded because it is the first time a misread bit position nearly cost an experiment rather than a build: the decode error had already propagated out of the analysis and into a **test predicate**, where nothing about it looks like a decode any more. Caught 2026-09-27. |
| "We are MASTER" — stated as a property of this hardware in build #36's entry and its source comments, on the strength of bit14 in `0x7800` | **True of the boot it was written from and contradicted by the next one.** Attempt 20 (build #35) reads `s1000=3800` at the pre-touch — master bit **clear** — and `0x7800` for the rest of the boot. Attempt 21 (build #36) reads `s1000=7800` at the pre-touch and `0x3800`, master bit **clear**, for all 58 samples afterwards. Same cable, same switch, same silicon; the resolution comes out opposite ways round in the two boots, in both directions, with our own initialisation as the only thing in between. Both receivers read OK and the idle error count is zero in all four states, so it is not a signal-quality difference. **Unexplained, and not to be carried forward as a fact about the device.** Corrected 2026-09-28. |
| `build-history/BUILDS.md`: build #34 "**Boot result:** UNTESTED", and build #36 "**Deployed:** no. Archive only, `D:` untouched" with "**Boot result:** UNTESTED" | **All three wrong, and the cause is the same one as build #30's.** #34 booted as attempt 19 and its capture had already been extracted to `logs/uart/putty-20260927-attempt19-efusetrim.log` — the entry was simply never updated. #36 **was** deployed to `D:\bzImage` and **has** booted, as attempt 21, banner `#38`; its capture was sitting unextracted in the live `PS4UART.log` alongside build #35's, exactly as attempt 17's was (see commit `3f27b49`). Both segments extracted 2026-09-28 and written up. **This is the third time a boot result has been recorded as `UNTESTED` while its log existed**, and the honest-default rule is being satisfied by an omission rather than by the absence of a boot — which is the failure mode it was written to prevent. The rule needs a companion: *before writing `UNTESTED`, grep the live capture for the banner.* Corrected 2026-09-28. |
| Build #36's own log string: "neither the soft reset nor forcing slave set local receiver OK" | **The misreading surviving into the source of the build that corrected it.** `localrx=1` is printed on the same line, by the same `printk`, from the correctly decoded field. The verdict it reaches — both candidates eliminated — is right; only the reason given is the retracted one. **Deliberately not patched.** `driver/ps4_gbe.c` is committed as the exact source of the image that produced attempt 21, sha256 `a3fcf010…cdb677`, and editing it would break the source-to-image mapping that makes that boot citable. It goes in the next build's source instead. Recorded 2026-09-28. |
| `driver/ps4_gbe.c:528`, in a comment block predating #36's correction: "`stat1000` bit10 (local receiver OK) clear in every…" | **Same misreading, same file, superseded ~230 lines further down** by #36's own correction block at `:760`. Left standing for the same reason as the row above — the file is a build artefact under hash — and listed here so that a reader who greps the source and finds the earlier comment first knows which of the two is current. The later block wins. Recorded 2026-09-28. |
| `build-history/BUILDS.md` #37: "**Six consecutive builds (#32–#37) are now uncommitted**" | **Only #37 was.** #32–#36 landed at `8dfc399` before #37 was built. The wrong claim is the #36 entry's "#32–#36 are all uncommitted" line carried into the #37 entry without being re-checked against `git log` — a stale carry-forward, the same shape as several rows above. The rule it invokes (a build's source is committed before the next build starts) was met this time. Corrected in place. Filed 2026-09-28. |
| The ROM's Clause 45 `0x00a2.1e` bit15 gates its entire `BMSR` link wait, so if it is clear on this part the fault is **upstream of link indication** — the working hypothesis behind build #37 | **Bit15 is SET, in all 189 samples of attempt 22**, from before the driver's first write through 592 s of runtime. The ROM would have entered its `BMSR` poll exactly as we do. Recorded as a **clean negative**: the hypothesis was well-formed, the build that tested it wrote strictly less hardware than its predecessor, and the answer eliminates the last candidate for "we are asking the wrong register entirely". Not a reasoning error — a hypothesis tested and refuted, filed so it is not re-derived from the same ROM branch. Two real results came out of it: bits 14 and 9 latch and clear on read (the two reads differ by exactly `0x4200` unsettled and `0x0000` settled, which is *why* the ROM reads twice), and steady state is `0xa9f8`. Filed 2026-09-28. |
| Doc §1, `README.md`, `notes/gbe-findings.md`, `notes/hardware-bringup-log.md` and the design spec: the PHY **"matches no known vendor prefix"** / **"has an identity, and it is nobody's"** | **It is a MediaTek MT7531-family gigabit PHY.** `03a2:9441` (`PHYIDR1 0x03a2`, `PHYIDR2 0x9441`, model 4, rev 1) is an exact match for `MT7531` in mainline Linux's MediaTek PHY driver. **The "not Realtek, not Marvell, not Broadcom" point is not withdrawn** — it was correct and it is still the answer to the community RTL8111H/G and Marvell claims. It was *incomplete*, and it was stated as though negative exhaustion over three vendors were the end of the search. The reasoning error is narrow and worth naming: the search space was taken to be the three PHY OUIs this project already knew, and nobody queried the raw ID against the kernel tree, which names it. Corroborated independently of the ID by the ROM's vendor page `0x52b5` (MediaTek's vendor extended page), by five Clause 45 registers named in `drivers/net/phy/mediatek/mtk-ge-soc.c` with devads matching — including `0x0115` at devad `0x1f`/VEND2, the one row in §4.2 that is not devad `0x1e` — and by that driver performing EFUSE REXT/TX_OFFSET/TX_AMP/TX_R50/TX_VCM calibration of the same kinds the ROM does. Doc §1.1 added, §4.1 and §4.2 annotated; `README.md`, `notes/gbe-findings.md`, `notes/hardware-bringup-log.md` and the design spec's superseded-claims table corrected. Sources: `drivers/net/phy/mediatek/mtk-ge.c`, `mtk-ge-soc.c`, `mtk.h`; <http://lists.infradead.org/pipermail/linux-mediatek/2021-May/024689.html>. Filed 2026-09-28. |
| Doc §4.2 trap 2: the two codes per register are an **inference** about what the halves mean | **Decoded, not inferred.** MediaTek names `0x0172` `MTK_PHY_RG_CR_TX_AMP_OFFSET_A_B` and `0x0173` `MTK_PHY_RG_CR_TX_AMP_OFFSET_C_D` — the halves are TX amplitude offsets for the four twisted pairs, two per register, A/B then C/D. The inference was right; it is now evidence. The marker is removed for `0x0172`/`0x0173` and **kept** for `0x0174`/`0x0175`, which mainline does not name: the reading there is still only-one-consistent-with-the-packing, and marking all four decoded on the strength of two would be the same over-reach this log exists to catch. Consequence now stated in the doc: clobbering a half mistunes **one specific cable pair**, which is why that trap fails on marginal cable and not on the bench. Filed 2026-09-28. |
| `cl45 write 0x003c.07 = 0x0000` recorded as an undecoded fixed write — doc §4 said only "devad 7 is touched once during calibration" | **It is EEE advertisement off**, and it is now named. Devad 7 is `MDIO_MMD_AN`; register 60 = `0x3c` is `MDIO_AN_EEE_ADV`. Mainline makes the same write for this part, commented *"with EEE advertised, some link partners fail to establish a stable link."* This **closes the textbook cause of "trains but will not link"** — the standard explanation for our exact failure shape — because the driver already performs the write, at the ROM's position. Ruled out by name rather than left untested, which is the difference between a cause eliminated and a cause nobody checked. Doc §4.2 annotated. Filed 2026-09-28. |
| **"Zero idle errors"**, stated unconditionally and repeatedly as the evidence that the receive path is healthy — doc §3.6 and §4.4, `README.md`, `notes/hardware-bringup-log.md`, and `driver/ps4_gbe.c:601` | **It is zero on every *settled* sample and it is not zero unconditionally, and the claim was load-bearing in the reasoning about whether the receiver is healthy.** Three samples across the project are non-zero, one per boot: attempt 19 `0x78ff`, attempt 20 `0x38ff`, attempt 22 `0x78ff`. Attempt 22's is the instructive one — `linkopen[0]` at t=21.072540, counter **saturated at `0xff`**, and since the counter clears on read and the previous read is the `pretouch` `s1000=7800` at t=2.452231, that `0xff` covers an **18.6-second window containing this driver's entire PHY calibration and its autoneg restart**. 188 of the boot's 189 reads of MII register 10 are zero and the 187 settled ones span 571 seconds, **so the conclusion survives intact — the receive path is not erroring.** What did not survive is the sentence. **The shape of the error, which is the reusable part: a claim that was true of the samples anybody would look at, generalised into a claim about all samples, and then used as a premise.** It is not the "stated from impression rather than from `grep`" failure two rows above — the grep was done — it is the next one along: *a correct measurement with its scope quietly dropped*. The scope was the part doing the work, because "zero on settled samples" tells a reader which samples to go and look at and "zero idle errors" does not. Qualified in doc §4.4 with the mechanism written out, and in §3.6, `README.md` and the bring-up log. ~~**Not corrected in `driver/ps4_gbe.c:601`**: that source is parked and deployed as build #38 and must not diverge from the image on the media — it is listed for the next build's source, with the §1.1 OUI and §4.1 Realtek comments.~~ **Corrected in the source in build #39** (banner `#42`, archive-only, UNTESTED): `driver/ps4_gbe.c:602` now reads "zero idle errors on every SETTLED sample", and `:610`–`:632` carry the evidence, the counter's semantics and the refusal of the opposite over-reach. #38's source stays parked and unaltered, so attempt 23 remains citable against it. The §1.1 OUI and §4.1 Realtek comments went in the same build. Filed 2026-09-28, closed 2026-10-02. |
| `notes/hardware-bringup-log.md`: attempt 19's `0x78ff` "is the only sample in the entire project that has ever been non-zero" | **Wrong when it was written, and wrong twice over now — and it was already contradicted by a row in this very log.** The row above it records attempt 20's `0x38ff`; attempt 22 has since added a third. Two documents in this repository disagreed about a countable fact, and the one that was right was the corrections log, which is the one nobody re-reads. **The lesson is not about idle errors.** A superseded sentence in a per-attempt narrative does not announce itself: the attempt-19 section is *finished*, so nothing sends a reader back to it when attempt 20 lands, and the correction went into the log instead of into the sentence the log corrects. Corrected in place with the strikethrough, per the rule. Filed 2026-09-28. |
| `notes/hardware-bringup-log.md`, attempt 22's contradiction table: `BMSR` bit2 clear "187/187" | **189/189.** The boot takes 189 reads of `BMSR` — `pretouch`, `linkopen[0]`, and 187 × `linkctrl` — all `0x7969`. 187 is the `linkctrl` count, which is what the section's own prose is about, but the table row is not scoped to `linkctrl` and the neighbouring cell in the same row says 189/189. **Understating your own evidence is the benign direction and it is still wrong**, and here it mattered: the two extra samples are the `pretouch` one, which is the only read taken before this driver writes anything. Corrected. Filed 2026-09-28. |
| **NOT A CORRECTION — the provenance rule working, filed as the one positive entry in this log.** A true and load-bearing claim was **withheld from the documentation** because its provenance was memory: that our §4.1 Token Ring entry 5 shares `c = 0x8fae` and the low half `0x0671` with MT7531's *"Increase SlvDPSready time"* write. Recorded at the time as: *"I derived MT7531's value from memory, not from a source, so under the provenance rule it stays out of the docs."* | **Now sourced, and it is exact — promoted to doc §1.1 corroboration 2.** Mainline `drivers/net/phy/mediatek/mtk-ge.c`, `mtk_gephy_config_init()`: `mtk_tr_modify(phydev, 0x1, 0xf, 0x17, SLAVE_DSP_READY_TIME_MASK, FIELD_PREP(..., 0x5e))` — channel `0x1`, node `0xf`, data `0x17`, field bits **[22:15]**. Our `0x8fae` decodes to bit15=1, bit13=0, ch=1, node=`0xf`, data=`0x17`: the same accessor at the same address. The halves were checked by arithmetic rather than taken on trust — `0x00060671 >> 15 & 0xff = 0x0c`, and substituting MediaTek's `0x5e` gives `0x002f0671`, so the **low half `0x0671` is byte-identical** and the two words differ in exactly one field and nothing else. **Why this is filed here at all.** Every other row in this log is a claim that went in before it was solid, and the cost is recorded in each. This is the same rule running in the other direction: the claim was true, it was *known* to be true, it was the strongest single piece of identification evidence the project had — and it stayed out until the source could be read rather than recalled. **Waiting cost several days and lost nothing**, because a true claim survives being checked. A memory-sourced constant that had turned out to be off by a bit would have put a false exact-match at the centre of §1.1, where it would have been quoted onward and been very hard to dislodge. Filed 2026-09-28. |
| Doc §8 C ranked `TrFreeze = 0x4` and `SlvDSPreadyTime = 0x0c` as the top two candidates for the link failure, on the grounds that our values differ from MediaTek's | **Both are Sony's own settings, not ours — CLOSED at attempt 23 (build #38, banner `#40`).** All six Token Ring nodes read back **bit-for-bit identical to our values at pre-touch**, before this driver writes anything, and `postcal` is identical to `pretouch`, so our six vendor writes are idempotent no-ops. The difference from MediaTek is real and it is **Sony's choice, not our defect**. **Not a reasoning error** — the candidates were well-formed and were ranked with the caveat that `0x4` might be a default rather than a defect, and the build that tested them deliberately wrote nothing so the answer could not be contaminated. Filed as the good kind: *measured before being changed*, which is the distinction §8 question 1 draws and the opposite of the EFUSE absence-of-evidence failure this log opens with. **The doc now warns against the fix rather than suggesting it**: `0x0c` is the value present on a console that browses the internet under Orbis, so moving to MediaTek's `0x5e` would move away from the only configuration known to work on this board. Filed 2026-09-28. |
| The working frame since attempt 17 — that the link failure is something **our PHY initialisation** gets wrong, which is what four builds of MediaTek comparison were for | **The whole class is eliminated.** Attempt 23 read the PHY's vendor configuration *before* the driver's first write and found **all of it already correct**. Trim, the paged sequence, the latch, timing, EEE, isolate, power-down, the `0x00a2` gate, master/slave, `TrFreeze` and DSP-ready timing are now closed with evidence rather than argument. The link is already down at the moment Linux first looks, so it went down at the **kexec boundary** with the PHY's register space intact and the cause is outside that register space. Recorded as a correction and not merely a result, because the *frame* was wrong and the frame is what directed four builds of work: comparing our programming against a vendor driver can only ever find places we differ, and it cannot notice that we do not differ anywhere that matters. **The read that settled it was available the whole time** — the pre-touch read added at build #35 for link state alone, extended to the vendor registers only at #38. The reusable lesson is to read the part before comparing documents about it. One genuinely unmatched difference survives: VEND2 `0x0403` (`CORE_PLL_GROUP4`) reads `0x1000` with `pll4-bypass=0` where MT7531's bring-up sets that bit — carried into §8 C with the caveat that MT7531's PLL group 4 is switch-level and may not apply to a discrete part. Filed 2026-09-28. *(That caveat's mechanism is itself corrected below, 2026-10-03: `0x403` is the PHY core PLL, not switch-level. The caveat survives on other grounds.)* |
| Doc §4.4 and `driver/README.md`: the non-zero idle-error samples are **"three"** and **"all at or just after a PHY init or autoneg restart"** — written 2026-09-28, the same day, in the row two above this one | **Four, and the fourth is not at an init.** Attempt 23 reads `stat1000 = 0x08ff` at t=385.199340, **366 seconds after the last autoneg restart**, on a settled link. **The generalisation was one boot old when it broke**, which is the point: the row above corrected "zero idle errors" into "zero on every settled sample" and then immediately asserted a second, unearned generalisation about *where* the non-zero ones sit — from three samples, all of which happened to be at inits. Same failure, same day, one level down: a pattern true of every sample so far, stated as a property. **The conclusion survives again and the sentence did not again.** What makes attempt 23's sample a finding rather than an artefact is that the two receiver-status bits clear at the same sample and never return. Corrected in doc §4.4 and `driver/README.md`. Filed 2026-09-28. |
| `notes/hardware-bringup-log.md` attempt 22 and doc §8 C: `0x00a2.1e` bit15 is **SET in every sample, 189/189**, therefore the ROM's gate is open and `0x00a2` is a **closed** question | **Qualified, not overturned — and it was a window effect.** Attempt 22 ran 592 s. Attempt 23 ran **1362 s** and found **exactly one transition in 440 gate samples**: bit15 clears at t=385.2, at the *same sample* `stat1000` loses both receiver-OK bits (`0x3800` → `0x0800`). Two registers, one flip, no other transition anywhere in the boot — so **bit15 tracks 1000BASE-T receiver status** and is not an unrelated gate that happens to be set. **The hypothesis attempt 22 killed stays dead**: bit15 was set throughout the window in which anything happened, so the ROM would have entered its `BMSR` poll exactly as we do. What was wrong is the *status* — `0x00a2` was listed among the questions closed by direct observation and it is not one. **The reasoning error is the reusable part: "N out of N samples" is a claim about the window, not about the register**, and a capture that ends at 592 s cannot speak about t=385 of a longer boot. Attempt 22's 189/189 was true and is still true; it was read as though the boot had no more to show. **This was nearly missed again here** — the first draft of attempt 23's section asserted the gate held `bit15=1` "as at attempt 22" from the pre-touch sample alone, and only a `uniq -c` over the whole capture caught the 320 samples that say otherwise. Corrected in doc §4.4, §8 C and the bring-up log. Filed 2026-09-28. |
| My own dispatch instruction for build #39: that `driver/README.md`'s line numbers for the three stale comment blocks were **off by 126**, because #38's own comment insertions had shifted them, so the real targets lay elsewhere in the file | **There is no offset. 601, 1642–1644 and 1652 are exactly right against #38's parked source**, which is what the build agent established by reading the file, and what I then verified myself: line 601 of `ps4_gbe-5.4.247-ps4gbe-trread.c` is the "zero idle errors" clause, 1642–1644 the "look Realtek" paragraph, 1652 the `oui %06x` format string. **The reasoning error is the whole of the row: I computed an offset from reasoning about an edit instead of opening the file.** I knew #38 had inserted a large comment block, inferred that numbers recorded in the README must therefore predate it, and asserted a correction to numbers that were already correct — a plausible mechanism preferred over a one-command check, when the README's numbers had in fact been written *after* #38's source was final. **The agent did not take the instruction on trust, and that is the behaviour this log wants from the receiving end**: a dispatch instruction is a claim like any other and gets checked against the file. No wrong edit resulted; the cost was an instruction that would have sent all three fixes to the wrong lines had it been followed. Filed 2026-10-02. |
| **NOT A CORRECTION — the second positive entry in this log, and the first filed against an agent's own in-flight work.** Build #39's **first attempt** compiled and linked clean and was discarded by the build agent itself before archiving. Its `:601` comment corrected "zero idle errors" into "zero on every settled sample" and then immediately added *"so the non-zero reads belong to bring-up, not to the settled link"* | **That addition is the unearned generalisation two rows above, one level down and on the same day — and it was caught before it shipped.** Commit `3e5d4a9` landed mid-build carrying the counterexample: `stat1000 = 0x08ff` at t=385.199340, **366 seconds after the last autoneg restart**, on a settled link, at the same sample where both receiver-OK bits clear for good, with `BMSR` bit2, `reg04` bit0 and `BMCR` constant across all 440 samples of that boot. The agent read the new commit, recognised its own draft comment as the same shape of error, rewrote it to **record the counterexample, refuse the generalisation, and scope the finding `n=1`** (attempt 23 is the only boot that ran long enough — 1362 s against attempt 22's 592 s — to reach t=385 at all), and rebuilt. **The discarded attempt consumed banner `#41`**: never archived, never deployed, so `#41` will never appear in a boot log — the same category as `#23` and `#25`, and the reason the build-number-to-banner gap is now **three, not two**. **The reusable part is the mechanism, not the virtue.** What caught it was a corrections log that was current enough to recognise the pattern in, and a fresh commit read *during* the build rather than after it. A build is cheap; a confident wrong comment in the shipped source outlives the build that carried it, which is the lesson of the row at `0x00a2` and of the `localrx=1` row. Filed 2026-10-02. |
| `mtsc_probe` is at **`0xde1655ef`** — in this file's function inventory, in `notes/gbe-findings.md` as the range "`0xde1655ef - 0xde165639`", and in three Ghidra scripts | **The entry is `0xde165530`.** `0xde1655ef` is the `cmp eax, 0x104d` vendor compare — **mid-function**, and an address with **zero incoming references**. It was never a function entry; it is simply where the constant search landed, and the probe's identity was read outward from there without ever walking back to the prologue. The real entry is confirmed independently by the device-method pointer tables, and this file's own *Driver extent* line has said `~0xde165530` the whole time without anyone reconciling the two. Consequence beyond the wrong number: `scripts/ghidra/AnnotateGbe.java` calls `createFunction()` on it, which **splits one function into two in the Ghidra database and recreates the split on every run** — see `notes/ghidra-annotations.md`. **The reasoning error is treating a search hit as a boundary.** A constant match tells you a function contains that constant and nothing whatever about where it starts. Filed 2026-10-02. |
| **`mtx_lock_flags` @ `0xde2b2eb0` takes 2 parameters** — the reading its body supports | **It takes 4.** `_mtx_lock_flags(struct mtx *m, int opts, const char *file, int line)`, ABI `(RDI=m, ESI=opts, RDX=file, ECX=line)`. The body reads only RDI and ESI; RDX and ECX are zeroed and never read, because `file` and `line` are consumed *only* by `KASSERT` (INVARIANTS), `WITNESS_CHECKORDER`/`WITNESS_LOCK` (WITNESS) and `LOCK_LOG_LOCK` (KTR), and this build has all three compiled out. The callee is not merely unhelpful, it is **affirmatively misleading** — it looks like a clean 2-arg function. All four call sites load RDX and ECX, with plausible `__FILE__`/`__LINE__` values (`0x12df`, `0x130f`, `0x56c`, `0x589`), and settle it; the table is in the softc map section above. **Not a correction to anything shipped — filed for the lesson, which generalises well past this function: a dead parameter is invisible in a callee and only recoverable from its callers.** Any argument of the form "the body only touches N registers, so there are N parameters" is unsound on any build with debug facilities compiled out, which is every retail build. Filed 2026-10-02. |
| Four hand-applied Ghidra labels: `log_set_deferred` @ `0xde353610`, its argument structure `astruct`, `global_deferred_log_count` @ `0xdf52bb98`, and `log_flush` @ `0xde2c5ce0` | **All four wrong, and all four stock FreeBSD:** `device_set_desc`, `struct device`, `bus_data_generation`, and `free(void *, struct malloc_type *)`. Evidence per symbol is in the *FreeBSD kernel symbols* section above. **Marked as a Ghidra-project correction, not a repository one** — these labels lived in the database and were never asserted in this documentation, so nothing written here was built on them. They are logged anyway because the shape of the error is the point: four unrelated kernel symbols were all read as one imagined "deferred logging" subsystem, because a plausible story about unfamiliar code is easier to reach for than the types. **The fix was not cleverness, it was importing the headers** — see `kernel-re.md`. Each rename kept the old label as a secondary symbol with a dated comment, and `/auto_structs/astruct` is untouched, so the superseded reading stays recoverable. Filed 2026-10-02. |
| **The kernel clock on this console runs at ~0.4x real time**, offered as a measured property, and then built on: that attempt 23's drop was "really 16 minutes in" and therefore "more like something that builds up, thermal or a counter saturating" | **Void in full. The clock is fine; the two readings came from different boots.** `t=26.768332` and `t=238.737937` both exist in today's capture and they are on **opposite sides of a console restart** -- line 3132 and line 6885 of `logs/uart/putty-20261002-attempt24-watchlong.log`, with the restart at lines 5586/5587 between them. Two monotonic clocks, each correct, each counting from its own power-on; the ratio between them measures nothing. **The reasoning error is a boundary crossed without being checked**, and this is its third appearance in this log -- the console halt and the `0x00a2` steady-state errors are the same shape. What makes this instance worse than those is the second step: a scaling factor was invented to *reconcile* two readings instead of asking whether they shared an origin, and then a mechanism (thermal, counter saturation) was reasoned on top of the invention. Both the factor and the mechanism are withdrawn. Caught on review. **Attempt 24 independently removes the thing the mechanism was invented to explain** -- the drop does not reproduce at all, so there is no accumulation to attribute. Filed 2026-10-02. |
| **Dispatch instruction for this commit: "my capture scripts slice from the *last* `Linux version` line, which silently follows a restart"** | **The diagnosis is right and the attribution is wrong, which makes it worse rather than better. `scripts/poll-putty-log.sh` does no slicing at all.** It appends new bytes from `PS4UART.log` and copies the result to `logs/uart/latest.log`; every boot the source has ever carried is still in the file. The last-`Linux version` convention is **not in any script** -- it lives only in each agent's head, re-derived per session, which is why it keeps failing silently and why no amount of reading the script would have revealed the hazard. **Today's capture is the demonstration**: two `#43` boots, the first reaching probe at t=3.146 and stopping with **zero** `linkctrl`, `watch` or `gate` samples, the second running 1114 s. Slicing from the first banner concatenates them and restarts every sample index from 1 mid-file; slicing from the last banner is correct *here* and was wrong twice today for comparing samples across the restart. **The requirement, recorded so the next helper is built to it:** a capture helper must (a) report every boot-start offset in the source rather than silently choosing one, and (b) refuse to compare or tally samples whose boot origin differs, instead of relying on the reader to notice. Noted in `scripts/poll-putty-log.sh`'s header. This is adjacent to the three `latest.log`-went-stale rows above and has the same root: **a capture step whose correctness depends on a convention no tool enforces.** Filed 2026-10-02. |
| The working frame for the MAC, carried since attempt 17 alongside the PHY one: that the link failure is something **our MAC initialisation** gets wrong. Spec §7.1's fifteen register-value writes had never been checked against what the register already held | **The whole class is eliminated, and this is the second half of attempt 23's correction rather than a new kind of error.** Attempt 25 (build #41, banner `#44`) read all thirty-two registers §3 documents before the driver's first write and again after `gbe_hw_init()` returns: **`postinit` is byte-identical to `pretouch` on 31 of 32**, the exception being `0x00` because a Clause 22 read writes it. **All fifteen writes are no-ops on arrival**, `0x54 = 0x00001018` among them — the operating interrupt mask, already loaded, on the register whose polarity cost fifteen boots. The one asymmetry runs the wrong way for a defect: Orbis sets `0x08` **bit27**, which we never write. **The reusable lesson is the one attempt 23 already taught, and it had to be learned twice because it was only applied to half the device.** Attempt 23's row below says "read the part before comparing documents about it"; that was acted on for the PHY and not for the MAC, where the four registers probe happened to sample (`0x34`, `0x38`, `0x04`, `0x50`) felt like coverage. A class-eliminating read is cheap and the reason to do it is not that a particular register is suspected — so the question to ask after any such result is *which other half of this device has not had the same test.* Carried into §8 C and the README. Filed 2026-10-02. |
| Spec §7.1, step 9: *"nothing in this repository has read `0x04` either side of calibration"*, so the ROM's hoisted-read divergence is an untested risk | **Already false when it was written, and now measured.** `gbe_stage()` prints `0x04` at every stage boundary and has since build #35; attempt 25's log carries **31 samples in one boot**, from `stage reset` before the PHY identifier read through `stage pg5` after the whole paged sequence to `stage engines` at t=20.64, reading `r04=0x00000b18` on **every one**. Both `bar0` dumps agree. **Calibration changes no bit of `0x04`**, so hoisted read and fresh read produce the same word. The divergence stays documented as a real difference from the ROM — a part whose calibration *does* touch `0x04` would hit it — but it is not a lead and must not be cited as untested again. **The reasoning error is an "untested" claim written without checking the instrument the driver already had**, which is the same shape as the row two above about computing an offset instead of opening the file: both assert a property of the repository without reading the repository. Corrected in §7.1. Filed 2026-10-02. |
| **Dispatch instructions for this commit, twice over:** that `0x9c = 0x0000006f` and `0x1c8 = 0x00a00000` had *"neither ever been examined"*, and that `0x3c`/`0x40`/`0x44`/`0x48` are the *"first direct confirmation of where Orbis's rings lived; the spec has inferred that aperture from IOMMU fault addresses until now"* | **`0x9c` is genuinely new; the other three claims are wrong, and §3's own table was the counter-evidence.** The "Live value" column has carried `0x1c8 = 0x00a00000`, `0x44 = 0x10000000` and `0x48 = 0x10004000` since the original consolidation (`d9a6171`), from the phase-1 `sky2` survey of 2026-09-22 — `notes/kernel-re.md` names it as that spike's surviving artifact. Attempt 25's values are **identical**, which makes them a confirmation across ten days and eleven builds and is worth more than a discovery would have been; the aperture was never inferred from fault addresses. The ring *current* pointers `0x3c`/`0x40` **do** differ from the recorded values, exactly as §3.5 predicts since the device advances them. **The same error is in `build-history/BUILDS.md`'s #41 entry and in the driver's own `gbe_bar0_dump()` comment** — "we had never done it for the MAC", "the other twenty-eight were written blind" — when twenty-four of the thirty-two already had a recorded pre-touch value. **The reasoning error is treating a register as unexamined because the current instrument has not read it**, forgetting an earlier instrument that did; what makes it the document's problem rather than an agent's is that §3's table is where both the claim and its refutation live, one column apart. Corrected in the BUILDS.md entry, in §3 under the table, and in the bring-up log; the driver comment is **left as built**, because those bytes are in an archived image and the hash is the record. #41's real additions are the `postinit` half, a reading from this driver on the current boot, and the **eight** offsets §3 left blank — of which `0x200` and `0x204` sit outside the survey's `0x000`–`0x1FF` window and had never been read at all. Filed 2026-10-02. |
| Spec §3.4: **"bit0 — run … It does not read back"**, with the flat instruction *"Treat bit0 as write-only; never poll it to decide whether the engine is running"* — stated as a property of a layout §3.4 calls **identical** across `0x34` and `0x38`. §8 question 6's prediction table inherited it, listing *"`after-run` without bit0 → RUN does not latch"* as the outcome *"consistent with §3.4"* | **True for `0x38`, false for `0x34`, and the generalisation was never tested on `0x34`.** Attempt 26 (build #42, banner `#45`) read `0x34` after RUN for the first time on any boot: `0x00000001` at `post-run` and on **all ten** poll samples out to +4880 µs, with `txctl=0x00000005` hundreds of milliseconds later. **Bit0 latches and persists on TX**; bit3 never asserts. `0x38` in the same function microseconds earlier reads `0x00000008` — bit0 consumed, bit3 set — exactly as §3.4 documents. **The reasoning error is a layout shared on the write side read as a layout shared on the read side.** All twenty-seven post-RUN reads behind the claim were `0x38`; §3.4 generalised them to both registers because the bits you *write* are the same, and read-back behaviour is not something a write-side layout fixes. **It was already contradicted inside this repository and stood anyway** — §8 B has carried *"`txctl` reads `0x00000001` after RUN"* since attempt 15, and the TX state summary in `build-history/BUILDS.md` repeated it for many boots. Two documents disagreed about a readable fact, which is the same failure mode as the `0x78ff` row below, and this time the stale half was the **specification**, which the hard rules say wins. §3.4 is corrected in place: the "identical layout" claim is scoped to the written bits, the bit0 and bit3 entries are scoped to `0x38`, and a new subsection records `0x34`'s behaviour with the verbatim sample block. §8 question 6's fourth row is marked as having come back inverted. **The replacement reading is labelled a reading, not a fact**: on TX the RUN request is *latched but unconsumed*, which is a different state from "started then stalled" — and the ROM reads neither bit anywhere, so there is no second source for it and there never will be. Filed 2026-10-03. |
| Doc §8 C, `README.md` and `notes/hardware-bringup-log.md` (attempt 23): MT7531's PLL group 4 handling is **"switch-level"** — *"part of bringing up the switch's core PLL, not a discrete PHY's"* | **Wrong on the mechanism; the conclusion it was supporting survives and gets stronger.** MT7531's switch COREPLL is brought up in a *separate* function, `mt7531_pll_setup()`, which touches only `MT7531_PLLGP_EN` and `MT7531_PLLGP_CR0` and never goes near `0x403`. The `0x403` access is a distinct step that mainline's own comment calls the **"PHY core PLL"**, so bits 4/5/6 are the PHY block's PLL. The caveat's conclusion — *may not apply to an on-die PHY reached through this MDIO window* — is **unchanged**, but the reason is now the transport rather than the block: mainline reaches `0x403` on a dedicated **control pseudo-PHY** at `MT753X_CTRL_PHY_ADDR(addr) = (addr + 1) & 0x1f` (`mt7530.h:735`), and not by a host Clause 45 frame at all — `mt7531_ind_c45_phy_read/write` drive the switch's own `MT7531_PHY_IAC` at switch offset `0x701C`. **The reasoning error is worth naming:** "MediaTek's driver is a switch driver, therefore this register is switch-level" is a guess at the block from the *driver's* subject matter, where the file names the function and the comment names the block. The practical cost of having it wrong was that it understated the caution — the real one also tells us our `(0x403, devad 0x1f)` coordinate match is **partly definitional** (`MDIO_MMD_VEND2 == 31 == 0x1f`), so a #43 readback is not on its own evidence that this part implements MediaTek's bit semantics. Same pass labelled bit12 of `0x1000` (`RG_SYSPLL_DDSFBK_EN`, no `MT7531_` prefix) as **inferred, not fact**. Sources: `drivers/net/dsa/mt7530.{c,h}` raw from `git.kernel.org`, cross-read at v5.12, v6.1, v6.6, v6.12 and master, plus the local 5.4.247 tree. Doc §8 C rewritten; `README.md` and the bring-up log corrected. Filed 2026-10-03. |
| `softc+0x30a0` is a **`struct ifnet *`** — the type the FreeBSD 9 import assigned it, carried in this file's softc map and flagged "typed, not discriminated" | **It is a pointer to `{ struct ifnet *, device_t, void *softc }`.** Decoded from the `mts` child attach, `0xde168b14`–`0xde168c67`: the struct is built there, stored at `0x30a0`, and `ifp->if_softc` is set to *it* rather than to the big softc. **The behaviour every existing site recorded is unaffected** — they all deref at offset 0, which is `S->ifp` — so this is a type correction with no downstream consequences, and it is filed because a non-zero offset off `0x30a0` would have been read as an `ifnet` field and is not one. The section is now closed rather than flagged; the thing it said would settle it ("a field only `struct ifnet` has") turned up in the same function as `ifp[0xa8] = 0x5dc`, `if_data.ifi_mtu` = 1500. Filed 2026-10-03. |
| Doc §3.6: the ROM's link-state handler *"trusts bit0 directly, **with no settling delay and no re-read**"* | **The re-read half is wrong.** `0xde1681b0` reads `0x04` **twice** — the second read on the revision-2 path, before it ORs in `0x474700` — and `mts_intr` then reads it a **third** time for its event push. The conclusion survives untouched, because all three reads are back-to-back on one interrupt with nothing between them, so none of them waits for anything. Filed because "no re-read" was offered as *evidence* for "no settling", and it is a different claim that happens to be false: the handler re-reads without settling, which is the stronger version of the same point. Doc §3.6 fixed in place. Filed 2026-10-03. |
| `driver/ps4_gbe.c`'s build-#32 rationale: *"the transmitter gates on link and so has never transmitted"* (`:817`, and *"the transmitter gates on link, which is the whole TX asymmetry"* at `:740`), carried in `build-history/BUILDS.md` as well | **No ROM evidence supports it, and the ROM-side search is now exhausted.** Both of the ROM's `0x04` readers have been read end to end; `0xde1681b0` touches no `0x34`, no `0x3c`/`0x44`, no ring, no `tx_prod`/`tx_cons` and no OACTIVE flag, and `SceGbeMtsPhyCtrl` writes no MAC register at all. **What is closed is the *software* gate** — there is no missing handshake and no link state the ROM pushes into the TX path — so that whole hypothesis family dies. **A *hardware* gate inside the MAC remains possible and is unfalsifiable by reading firmware**, since the silicon needs no help from the ROM to hold the transmitter off. This row is not "TX is not gated on link"; it is "the ROM contains no such gate, and the ROM can no longer be cited for one". Doc §3.6.1 carries the scoped statement. The two source comments are **not** edited here: `driver/ps4_gbe.c` carries build #43's uncommitted change and belongs to that build's record — logged in `driver/README.md` Known gaps and due with the next build's source. Filed 2026-10-03. |
| Doc §8 C, after attempt 25: *"**Something we fail to write** that Orbis never had to … The dump produces exactly one candidate of this kind: `0x08` **bit27**, set by Orbis and never written by us. One bit."* — carried as surviving hypothesis class 1, and repeated in the bring-up log (attempts 25 and 26) and the row above's own "the one asymmetry runs the wrong way" | **The candidate was never a divergence, and the attempt-25 dump already contained the refutation.** Attempt 27 (build #43, banner `#46`) reads `0x08 = 0x0f597c00` at **`pretouch`** and `0x0f597c00` at **`postinit`** — identical, **bit27 set in both**. We do not *fail* to write it; it **survives our entire initialisation untouched**, which is the opposite of the thing the class describes. **The reasoning error is reading "we never write it" as "it is not there after we run", which is only true of a register our init clears** — and attempt 25's own `postinit` dump, in which `0x08` is one of the 31 byte-identical registers, says it is still there. The claim and its refutation were one line apart in the same log, which is the same failure mode as the `0x1c8`/`0x44`/`0x48` row above (`0x9c = 0x0000006f`), where §3's table held both halves one column apart. **Consequence, and it is large:** class 1 had exactly one member, so the class is now **empty** — and with `0x0403.1f` BIT(4) written in the same boot, **"a register value this driver programs wrongly or omits" is empty for the whole device**, on both halves and, with `c9416f2`'s ROM pass, on both sides. §8 C's surviving list is rewritten as **two** classes, neither about a register value. Filed 2026-10-03. |
| Doc §8 C: *"**We have never written it.** Unlike the Token Ring rows, this is a register we have only read, so 'bypass is clear' is an observation about the part's state and not yet a difference between our programming and Orbis's"* — caution 2 on VEND2 `0x0403` | **No longer true, and this row is not a correction: it is the caution being discharged by the experiment it asked for.** Attempt 27 wrote the bit: `pll4: 0x0403.1f 1000 -> 1010 bypass=1 poweroff=0 took=1 ret=0`, with the `cl45[pretouch]` read immediately before still showing `0x1000`. **`CORE_PLL_GROUP4` BIT(4) is writable at Clause-45 `0x0403` devad `0x1f` over a host Clause-45 frame** — a new positive fact about the silicon, previously unknown. **Caution 1 of the same pair predicted the exact strength of that result in advance** and it is worth recording that it was right: because mainline reaches `0x403` through the switch's internal MDIO master at a control pseudo-PHY, and because `MDIO_MMD_VEND2 == 31 == 0x1f` makes our coordinate match partly *definitional*, a positive readback could prove only writability and **not** that this part implements MediaTek's bit semantics. It proved exactly that and no more; "bit4 means PLL bypass on this part" stays **unestablished**. And nothing downstream moved — `reg04=0x00000b18` 110/110, `bmsr=7969/7969` 110/110, `prod=56 cons=0 tx=0/0` on all 64 `txwatch` samples, `es[]` byte-identical to attempt 26's. **Filed as the caution paying off**, the way the provenance and build-#39 rows in the preamble are: a documented limit on what an experiment could show, written before the experiment, and matched by the outcome. Filed 2026-10-03. |
| Doc §3's "Live value" column, read as a property of the part — attempt 25's note under the table says *"Every value the column already carried is unchanged"* and lists `0x1c = 0x0003207b` as a first-ever reading, with no statement that it might not repeat | **One of the thirty-two is not stable across boots.** Attempt 27 reads `0x1c = 0x0003206d` where attempt 25 read `0x0003207b`; every other offset agrees, and `pretouch`/`postinit` agree with each other within this boot. `0x1c` is the multicast hash **command port** (§3.3), whose low-byte readback semantics §8 C already declines to decode ("conjecture, not a decode"), so **the delta is recorded and not interpreted**. Filed because the column reads as a list of constants and one row now has to be read as a sample — scoped in place under §3's table. Not a wrong claim so much as one that was never qualified, and the qualifier would have been free. Filed 2026-10-03. |
| Doc §8 B / bring-up log attempt 26: *"**`word2` has never been sampled on a queued TX descriptor on any boot**, so nothing here can state that the marker was cleared"* — logged deliberately as an open instrument gap, with "the next TX probe must dump `word1`, `word2` and `word3`" as the action | **Closed in the affirmative by the next boot, so this is a resolution rather than a correction.** Attempt 27's `txwatch` dumps `word1` and `word2`: `w0=30000149 w1=7fe00000 w2=00000000`, on every sample in the boot. **§5.4's `0xffff0000` idle marker *is* cleared on a queued descriptor**, `word1` is a valid buffer address inside the 31-bit-clamped aperture, and the descriptor is now verified field by field in **all four words**. The worry was correctly scoped when it was filed — §5.4's wording already made it unlikely and the record said so — and it is answered negative. **`word3` is still not dumped**; it is `0` unless TSO and no offload bit is set, so nothing turns on it, but the action item is only three-quarters discharged and is recorded that way rather than ticked off. **What this does *not* close is §8 question 3:** it confirms the *driver* writes §5.4's layout, not that the decoded field boundaries are right — that still needs a boot that transmits. Filed 2026-10-03. |
| `notes/mts-init-sequence.md`, “No per-function DMA window”: the stride-`0x400` register file at glue window `0x140000` is **“interrupt configuration touching bits 0 and 4 only”** | **It is the Sonics Silicon Backplane initiator-agent control array**, and bits 0 and 4 are an agent **reset** and a **quiesce request**. Decoded 2026-10-03 from `0xddff8a70`, which sets `BASE+0x20 |= 0x10`, polls `BASE+0x28` until `(v & 0x70) == 0` with `DELAY(10)` up to 1,000,000 times, prints `"IA reset timeout 0x%X, reg 0x%X"` on expiry, then sets `BASE+0x20 |= 0x01` — “IA” is Sonics' Initiator Agent. Two further string confirmations: `0xde23a910` is `ssb_disable_sonics_protection` and `0xde2b0700` is `get_sb_sonics_status`. **The reasoning error is naming a register file from its access pattern.** “A per-function array whose code only ever touches bits 0 and 4” is the shape of an interrupt-enable file, and that resemblance was taken for an identification while the function that writes it went unread; the strings naming it were in the same image the whole time. It matters because the thing misnamed is a **fabric master port** — exactly what a DMA engine sits behind — so the wrong name parked it as uninteresting for a week of TX hunting. Filed 2026-10-03. |
| **This file's reset-array map and `outside-bar0-draft.md`'s copy of it**: *"each block has **two** reset handles, a low bank at `0x14`+4k and a high bank at `0x64`+4k"*, presented as the array's layout, with the nine-row block table read as its contents | **The layout is right and the array is wider than it in both directions.** Attempt 28 (build #44, banner `#47`) read all thirty-two dwords of `+0x180000` and **index 0 — byte offset `0x00`, below the decoded low bank's first handle `0x14` — is held in reset**, as are indices 15, 17 and 19 (`0x3c`, `0x44`, `0x4c`), all of which sit above the low bank's last handle `0x34` and outside the high bank `0x64`–`0x74`. Four live entries, none of them a decoded block. **Not a wrong claim so much as one that was never scoped**, and the qualifier would have been free: the nine rows are *the blocks the ROM touches*, which was mistakable for *the blocks that exist*, and the ROM touching an entry was quietly standing in for the entry being there. The same shape as the §3 "Live value" row above — a table read as a census when it was a sample. The map's own preamble already said `get_sb_sonics_status` reaches agents `0x5c00` and `0x6000` and so *"the array is at least 25 blocks wide and the nine below are a selected subset"*, which is the correct reading stated one paragraph away from the table that invited the wrong one. Scoped in place. Filed 2026-10-04. |
| **Doc §8 C's class-3 experiment**, as designed: *"If all nine read `0` and all nine agents read `bit0 = bit4 = 0`, that is a clean, final negative on the whole clock/reset hypothesis family"* — and the plan in `outside-bar0-draft.md` which predicted the dump would show *"which of the four unknown entries is anomalous"* | **This is not a correction — it is a pre-registered criterion meeting a result it did not cover, and it is filed so the half-outcome is not read as either half.** Attempt 28's dump is **half clean**: all nine agents read `quiesce=0 reset=0 busy=0`, so the agent branch closes negative by measurement; the reset array does **not** read all zero, so the reset branch stays open. And the dump's calibration worked exactly as designed — USB's `0x24+4k` / SATA's `0x2c+4k` and their high-bank partners, all eight, read **released** on a console that boots and reads its own disk, which is what licenses reading the four non-zero entries as real. **What the design did not anticipate is the shape of the anomaly it found.** It expected one of the *known* unidentified entries to be anomalous and asked which; the answer is **none of them** — all five GBE candidates (`0x14`/`0x18`/`0x1c`/`0x20`/`0x34`, indices 5–8 and 13) read released, and the four held entries are in parts of the array the ROM sweep never reached. So the experiment neither closed the family nor named a candidate; it produced a new measurement that needs its own identification pass. Recorded as **§8 open question 7** rather than folded into class 3's verdict, and the pass/fail criterion is left standing unedited in §8 C because having set it in advance is what makes the half-outcome legible at all. **The reusable part:** an all-or-nothing criterion over an array is only as good as the census of the array, and the census was nine rows of a thirty-two-dword table. Filed 2026-10-04. |
| **`driver/ps4_gbe.c`'s `gbe_glue2_dump()` and build #44's own header comment**: the agent loop is written as `base = 0x140000 + i * 0x400` for `i` in 0–8 with the comment *"The nine Sonics initiator agents"* and `GLUE2_SSB_COUNT 9u /* 0xddff8760 walks nine blocks */` | **It reads nine agent slots and they are not the nine the ROM walks.** The loop reads bases `0x0000`, `0x0400`, `0x0800`, `0x0c00`, `0x1000`, `0x1400`, `0x1800`, `0x1c00`, `0x2000`. The nine decoded bases are `0x0c00`, `0x1000`, `0x1400`, `0x2000`, `0x3800`, `0x3c00`, `0x4000`, `0x4400`, `0x4800`, and `outside-bar0-draft.md`'s dump specification listed them explicitly. So **four** of the nine decoded agents were read and **five were missed**: `0x3800`/`0x4800` (SATA), `0x4000`/`0x4400` (USB) and `0x3c00` — the fifth GBE candidate. **The cost is specific and it is the calibration.** The SATA and USB agents were the control for what an un-fenced agent looks like, exactly as their reset handles are the control for a released block; the reset-array half of the dump self-calibrates and **the agent half does not**. Attempt 28's agent negative therefore rests on nine real agents plus the ROM's `bit4`/`bit0` decode, not on a measured reference, and is recorded at that strength. **The reasoning error is a count standing in for a set** — "nine" was carried from the ROM walk into a loop bound while the bases it indexed came from a stride, and the comment asserting the identification sat directly above the arithmetic refuting it. Same family as the `0x140000` row below, where a register file was named from its access pattern: in both cases the thing that would have caught it was reading the spec's own list rather than its cardinality. **The source comment is left as built** — those bytes are in an archived image and the hash is the record, per the convention in the `:817`/`:740` row above. ~~The fix is one line in the next build: walk the decoded bases.~~ **DONE in build #45** (`5.4.247-ps4gbe-cl45residue`, banner `#48`): the stride is replaced by a file-scope `static const u32 gbe_ssb_base[GLUE2_SSB_COUNT]` holding the nine decoded bases, which the loop indexes; `GLUE2_SSB_STRIDE` is left defined and unreferenced to document the stride. **The fix renumbers every label, which is the part that can mislead a reader of the two dumps.** Not one of the nine `ssbN` labels means the same address in #44 and #45, and the four addresses read by both are each renumbered — `0x0c00` ssb3→ssb0, `0x1000` ssb4→ssb1, `0x1400` ssb5→ssb2, `0x2000` ssb8→ssb3. Both builds print the absolute `+%06x`, so **compare the dumps by address, never by label**; the mapping table is in `build-history/BUILDS.md` #45 and the warning is repeated in `notes/hardware-bringup-log.md` where the two dumps will sit side by side. The reset-array half reads all thirty-two words and needs no table, so it is unaffected. Filed 2026-10-04, fix recorded 2026-10-04. |
| **This file's `0xddff8760` section**: *"The block resets are **never released**, so at exit nine blocks are held in reset with their fabric agents live"* — a true statement about the walk, carried where a reader reaches for the array's expected contents | **True of the walk and not of this console, and attempt 28 is what makes the distinction visible.** None of the nine blocks the walk asserts is held on a running console: indices 5–13 all read `0x00000000`. That is the *correct* and expected result — `0xddff8760` is reachable only from `icc_power.c`, the southbridge power-**down** path, and this console is up — so the dump is consistent with the walk not having run, which is the only thing it can say about it. **Filed because the sentence is the kind that gets quoted out of its path**: "nine blocks are held in reset" reads as a statement about the machine's state, and a future reader comparing it against a dump of four held entries at different indices would have had a contradiction with no way to see that the two describe different moments. Scoped in place. **Not a correction** in the sense the rest of this log uses — nothing asserted here was wrong — but the same hazard as the attempt-19 `0x78ff` row: a finished section that does not announce itself as superseded when a measurement lands beside it. Filed 2026-10-04. |
| **Build #45's manifest audit**, in `build-history/BUILDS.md`: *"**10 files in the archive with no manifest entry at all**"*, with `bzImage-5.4.247-ps4gbe-iommu-pt` named among them and described as *"mentioned once in prose but has no formal entry"*; the same audit reports **54** entries in `MANIFEST.txt` | **It is 9 files, and the file holds 55 entries. `bzImage-5.4.247-ps4gbe-iommu-pt` has a complete formal entry** — name line, `WARNING: DO NOT DEPLOY. This image PANICS ON BOOT …`, `size 9876992`, `sha256 324813D2…` (matching the file on disk and #21's entry in this log) and a full `notes` field. **The reasoning error is a parser whose record shape was narrower than the file's**, and the one entry that differs is the one that differs *because it is dangerous*: it carries a `WARNING:` line between the name and `size`, so a check expecting `size` immediately after the name read it as a bare filename and reported the file as unmanifested. The off-by-one in the entry count has the same single cause. **Two lessons, and the second is the reusable one.** First, nothing was wrong with the manifest — the checker was, and a checker that reports a clean subject as dirty is the shape of error that gets *acted on*, since the obvious remedy is to add an entry that already exists and so create the duplicate the same audit was checking for. Second, this is the `0x00a2` row's failure in a new place: a count standing in for a set. "10 orphans" was a number, not a list that had been re-derived against the file; the fix was to re-derive it, which took one pass and found the entry sitting there. The nine genuine orphans were entered 2026-10-04 and `MANIFEST.txt` now re-parses to 64 entries with every size and sha256 matching disk. Filed 2026-10-04. |
| **This file's clean negative 3**: *"Nothing in the ROM asserts or releases a reset handle for the GBE"*, with the same claim in doc §8 C as *"Nothing in the ROM ever writes the GBE's"*, in this file's `0xddff8760` “Against” bullet as *"Nothing in the ROM ever writes the GBE's handle"*, and in `outside-bar0-draft.md`'s clean negative 2 | **Wrong in one direction, and the correction is what identified the GBE's reset index.** The ROM *does* assert the GBE's handles — `+0x180000` offsets `0x20` and `0x74`, written `1` at `0xddff88f7` and `0xddff8906` inside the `icc_power.c` power-down walk, immediately after the walk calls the GBE's own BAR0 quiesce routine `0xde1653f0`. The two accurate statements are **(a) no GBE *driver* code asserts or releases them** and **(b) nothing in the ROM ever *releases* them, on any path**. (b) is the half the claim was reaching for and the half that was load-bearing — it answers *“is there a release step we are missing”*, and the answer is still no. **The reasoning error is a scope word doing two jobs.** “The ROM” was written meaning *the code that drives this device*, which for every other fact in that section is the `mtsc`/`mts` range, and was then read as *the image*; the complete 22-write call-site list sitting in the same sentence already contained the 14 walk writes that refute it, correctly counted and not attributed. The same shape as the `0x90d8`-occurs-once negative one bullet above, which is true of the image and was stated that way on purpose — here the quantifier slipped and nothing in the sentence showed it. **What makes this entry worth more than the usual row: being wrong was the finding.** Searching for who asserts the GBE's handle is what located the one device-specific callback in a nine-iteration walk, and that callback is what named index 8 / index 29 as the GBE's — both of which read `0` on hardware, so doc §8 question 7 closes negative for this device. A clean negative stated one quantifier too wide hid the positive result underneath it for a day. Corrected in place in all three live locations; `outside-bar0-draft.md` is a merged working record and is left as written, per the convention that the specification wins where a draft differs. Filed 2026-10-04. |
| **NOT A CORRECTION — a sweep window that excluded its own answer by construction, filed because the method is in `notes/kernel-re.md` and will be reused.** Every ROM pass over this driver has swept `0xde165530`–`0xde16c000`, stated as *“the `mtsc`/`mts` range”* in this file, in the doc and in three drafts | **The lower bound is `mtsc_probe`, and a GBE function below the probe was outside the window every time.** `0xde1653f0` is exactly that: a complete GBE shutdown sequence — BAR0 `0x54`, `0x34`, `0x38` and a Clause 45 write — **320 bytes below** the address the window starts at, with one caller in the whole image. Nothing was asserted wrongly: the range was always described as what it is, and the four earlier reads each stated their window honestly. **The defect is that the window's lower bound was an artefact and was never labelled as one.** `0xde165530` is where the *first function anyone had identified* happens to sit; it is not a section boundary, not a translation-unit boundary and not derived from anything. An upper bound of `0xde16c000` was chosen with a margin; the lower bound was chosen with none, so the range could only ever grow upward. **What found it anyway was not a wider sweep but a caller.** The function was reached from `0xddff88de` by following who asserts the GBE's reset handles — i.e. from the correction in the row above — which is the generalisable part: **a call-site walk crosses a window boundary and a range sweep cannot.** The practical fix for the next pass is to bound by caller and callee sets rather than by addresses, or at minimum to carry a margin below the first known function as well as above the last. Filed 2026-10-04. |
| **Doc §3.4**: *"`0x34` bit0 **latches at the RUN write and persists**. It is not write-only here"*, and *"**`0x34` bit3 never asserts** — it has never been observed set, on any boot"*. Both stated as properties of the register, from ten poll samples in attempt 26 and reproduced in attempt 29 | **Both are properties of the part while the transmit path is inhibited, not of the register.** Attempt 30 (build #46, banner `#49`) clears the PHY shutdown residue and `0x34` behaves the opposite way in every respect: `txctl[4]` reads `after-run=0x00000000` — **bit0 does not latch** — `0x00000000` on all 66 `txwatch` samples, and **bit3 reads back set on all eight doorbell samples** (`txctl=0x00000000->0x00000008`), transiently, clearing again at idle. So `0x34` is write-only on bit0 and status on bit3, exactly like `0x38`. **The reasoning error is stating a conditional observation unconditionally.** Both boots were measured correctly; what neither could see is that the part was in a state. The tell was available and was not read as one: the same subsection notes that `0x38` — the identical layout, microseconds apart — did the opposite, and "two identical registers disagree" is a prompt to ask what differs about the two paths, not to write down two different decodes. §3.4 now carries both tables and says which state each describes. Filed 2026-10-04. |
| **Doc §4.4**: *"The link went down at the kexec boundary with the PHY's register space intact, **which puts the cause outside that register space**"* — the conclusion of attempt 23's pre-touch sweep, carried for seven boots and load-bearing for the direction of the whole search | **The cause was inside the PHY's register space: bit4 of `0x0147.1e`.** The premise is correct and is in fact *why* the bit persisted — the register space does survive the kexec intact. The inference does not follow. "The whole vendor configuration is already correct" means **matching Orbis's running values**, and `0x0147.1e` is not one of them: nothing in either init path writes it, so there was no value to compare against and the register was never in the comparison set. **An exhaustive comparison is only exhaustive over the registers somebody programs.** This is the same reasoning error as the EFUSE row below — treating the absence of a thing from a search as evidence about the world rather than about the search. Corrected in §4.4 and §8 question 4. Filed 2026-10-04, attempt 30. |
| **Doc §7.1**: *"**Calibration changes no bit of `0x04`**, so the hoisted read and the fresh read produce the same word and the divergence cannot be the cause of anything here … it must not be cited as an untested risk again"* — measured at attempt 25 over 31 `stage` samples, all `r04=0x00000b18` | **True of every boot up to attempt 29 and false of the first boot that works.** Build #46 clears the residue at the top of `gbe_phy_calibrate()`, `0x04` bit0 sets 39 ms later, and **21 of attempt 30's 31 `stage` samples read `0x00000b19`**. So the hoisted read and the fresh read now differ, and they differ on bit0, which is inside the `0x7fffcfff` mask and is written back: the ROM would restore `0`, this driver writes `1`. **Almost certainly harmless** — bit0 is the status half of a register the ROM never writes link state into — and **not measured**. What is corrected is the claim's scope: the divergence is observable on this part, and a port reproducing the ROM's hoisted read must capture `0x04` **before the residue clear**, not merely before the EFUSE trim. The reasoning error is the closing instruction: *"it must not be cited as an untested risk again"* foreclosed on a measurement taken in one device state, one build before the state changed. Filed 2026-10-04. |
| **NOT A CORRECTION in the spec — a scribe's own draft of attempt 30, caught before it was committed, filed because it is this project's characteristic error arriving on the best day it has had.** The draft recorded *"the reclaimed descriptor reads `w0=80000000 w2=ffff0000` — the idle marker restored exactly as §5.4 says happens on reclaim, which independently confirms that section's claim on silicon for the first time"* | **`w2 = 0xffff0000` is the driver's own write.** `gbe_tx_reclaim()` executes `d->word2 = TXD_W2_IDLE` before the sample is taken, so the reading confirms that the reclaim path ran and nothing whatever about silicon. **The half that *is* a silicon fact is the other word**: `w0 = 0x80000000` means the **device** set bit31, because `gbe_start_xmit()` writes `word0` with OWN clear and the reclaim loop only advances on `w0 & OWN` — so §5.2's ownership convention is confirmed on TX for the first time, 71 times over. Caught by reading `gbe_tx_reclaim()` rather than the log. **The lesson is specific and reusable: when a descriptor field reads the value the specification predicts, check which side of the DMA boundary wrote it before claiming a confirmation.** A driver that writes a value and then reads it back has confirmed itself. §5.4 now carries the split explicitly. Filed 2026-10-04. |
| **Doc §8 B**: *"**WRAP at index 255 is still unobserved**"*, and §8 B's *"the highest `cur` observed is `0x7fffeb80`, 184 descriptors past the base, so index 255 was never reached"* — standing since attempt 13 | **Observed at attempt 30, on the RX ring, by accident.** `cur` reads `0x7fffef60` (offset `0xf60`, index 246) at `watch[49]` and `0x7fffe020` (index 2) at `watch[50]`, with `pkts` advancing 758 → 770: **twelve descriptors across the index-255 boundary against twelve frames delivered**, `kick=0` either side. So the device reads `word0` bit30, returns to the ring base rather than walking off the end, and loses nothing. **Not a correction of a wrong claim** — the statement was accurate when written and for seventeen boots after — but it is filed here because it closes a "still not established" item that three documents repeated, and because what closed it was a 362-second boot rather than any new instrument. **TX WRAP is still unobserved** (71 descriptors). §5.3 carries the arithmetic. Filed 2026-10-04. |
| **Doc §5.4**: *"still unexercised: … **WRAP at TX index 255** — 71 descriptors is 71, and the ring is 256"*, with the matching bullet in `notes/hardware-bringup-log.md` attempt 30's "What this boot does NOT establish" reading *"**Not WRAP on the TX ring** … the highest `txcur` is 71 descriptors past the base"* | **Exercised, and the engine hangs on it.** Attempt 31 reached index 255 twelve times in 321 s and the transmitter froze at `txcur=0x7ffffff0` every time; attempt 32 traverses WRAP cleanly at index **254** with a 255-entry ring and moves 1.5 GB. **The statement was already false of attempt 30's own boot when it was written** — that entry was recorded against a live capture at `txwatch[66]`, and the same boot went on to reach 255 packets / 30,498 bytes and park there for five byte-identical watchdog samples (`build-history/BUILDS.md` #47). So this is not "accurate when written and overtaken later", which is how the RX row below reads: it is **the live-capture hazard again**, on the best day the project had had. Doc §5.4 and §8 question 8 carry the result; the attempt 30 bullet is struck in place with the reason. 2026-10-09. |
| **`build-history/BUILDS.md` #48's boot result**, and the report it came from: *"**Three** stalls, three clean restarts: `tx=255`, `tx=510`, `tx=765`"*, *"`reg04=0x00000b19`, `up=1` on **38 of 38** samples"*, *"`isr=0x00000000` on all **twenty** `txwatch` samples"* | **All three low, and one of them wrong about which instrument it counted.** Against the completed 65,536-byte capture (`logs/uart/putty-20261009-113608-attempt31-isrsample.log`, md5 `ab28f7eb…`): **twelve** stalls, `tx` = 255 … 3060; **106 of 106** `linkctrl[]` samples across 316 s; and **124** sample lines carrying `isr`/`imr`, of which **62 are `txwatch` and 62 are the RX `watch` line** — so "all twenty `txwatch` samples" understates the count and misattributes it. **Every conclusion #48 drew is unaffected and all of them got firmer.** Filed because this is the **third** time a figure read from a live capture has been published as a figure about a boot (the other two are the "console stopped mid-line" row and attempt 30's TX WRAP bullet above), and because the fix is cheap and now standard: name the file, its byte count and its md5 in the entry's scope note, and say which sample index the figures are as of. Figures corrected in place in #48's entry with the original wording left standing. 2026-10-09. |
| **"12 recoveries, all successful, `txcur` back to `0x7ffff000` each time"** — the attempt 31 report's account of #47's recovery | **The restart ran twelve times; the register readback exists for four.** `txrst[]` is capped by `if (gp->txrst_logged < 4)` at `driver/ps4_gbe.c:2987`, so four lines are printed and all four read `txctl=0x00000000 txcur=0x7ffff000 base=0x7ffff000 errctl=0x0000006f`. That the other eight ran is certain — `gbe_watch()` prints `txstuck[N]` and calls `gbe_tx_restart()` in the same block with no path between them — and that they **succeeded** is established functionally, by `tx` advancing exactly 255 between consecutive stalls and reaching 3,115, not by observing `txcur`. **A bounded log makes "on all N" a claim about the cap, not about the device**, and the two must not be merged. Same shape as attempt 21's instrument row. 2026-10-09. |
| **"The residue clear reproduced for the third consecutive boot — `0x0147.1e` read `0x0010` and cleared to `0x0000` every time"** — the attempt 31 report | **Three consecutive *reads* of `0x0010`, two consecutive *clears*, at the time that was written.** Build #45 / attempt 29 read the register and did not write it — attempt 30's own comparison table records that as "no — read only" — so the clear had run on attempts 30 and 31 only. **Attempt 32 then made it four reads and three clears**, which is what the bring-up log now states. The arrival value is the figure worth repeating, because §8 question 4's reset-default caveat is about the arrival value; the clear reproducing is a statement about this driver. 2026-10-09. |
| **Doc §3.4**: *"bit3 asserts **transiently**, for as long as the engine is working, and reads clear at idle"* — recorded at attempt 30 from eight doorbell read-backs and 66 idle `txwatch` samples | **True of a healthy engine and false of a stalled one.** Attempt 31 holds `0x34 = 0x00000008` on **36 of 62** `txwatch` samples plus all twelve `txstuck` reads — **48 observations** — with `txcur` frozen at `base+0xff0` and `tx_packets` flat, the longest standing 46 s. So **"active" does not mean "making progress"**, and bit3 must not be polled to decide whether TX is healthy: a dead engine answers "active". The attempt 30 reading is reproduced by the boot's 26 healthy samples and is kept; what it cannot be read to mean is progress. This is the same failure mode as the row above about this register — a property measured in one state of the part, written down as a property of the register. Doc §3.4 qualified in place. 2026-10-09. |
| **Doc §7.4**, by implication throughout: that the eight-step TX restart is the **designed** response to a condition this device reports — *"`0x50` bits 20\|22 are a full TX restart"*, with `gbe_isr()` built to that reading and #47's watchdog described as reaching it "by another route" | **The condition never occurs on this part, so the sequence is ROM-faithful and the reason for running it is not the ROM's.** `isr = 0x00000000` with `imr = 0x00001018` on **124 of 124** sample lines at attempt 31, **including all 36 taken while the transmitter was frozen**, so bits 20\|22 are **not unserviced — they are not set**; attempt 32 reproduces it over 268 s and 1.5 GB with `status=` taking only `0x04`, `0x80` and `0x000000c0`. Build #48 existed to choose between exactly these two readings and wrote both down in advance, and `notes/tx-wrap-draft.md` item 6 asked for the measurement and named this consequence. **The steps are the ROM's; the trigger is ours.** The expected open question — "why does the interrupt never arrive" — does not survive either: there is nothing to interrupt about. §7.4 now says so explicitly. 2026-10-09. |
| **`build-history/BUILDS.md` #49**: *"**Deployed: no.** … Nothing was written to the USB"* and *"**Boot result: UNTESTED**"*, and the build report's *"`E:` currently carries #46"* | **All three superseded, and the last was wrong when written.** #49 was deployed to `E:` and hash-verified (`e3d8ccdd…`), and the media held **#48** (`2ad5990e…`) before the overwrite, verified by hash — not #46. **And #49 booted: attempt 32, and it is the fix** — zero stalls in 268 s, 4,589 clean ring laps, `tx = 1,170,378 / 1,519,887,431`, 847.6 Mbit/s from the driver's counter and an external browser speed test of 711 Mbit/s down / 768 up against attempt 31's 0.37 / 0.26. The "deployed: no" was accurate at the moment it was written and the deployment followed; the `#46` claim was an error about the state of the media. Recorded because the deployment hash check is the one step that would have caught it and did. 2026-10-09. |
| **NOT A CORRECTION — a cross-boot quote caught before publication, filed because it is the live-capture hazard's fourth appearance in two days and the first in a *new* shape.** The attempt 32 report quoted the probe line *"`[    2.432505] txring: using 255 of 256 TX descriptors, WRAP on index 254`"* as attempt 32's, while also naming the boot as the one beginning at the **last** `Linux version … #52` banner | **Those are two different boots.** `<the serial capture>` holds **two** `#52` boots. The first ran t=0 to t≈27, printed that `txring:` line at **t=2.432505**, reached `Baikal GBE at xx:xx:xx:xx:xx:xx, IRQ 34` at t=3.438275, and was restarted before a single `txwatch` sample — it carries no throughput data at all. Attempt 32 is the later boot and its own `txring:` line is at **t=2.452328**. Every other figure in the report is from the later boot and is correct; only the quoted line crossed. **The new shape is worth naming:** the previous three cases were a *truncated* capture read as complete, and this one is a *complete* capture containing two boots of the same banner, where "the last banner" and "the line I can see" disagree. A banner number no longer identifies a boot. 2026-10-09. |
| **Doc §4.4**: *"`0x7800`/master is the resolution a working link has been observed under"*, carried with the explicit caution that *"nothing here says a slave-resolved boot could not also transmit"* | **A slave-resolved boot transmits at line rate.** Attempt 32 settles at **`0x3800` on 57 of 57** samples and in the same boot moves 1,170,378 packets / 1,519,887,431 bytes at a peak of 847.6 Mbit/s, with an external speed test reading 711/768 Mbit/s. It also takes a **fifth** path through §4.4's table — `0x7800` pre-touch → `0x3800` settled, which only attempt 21 had taken, and attempt 21 transmitted nothing. **So the resolution gates neither the datapath nor the throughput**, and it is struck as a candidate explanation for anything on the TX path. **Filed as a resolution rather than a correction**, because the caution was written into the claim and is what survived: the sentence hedged correctly and the hedge turned out to be the finding. The instability itself remains unexplained. 2026-10-09. |
| `notes/hardware-bringup-log.md` attempt 13: the `sdb2` EXT4 storm means *"the boot media is the first suspect … It may equally be a **failing USB stick** or a bad cable rather than anything this project wrote"* — and the "~5,482 EXT4 errors" headline that storm was first reported under | **The device attribution was always right; the *cause* is the candidate re-reading. Filed as a candidate, not as a settled correction.** Attempt 34 (2026-10-10) logged three EXT4 errors on `sdb2` — `block bitmap and bg descriptor inconsistent`, `__ext4_new_inode:1112: doubly allocated?` — on a boot whose headline was a **SATA** PHY change, which is exactly the setup for misattributing them. They are not the SATA drive's: `sda` is the HGST on `ata1`, carries the PS4's own partition table, is **never mounted**, and has **zero** errors on any boot; **all ten `blk_update_request: I/O error` lines in all 44,791 lines of that capture are `dev sdb`, none is `dev sda`**; and `exception Emask`, `SError`, `hard resetting link` and `failed command` appear **zero times in the whole file**, which is what rules out transport damage — a mistuned PHY produces CRC errors and link resets, not bitmap inconsistencies. The damage is **pre-existing** (`error count since last fsck` runs **1 → 9 → 15** across the file's boots, with `initial error at time 1670569404: __ext4_new_inode:1112` identical in the earliest boot and the newest) and **almost certainly ours**: a sibling `#53` boot in the same file carries 32 `EXT4-fs error` lines, `EXT4-fs error: 6034 callbacks suppressed`, two `EXT4-fs (sdb2): I/O error while writing superblock` and a run of `ext4_end_bio:311: I/O error 10 writing to inode …` — **a stick that stopped answering mid-session**, which is what unplugging it live and hard-powering-off the console does. **So "a failing USB stick" may be the wrong half of attempt 13's disjunction, and the same unclean-removal damage may be what that storm was.** Three reasons this stays a candidate: those captures have **not** been re-read for `blk_update_request` or superblock-write failures; the "~5,482" figure was **already** corrected twice (it is a whole-file case-insensitive `ext4` match, and the "deteriorating across boots" trend built on it was retracted 2026-09-27 — both rows above); and nothing here distinguishes "the stick is being abused" from "the stick is also wearing out". **The transferable rule is a correction and is not conditional: attribute a filesystem error to a *device* before attributing it to a *cause*, and `sda` and `sdb` on this console are different physical links on different controllers.** Attempt 13's bullet amended in place with the candidate; outstanding action is `e2fsck -f /dev/sdb2`, which cannot run on the mounted rootfs. 2026-10-10. |
| **That a boot quoted in this repo can be re-read later from `<the serial capture>`** — the working assumption behind every scope note that identifies a boot by its banner and a line number, including attempt 33's *"line 19427 at the time of reading"* | **The boot attempt 33 was written against is GONE, and this is the live-capture hazard's fifth appearance and the first where the cost is the evidence itself rather than a wrong figure.** The file now holds **three** `#53` boots and **none** of them matches attempt 33's quoted `linkchg[1]` at t=20.621237 (they read t=21.856278, t=21.389945, t=20.561360) or its `txring:` at t=2.433024 (t=2.469868, t=2.474518, t=2.478419). PuTTY's log was restarted between the sessions, and **attempt 33's capture was never copied into `logs/uart/`** — only to `latest.log`, which attempt 34 then overwrote. So attempt 33's figures cannot be re-verified from anything in this repo or on this disk. **The reasoning error is treating an append-only log as an archive.** `AGENTS.md` already names `logs/uart/` as the archive and `latest.log` as a pointer; the step that was skipped is the archive copy, not the capture. Two consequences recorded rather than fixed: a banner plus a line number is **not** a durable identifier for a boot, so a scope note must quote a timestamp from *inside* the boot (attempt 34's does, which is how this was caught); and **every per-attempt entry whose capture is not in `logs/uart/` is now unverifiable in principle**. Attempt 34's own capture **is** archived, as `logs/uart/putty-20261010-074606-attempt34-sataphy.log`, md5 `7d5b1eb7886854e3f5eabf71f0b7452b`. 2026-10-10. |
| **Dispatch instruction for this commit:** that attempt 33's log carried *"`error count since last fsck: 1` with `initial error at __ext4_new_inode:1112`"* | **The `1` belongs to a different boot, and the number that matters is 9.** `error count since last fsck: 1` is in the **first** `#53` boot in the file (t=333.869834), which logged **zero** `EXT4-fs error` lines of its own — so that single error predates the whole session. The **third** `#53` boot reads **9** at t=338.267000 after logging **eight** errors of its own at t=18.88–21.05, including `mb_free_blocks:1452 … block bitmap corrupt` and three `ext4_free_inode:352: Corrupt filesystem`. **The conclusion is unaffected and strengthened: attempt 34's three errors are fewer than that boot's eight and are not an increase.** The `initial error at __ext4_new_inode:1112` detail is correct and is the same in both. **The reasoning error is the one in the row above** — a figure taken from "attempt 33's log" when three boots in that file share attempt 33's banner and the one attempt 33 documents is no longer among them. Caught before publication. 2026-10-10. |
| `notes/gbe-findings.md`: *"That `trace length 6` matches `ctlr->trace_len = 6` in the decompiled AHCI code"* — offered as a cross-check between the Orbis boot log and the decode | **The `6` in the decompiled code is the *initialiser*, not the computed value, and the computed value on this console was `4`.** `bpcie_sata_phy_init()` sets `trace_length = 6` and then overwrites it from `ctlr_->apcie_bpcie_buffer`, a `void *` declared in `drivers/ata/ahci.h:486` and **assigned nowhere in `drivers/ata`**, cast to `int` and bit-sliced; `6` also happens to be what the `apcie_bpcie_buffer == NULL` branch prints. Linux printed `Baikal SATA PHY Trace length : 4` on every boot up to and including attempt 33, so the "match" was a coincidence of a dead initialiser. **The reasoning error is reading a value out of decompiled source without following what overwrites it** — the same shape as the EFUSE row, where a conclusion was drawn from what a listing *said* rather than from what the code *does*. Build #53 pins the selector to `6` from the firmware's own board data and attempt 34 observed `: 6` from Linux for the first time, with the drive enumerating byte-identically (`SStatus 123 SControl 300` on all four boots). Corrected in place in `notes/gbe-findings.md`, 2026-10-10. |
| **The comment build #53 added to `drivers/ata/ahci.c`**: that the `apcie_bpcie_buffer == NULL` branch *"prints 6 and then **applies no tuning at all**"* | **That clause is false and the comment still carries it.** The NULL branch prints `6` and then falls into the same label as `case 6`, executing the identical twelve read-modify-writes. **The comment's conclusion is strengthened, not weakened** — the two paths agree on `6` precisely *because* they run the same register writes — but the clause should read that the NULL branch applies the **same** tuning `case 6` selects. Recorded in build #53's entry at build time and reaffirmed at attempt 34 rather than quietly fixed, because the source is in the kernel tree and not in this repo: fixing it needs a build. **Also still outstanding from #53, also deliberate:** the three `-Wunused-variable` warnings for `v55`/`v56`/`v57` at `ahci.c:2017`–`2019` — the decompiler temporaries whose only uses the patch deleted. The declarations were left so the hunk stays byte-identical to the reviewed diff and to the archived patch; deleting them is the operator's call. 2026-10-10. |
| **`build-history/BUILDS.md` #51 and #52: `Boot result: UNTESTED`**, left bare | **Still true of the images and no longer the whole truth.** Banners `#54` (#51) and `#55` (#52) appear in no capture and never will — both were archived and superseded for deployment. But #53's `ps4_gbe.c` is **byte-identical to #52's**, which is a superset of #51's and #50's, so **both builds' driver changes ran on hardware at attempt 34** under banner `#56`. The row exists because this is the same failure mode as the attempt-5 row above: `UNTESTED` is the honest default and it stops being honest once a log exists, and here the log exists under a *different build's* banner, which is how it would have gone unnoticed. Both entries amended with what did and did not get exercised — and **#52's carrier sync is in the "did not" column**: the ISR reaches `netif_carrier_on()` 16 ms before `gbe_open()`, so the helper had no transition to announce, and `mdio=0` on all 170 samples means the dead-MDIO path #52 fixes was never entered. 2026-10-10. |

---

## Still open

- ~~**EFUSE field packing — the highest-value unfinished decode.**~~ **CLOSED.** Decoded
  2026-09-27 (`0xde1662c0`–`0xde166728`), implemented in build #34, and confirmed on hardware
  2026-09-28: four of the destination registers are read-modify-write and already held
  **exactly** the value the decode computes — Sony's own programming, surviving the kexec.
  Eighteen fields out of four words, not the fourteen byte lanes that were guessed. The table,
  the two structural traps and both trustworthiness arguments live in **doc §4.2**; doc §8 A
  records the closure. The motivation once given for the hunt — a mistuned analog front end —
  is retracted in the corrections log above; the decode does not depend on it.
- **Resource identifiers.** The resource specs at `0xdf503d20` and `0xdf503d40` read as
  high-entropy bytes in the dump rather than plausible `{type, rid, flags}` triples — either
  the dumper caught that page mid-use, or something else lives there at rest. Does not block
  a Linux port, which discovers the BAR index and IRQ from PCI config space.
- **Interrupt status taxonomy.** Reg `0x50`'s error bits are only partly enumerated; see the
  doc §8 for `0x9c` bit6 and the shutdown PHY quiesce. Bits **1 and 5** are the newest
  entries — attempt 15 read `0x50 = 0x00000062` (bits 1, 5, 6) on all 70 watchdog samples,
  and only bit6 is decoded. (`0x54` polarity is no longer open — it is a **mask**, a set bit
  masks, doc §8 question 1. `0x04` bit0 is no longer open either — it is link-up; doc §3.6.)
- ~~**When to unmask, under edge-triggered MSI.**~~ **Closed by build #29 / attempt 16.** No
  acknowledge-before-unmask is needed: §7.4 has the ISR acknowledge on entry, so `0x50` is
  already clear when the poll path unmasks, and unmasking a *clear* source re-delivers on the
  next assertion — 535 times in that boot, and 542 in attempt 17. Doc §3.7.1 and §8 question 1
  own the measurement and the protocol. Whether unmasking a source that is **still asserted**
  re-delivers is still unobserved, and is uninteresting: no correct driver reaches that state.
- ~~**The TX engine reset we do and the ROM does not.**~~ **CLOSED — doc §8 question 6.** `0x34`
  reads `0x00000000` at `gbe_start_engines()` entry, after the reset and after the base writes,
  on attempts 26, 29 and 30 alike, so the bare `DMA_RESET` full-word store destroys nothing.
  **And it is no longer a TX suspect at all**: attempt 30 transmits 71 frames with the reset
  still in place. It stays, because without it the engine kept walking its Orbis-era ring after
  kexec and faulted every access.
- **NEW, and it is the answer to everything above it in this list's history: the PHY shutdown
  residue must be cleared on a kexec port.** `0x0147.1e` bit4, left set by the ROM's teardown at
  `0xde1653f0`, inhibits the transmit path and holds both link bits down. Doc §8 question 4,
  §7.1's required deviation, and `notes/hardware-bringup-log.md` attempt 30. **What bit4 does
  mechanically is unknown and is not findable in this ROM** — the register is unnamed in every
  MediaTek source searched, and the ROM only ever writes it, never reads it. Settling that needs
  a vendor SDK with a non-stub `mt7531_internal_phy_calibration()`, which is the same artefact
  §1.1's open list is waiting on.
- **Which southbridge blocks are reset-array indices 0, 15, 17 and 19.** Doc §8 question 7's
  surviving half, and it is **could not determine from this ROM** rather than not yet done: no
  instruction in the image references offsets `0x00`, `0x3c`, `0x44` or `0x4c`, the four callers
  of the raw-resource getter do not reach them, and the other southbridge's window is a
  different structure so nothing transfers. Two readings cannot be separated — unreleased
  resets for blocks this firmware never power-manages, or **not reset bits at all**, since the
  `bit0 = reset` semantic is decoded only from the four handles two PHY drivers bracket their
  own init with. **Does not block the port**: under both readings none of the four is the GBE's,
  whose handles are identified (`0x20`, `0x74`) and read released. Settling it needs a source
  this project does not have — a second southbridge revision, a populated block to compare
  against, or silicon documentation.
- ~~**Where the GBE sits in the `+0x180000` reset array.**~~ **CLOSED 2026-10-04** — low bank
  `0x20` (index 8), high bank `0x74` (index 29), Sonics agent `0x2000`, all three reading `0` in
  build #44's dump. Found from the one device-specific callback in the `0xddff8760` walk; the
  map above has the decoded hook and the inferred half, and doc §8 question 7 has the verdict.
  **Nothing here authorises a write to `+0x180000`.**
- **RX tag steering.** `mts_rxeof` has a path gated on the per-revision flag `softc+0x30e0`
  that inspects the frame at offsets 12-13 for EtherType `0x9100` and compares a field
  against `softc+0x310a`. Not read in full; not needed for a basic port. **The `msk`
  analogue was checked and does not fit** — see the template section below.
- ~~**What `softc+0x30a0` actually points at.**~~ **CLOSED 2026-10-03** — it is
  `{ struct ifnet *, device_t, void *softc }`, built by the `mts` child attach, and the
  `ifnet *` typing was wrong. Section above; corrections log has the row.

---

## FreeBSD `if_msk.c` as a structural template — 2026-10-02

With the FreeBSD 9 tree local, `sys/dev/msk/if_msk.c` is the closest structural relative
available: a two-layer FreeBSD GBE driver with the same parent/child shape. It is useful for
**what to look for**, and dangerous for **what to assume**.

### What genuinely matches

The function split is nearly one-to-one — `probe` / `attach` / `init` / `init_locked` /
`stop` / `intr` / `txeof` / `rxeof` / `ioctl` / `rxfilter` / `start` / `start_locked` /
`phy_readreg` / `phy_writereg`, all present in both. The parent/child split is literal:
`msk_softc` is the controller and `msk_if_softc` the port, exactly as `mtsc`/`mts`.

Two of our weaker identifications get independent corroboration from the shape:

- `msk_start` / `msk_start_locked` is the same pairing our **inferred** `mts_start`
  @ `0xde1693d0` sits in. Still inferred — a matching idiom in a cousin driver is
  corroboration, not a symbol — but it is a second reason to believe it.
- `0xde169750`, the 96-byte lock-wrap around `mts_init_locked`, fits `msk_init`'s shape
  exactly: `msk_init(void *xsc)` takes the lock, calls `msk_init_locked`, drops it.

### Candidate field names — **hypothesis, not findings**

From `msk_if_softc`, worth *looking for* in the mts softc and nothing more:
`msk_ifp`, `msk_miibus`, `msk_phyaddr`, `msk_framesize`, `msk_flags`, `msk_tick_ch`
(a `callout`), `msk_watchdog_timer`, `msk_vtag`. From `msk_chain_data`:
`msk_tx_prod`/`msk_tx_cons`/`msk_tx_cnt`, the `*_ring_tag` / `*_ring_map` / `*_sparemap`
triples, and `msk_parent_tag` — that last plausibly accounting for the five DMA tags in
`mtsc_attach`, which already has a parent tag at `softc[0]` and four children.

None of these is applied. They are search terms.

### Divergences — the warnings matter more than the matches

- **Field order is inverted.** `msk_ifp` is at offset **0** of `msk_if_softc`; our
  equivalent candidate is at `0x30a0`, with the DMA tags and rings occupying everything
  below. **Any `msk` offset applied to mts will be wrong.** Use the names, never the
  numbers.

  **Qualified 2026-10-03, and the warning survives.** The structs being compared were not
  counterparts. `msk_if_softc` is the *child* softc; our counterpart to it is the three-field
  struct `S` that `0x30a0` points **at**, and `S->ifp` is at offset **0** of that — the same
  place `msk` puts it. What sits at `0x30a0` is the *parent* softc's pointer to the child, so
  the "inversion" was an artefact of comparing a parent field against a child field. The
  instruction is unchanged: the two parent softcs share no layout, and `msk`'s numbers are
  still never transferable.
- **One mutex vs two.** `msk` has a single `mtx_init` call; `mtsc_attach` makes five. The
  dedicated MDIO mutex has no `msk` counterpart at all.
- **No taskqueue vs a kthread.** `if_msk.c` contains **zero** occurrences of `taskqueue`;
  mts runs a kernel thread — the string `"kthread_resume failed (gbe:phy_ctrl)"` at
  `0xde761fe5`, and the PHY-thread acquisition loop already recorded in the Ghidra
  annotations. Nothing in `msk` models this.
- **Dual-port vs single-port.** `msk` indexes everything by `msk_port` and carries
  `sc->msk_if[port]`; Baikal is single-port, so every per-port indirection in `msk` is
  structure we should *not* expect to find.
- **Descriptor and ring geometry — do not carry these over.** `struct msk_tx_desc` is
  **8 bytes**; ours is **16 and fully decoded** (doc §5). `MSK_TX_RING_CNT`/
  `MSK_RX_RING_CNT` are 256/256, or 384/512 under `MSK_64BIT_DMA` — our 256-entry software
  rings come from the decode, not from here. A shape to test against, never a size to
  assume.

### The `msk_vtag` lead was followed, and it does not fit

`msk_vtag` was the most promising candidate for the open `0x9100` steering path, so it was
checked. **It is the wrong mechanism**, and the difference is instructive:

- `msk` has **hardware VLAN strip** (`msk_setvlan` writes `RX_VLAN_STRIP_ON`), and the tag
  then arrives **out of band**, as a separate `OP_RXVLAN` / `OP_RXCHKSVLAN` entry in a
  *status ring*. `msk_vtag` is just the field that carries the tag from that entry to the
  `OP_RXSTAT` entry that follows it.
- `mts` has **no status ring**, and `mts_rxeof` reads bytes **12–13 of the frame itself** —
  i.e. the tag is still in the packet. There is nothing in `msk` shaped like that.

So the analogue is rejected, and with it the easy assumption that the path is VLAN handling
at all. Two things are worth recording in its place:

1. `0x9100` is the **pre-standard Q-in-Q service-VLAN TPID**, not `0x8100`. If this is VLAN
   work, it is specifically stacked-tag work, which is an odd thing for a console NIC.
2. **A competing hypothesis, better supported by evidence already in this repo:** that this
   is Sony's remote-management frame path, not VLAN. The gated flag `softc+0x30e0` is a
   *per-revision* flag; the compared field `softc+0x310a` is already identified as
   **wake-frame reply bytes**, immediately after the wake-frame sequence counter at
   `0x3108`; `mts_suspend` arms wake frames; and there is a `gbe:rmu` mutex. A proprietary
   management EtherType fits all of that; VLAN fits none of it.

**Neither is settled.** What would settle it is reading `mts_rxeof`'s gated path in full —
specifically whether the matched frame is delivered up the stack or consumed and answered.
Still not needed for a basic port.

---

# Southbridge DMA gating — investigated and ruled out (2026-09-26)

Decoded from the Orbis 9.00 kernel. The hypothesis was that the southbridge needs
something from the CPU side before the GBE's DMA engine addresses memory correctly.
**It does not.** Recorded because a clean negative is worth as much as a fix.

## The BAR0 register map is closed

115 `bus_space` accessor sites across `0xde165000`-`0xde16c800`, 100% resolved. Every
one resolves its bus handle to `softc+0x3068`. **There is no second bus resource**
anywhere in `mtsc`/`mts` — no second BAR, no alternate window, no hidden aperture.

The complete offset set is exactly the 32 offsets already in the specification.
Nothing outside it. Specifically absent:

- no physical-address base register
- no second descriptor-list pointer beyond the `(0x3c,0x44)` and `(0x40,0x48)` pairs
- no DMA context or channel selector
- no mode bit altering how descriptor `word1` is interpreted

`word1` is a bare 32-bit address: no scaling, no segmenting, no gate. RX buffers are
one contiguous `0x60000` block at `softc+0x32c8` with `0x600` stride, not scatter.

Caveat stated by the decoder: exhaustive for accesses *through the bus resource*. A raw
virtual address stashed elsewhere would be missed, but no such pointer is constructed
anywhere in the range.

## Only one access leaves BAR0, and it is read-only

`0xde1d9af0`, called 15 times, all inside `mts_init` — the EFUSE window at southbridge
BAR4 + `0xc000`. It sits in a stub table where every other window has a read/write
pair; **the `0xc000` window is the only one with no write stub at all.** `mtsc`/`mts`
never writes to the southbridge.

## One thing Orbis does that Linux does not

`mts_init` reads the PCIe Device Control field and, if it is 512, writes 2048 —
**Max Read Request Size raised to 2048 bytes**. The only PCI config write the vendor
driver makes. Now replicated in the driver via `pcie_set_readrq()`.

## No per-function DMA window

`APCIE_RGN_PCIE_BASE` (`0x1c8000`) **never appears as a code immediate anywhere in the
Orbis text segment** — the only byte matches are inside data tables. Orbis does not use
it. The one per-function-shaped register file, stride `0x400` at window `0x140000`, is
**the Sonics Silicon Backplane initiator-agent control array** — *not* interrupt
configuration, which is what this paragraph said until 2026-10-03 (corrections log). The
"bits 0 and 4 only" observation was right; the identification was wrong. Bit4 is a quiesce
request and bit0 is an agent reset, with status at `+0x28`. See "Outside BAR0" below.

## What Orbis's IOMMU entry actually does — `0xde02ade0`

Guarded on `(device_id & 0xf0) == 0xa0`, the southbridge family; `mtsc` is `0xa1`.
It builds a real 32-byte AMD-IOMMU Device Table Entry:

- allocates a fresh 4 KB page and **zeroes it** — the page-table root starts empty
- `V=1`, `TV=1`, Mode `0b011` (3-level), page-table root split across the DTE
- `IR` and `IW` set
- **DomainID = device_id**, so the GBE gets domain `0x00a1` to itself
- `IV` set for interrupt remapping
- appends an `INVALIDATE_DEVTAB_ENTRY` command

And a software IOVA aperture:

```
slot+0xb8 = 0x10000000    aperture base
slot+0xc0 = 0x30000000    span  ->  IOVAs allocated from [0x10000000, 0x40000000)
```

**This explains the stray high-address faults.** `0x1000c350`, `0x100740c0`,
`0x1014c140` are not random and are not addresses our driver produced — they are
**Orbis-era IOVAs from this aperture**, left behind in device state. It also matches
the live dump showing Orbis's own rings at `0x10000000` and `0x10004000`.

Honest framing: this is an IOMMU-side mechanism and Linux's generic DMA path *does*
replicate the equivalent (DTE, domain, IOVA allocation). It is not something only Orbis
does. What it buys us is the provenance of those faults.

## What the evidence now says about the low-address faults

`0x0, 0x10, 0x20, 0x40, 0x60, 0x80` is **descriptor index x 16 with no base added**.
Meanwhile the current-pointer register advances `0xffffe000` -> `0xffffe020`, so the
engine holds the correct ring address internally. Two different address paths: the
visible pointer has the base, the DMA request does not.

Checked and ruled out as causes: we write all four ring registers, in Orbis's exact
order (`0x44`, `0x3c`, `0x48`, `0x40`); and our RX pool is contiguous, matching the ROM.

---

# Outside BAR0 — the ROM's complete external surface for this device (2026-10-03)

Merged from `outside-bar0-draft.md`, which is kept as the detailed
working record: the full accessor table, every call-site count, the `0xddff8760` block list and
the extended-config protection windows live there. Scope of the sweep: Orbis 9.00 `kernel.bin`
code **outside** `mtsc`/`mts` (`0xde165530`–`0xde16c000`) that configures the Baikal GBE. Motive:
attempt 27 emptied the value-level hypothesis space and pointed doc §8 class 3 outward.

**Method, and it differs from the previous two reads.** Capstone x86-64 straight over
`kernel.bin` at the per-segment file offsets in this file's segment table. **Ghidra was not used
at all** — see `notes/kernel-re.md`.

## The headline is a negative, and it is the useful kind

**Outside BAR0, the Orbis ROM does exactly three things to this device:**

1. fifteen read-only EFUSE reads of five words (glue BAR 4 `+0xc000`, offsets `0x5c`–`0x6c`),
2. the `Max_Read_Request_Size` bump to 2048 (`mts_init`, `0xde1661ff`–`0xde166228`),
3. bus-master enable plus one MSI vector — `0xde165828` `pci_enable_busmaster`, `0xde1658cf`
   `pci_msi_count`, `0xde165926` `pci_alloc_msi`, `0xde165985` `pci_release_msi` on the failure
   leg, all four in `mtsc_attach`.

**All three are already in the port. There is no fourth thing.** So no outside-BAR0 *write*
Orbis performs is missing from this driver — that sub-class of doc §8 class 3 is empty.

## The glue driver, and why the enumeration is provably complete

`baikal_pcie.c` — `__FILE__` string at `0xde771060`; driver body `0xde1d99d0`–`0xde1da4xx`;
`"Baikal PCI Express glue"` @ `0xde77110b` used at `0xde1da1bf`.

Every glue register access in the entire ROM goes through **eleven accessor leaves** at
`0xde1d99d0`–`0xde1d9bb0`, each hardcoding one window offset, plus four leaf getters at
`0xde1d9be0`/`0xde1d9bf0`/`0xde1d9c10`/`0xde1d9c30` that vend a raw `struct resource *`.

**The proof is structural, not a search.** The bpcie softc pointer lives in **one BSS cell**
(rip-relative target `0xe0174600`) with **eighteen** rip-references across the whole 13 MB text
segment, and **every one of them is inside `0xde1d99d0`–`0xde1da420`** — inside `baikal_pcie.c`
itself. No other translation unit knows where the glue BARs are, so the accessors' call sites
*are* the whole glue surface. The four raw-resource getters were checked too rather than left as
a hole: **nine external callers** (`uart_bus_attach`, `rtc_mtk_attach`, `srtc_drv.c`,
`qafutkn_drv.c`, the icc tree, two bus attaches), and **none touches the reset array or anything
GBE-relevant**.

A second, structurally identical accessor family at `0xde2ceac0`–`0xde2cebf8` keys off a
different BSS cell and is the **Aeolia/Belize** glue, a different southbridge generation — not
this part.

## Which glue BAR each window is in — decoded from `bpcie_attach` rids

A PCI `rid` for a memory resource is the config-space offset of its BAR:

| softc slot | `bus_alloc_resource` site | rid | BAR | Windows served |
|---|---|---|---|---|
| `softc+0x00` | `0xde1da2cc` | `0x10` | **BAR 0** | none of the accessors |
| `softc+0x08` | `0xde1da259` | `0x18` | **BAR 2** | `+0x10a000`, `+0x140000`, `+0x180000` |
| `softc+0x10` | `0xde1da302` | `0x20` | **BAR 4** | `+0x4000`, `+0xc000`, `+0xe9000` |

`0xde1da26d` sets the rman description to `"Baikal Func 4 BAR 2"`, confirming the `0x18` slot.

**EFUSE is BAR 4** — which cross-checks "southbridge BAR4 + `0xc000`" above exactly, so this is
a validation of the method rather than a discovery. **But the reset and agent windows are BAR 2,
and this project maps only BAR 4.** It has never read either.

Glue `+0xc000` has **no write accessor at all** — every other window exists as a read/write pair.
The EFUSE window is read-only in the whole ROM, which independently reproduces the finding above.

## THE FIND — glue BAR 2 `+0x180000` is a per-block hardware reset array

One dword per southbridge block, **`bit0 = 1` holds that block in reset**, `bit0 = 0` releases
it. Decoded from two independent sites that each assert reset, program a PHY, then release:

- **USB PHY init, `0xde4453d0`** (`"[USB] Baikal USB PHY Init(V5_0)"`): `unit` from
  `device_get_unit(dev) != 0`, handles at `0x64 + unit*4` (`0xde44540d`) and `0x24 + unit*4`
  (`0xde44543d`); set both, clear the high one, run the EFUSE programming, clear the low one,
  `DELAY(2000)`.
- **SATA PHY init, `0xddfcc9a0`** (`"Baikal SATA PHY init"`): same shape, `unit` from
  `devid != 0x90d9104d`, handles at `0x6c + unit*4` and `0x2c + unit*4`.

So each block has **two** reset handles, a low bank at `0x14`+4k and a high bank at `0x64`+4k.

**SCOPED 2026-10-04 (attempt 28) — those nine rows are the blocks the ROM touches, not the
blocks that exist.** The whole thirty-two-dword array has now been read on hardware, and
**four entries are held in reset: indices 0, 15, 17 and 19** (byte offsets `0x00`, `0x3c`,
`0x44`, `0x4c`). Index 0 is **below** `0x14`, the low bank's first decoded handle; the other
three are above `0x34`, its last, and outside the high bank `0x64`–`0x74`. **None of the four
is a decoded block, and all nine decoded low-bank handles plus all five decoded high-bank ones
read released** — which is the calibration that makes the four readings trustworthy, since USB's
and SATA's entries had to read released on a console that boots and reads its own disk. The
`0x5c00`/`0x6000` note below said the right thing already: the nine are a selected subset.
Corrections log has the row; `docs/baikal-gbe-register-map.md` §8 question 7 has the open
question and the counter-argument.

Paired **1:1** with a **Sonics Silicon Backplane initiator-agent array** at BAR 2 `+0x140000`,
`0x400` stride: control at `+0x20` (**bit4 quiesce request, bit0 agent reset**), status at `+0x28`
(**bits 6:4 busy**). Three independent string confirmations: `"IA reset timeout 0x%X, reg 0x%X"`
in `0xddff8a70`, `ssb_disable_sonics_protection` at `0xde23a910`, `get_sb_sonics_status` at
`0xde2b0700`. "IA" is Sonics' Initiator Agent — a master-side fabric port, which is what a DMA
engine sits behind. This is what corrects the `0x140000` identification above.

`get_sb_sonics_status` also reads agents `0x5c00` and `0x6000`, so the array is at least 25 blocks
wide and the nine below are a selected subset.

| low-bank reset | idx | high-bank reset | idx | `+0x140000` agent | walk order | Block identity | #44 value |
|---|---|---|---|---|---|---|---|
| `0x14` | 5 | — | — | `0x0c00` | 7 | **unidentified** | `0` |
| `0x18` | 6 | — | — | `0x1000` | 8 | **unidentified** | `0` |
| `0x1c` | 7 | — | — | `0x1400` | 9 | **unidentified** | `0` |
| `0x20` | 8 | `0x74` | 29 | `0x2000` | 5 | **GBE — inferred, high confidence (2026-10-04)** | `0` |
| `0x24` | 9 | `0x64` | 25 | `0x4000` | 1 | USB unit 0 — decoded | `0` |
| `0x28` | 10 | `0x68` | 26 | `0x4400` | 2 | USB unit 1 — decoded | `0` |
| `0x2c` | 11 | `0x6c` | 27 | `0x3800` | 3 | SATA unit 0 (`00:14.2`, `0x90d9`) — decoded | `0` |
| `0x30` | 12 | `0x70` | 28 | `0x4800` | 4 | SATA unit 1 — decoded | `0` |
| `0x34` | 13 | — | — | `0x3c00` | 6 | **unidentified** | `0` |

**The GBE row was filled in 2026-10-04, and it is an inference with a decoded hook.** The
decoded part: the walk splices **exactly one** device-specific callback into nine otherwise
identical iterations, at `0xddff88de`, and that callback is the **GBE** quiesce routine
`0xde1653f0` — identified by `softc+0x3068`, `smi_cl45_write` and the `IFF_DRV_RUNNING` clear,
not by a name — sitting immediately before block 5's `reset_helper(0x2000)` and the `0x20` /
`0x74` asserts at `0xddff88f7` / `0xddff8906`. A quiesce is what has to precede a block reset,
and the GBE is the one block in the walk whose own driver never asserts its own handle, which
is why the ROM has to stop it from inside the walk. The inferred part: that the **high bank is
the PHY reset**, because only blocks 1–5 have one and blocks 1–4 are the four decoded PHY
devices, so the two-handle set is exactly the five blocks that have a PHY. **No string or table
in the ROM names block 5**, so this row is `inferred` and stays that way. **Nothing rests on
it** — all four fallback candidates (`0x34`, `0x14`, `0x18`, `0x1c`) also read `0` in the build
#44 dump, so every candidate for the GBE is released either way. Doc §8 question 7.

**And the `0x00`/`0x3c`/`0x44`/`0x4c` entries that build #44 found held cannot be named from
this ROM, nor from the other southbridge.** All **26** `+0x180000` access sites in the image
were recovered and back-disassembled on 2026-10-04 — 22 writes, 4 reads; 4 SATA, 8 USB, 14 the
walk — and **no instruction anywhere in the ROM references those four offsets**. The
raw-`struct resource *` hole was checked rather than assumed: `0xde1d9bf0`'s three callers use
`0x109000`, `0x108000` and the UART. The Aeolia/Belize twin at `0xde372ac0` touches offsets
`0x38`, `0x3c`, `0x40`, `0x48`, `0x4c`, `0x50` in *its* `+0x180000`, but it touches them as
**read-modify-write bit masks with dwords shared between blocks** (`0x4c` serves two, `0x3c`
serves three) and per-block masks like `& ~0x07`, `& ~0x3c7`, `& ~0xc8`. Baikal's window is the
opposite structure: one dword per block, written with a literal `1` or `0`, never an RMW. **The
layout is generation-specific and nothing transfers** — `0x3c` and `0x4c` appearing in both
lists is a coincidence between two chips' register files and must not be read as evidence about
the Baikal entries. Flagged here deliberately, because it is exactly the kind of near-miss that
otherwise becomes a corrections-log row later.

## `0xddff8760` — a block-table reset walk, and both sides of what it means

Nine blocks. Per block, in order: quiesce and reset the Sonics agent (`0xddff8a70`), **assert the
block reset(s) in `+0x180000`** with value `1`, then `agent+0x20 &= ~0x01` and
`agent+0x20 &= ~0x10`. **The block resets are never released**, so at exit nine blocks are held
in reset with their fabric agents live. The final agent write per block is a tail
`jmp 0xde1d9b50`, which is why an `e8`-only scan counts 18 `+0x140000` writes rather than 20.

**One device-specific callback is spliced into the nine otherwise identical iterations, and it
is the GBE's — decoded 2026-10-04.** At `0xddff88de`, between block 4's agent re-enable and
block 5's `reset_helper(0x2000)`, the walk calls **`0xde1653f0`**, the GBE BAR0 quiesce routine
in the function inventory above. Nothing else in the walk is device-specific. That is the whole
basis for the GBE row in the map above, and the walk order column there is this splice point's
coordinate. Block ordering, call sites and index arithmetic are in the map; the per-site offsets
and the proof that the site list is complete are in
`reset-array-draft.md`.

**The whole chain is reachable only from a halt, and it therefore cannot have run on any boot
this project has taken — decoded 2026-10-04.** `0xde3bf4b0` has exactly one caller,
`0xde72acf7` inside `0xde72acc0`, which prints `"mDBG: Power down...\n"` (string `0xdeafa56d`),
`DELAY(0xf4240)` — one second — prints again, sets a global, calls the southbridge power-down,
and ends `hlt ; jmp .` at `0xde72ad00`–`0xde72ad01`; its own caller `0xde72acb0` follows the
call with `ud2`, i.e. `noreturn`. **Checked for the obvious hole rather than assumed:**
`0xde1653f0`, `0xde3bf4b0`, `0xde23b2c0` and `0xddff8760` each occur **zero** times as a
little-endian 64-bit pointer anywhere in the image, so none is reachable through a method table
or an event handler. The chain is rel32 only and it terminates in a halt. This is what makes
the fourteen zeroes in the build-#44 dump the *expected* reading rather than a puzzle, and it
is why there is no missing release step to look for: nothing released those blocks after the
walk, because the walk never ran.

**That is a statement about the walk, not about this console — scoped 2026-10-04 (attempt
28).** None of the nine is held on a running machine: indices 5–13 all read `0x00000000`, and
all nine agents read `quiesce=0 reset=0 busy=0`. Which is the expected result, because this
function is reachable only from `icc_power.c`, the southbridge power-**down** path. The dump is
consistent with the walk not having run, and that is the only thing it can say about it. Do not
read "nine blocks are held in reset" as the array's resting contents.

Reachable only from `0xde23b2c0` (dispatch on `0xde064520() & 0xff0000`: `0x30000` → Baikal
`0xddff8760`, `0x10000`/`0x20000` → the Aeolia/Belize twin `0xde372ac0`, else return), which is
called only from `0xde3bf4b0` in **`sys/dev/scesb/icc/icc_power.c`** (that TU's `__FILE__` is
referenced at `0xde3bf2b8`/`0xde3bf326`). So it is the southbridge power-down path — and it is
exactly the "routine that walks a table of device blocks asserting resets" that was asked about.

**Stated both ways, because the counter-argument matters:**

- **For.** Four of the nine entries are unidentified; `00:14.x` is eight functions; the GBE is a
  DMA master on this fabric and must have an initiator agent, so it has to be one of them. The
  candidates are `0x14`/`0x0c00`, `0x18`/`0x1000`, `0x1c`/`0x1400`, `0x20`/`0x2000`,
  `0x34`/`0x3c00`. ~~**Which one is the GBE is NOT decoded**~~ — no table or string names them,
  and the three decoded assignments all come from a device's own driver asserting its own
  handle, which the GBE's driver never does.
  **IDENTIFIED 2026-10-04: it is `0x20` / `0x74` / `0x2000`** — walk block 5, the one the
  spliced GBE quiesce callback `0xde1653f0` precedes. Inferred, high confidence, and the map
  above has both the decoded hook and the inferred half. The GBE's driver still never asserts
  its own handle; the walk in `icc_power.c` does it for the GBE, which is the whole reason the
  callback exists and the whole reason this index was findable at all.
  **MEASURED 2026-10-04 (attempt 28): all five candidate reset entries read `0x00000000`**, and
  four of the five candidate agents (`0x0c00`, `0x1000`, `0x1400`, `0x2000`) read
  `quiesce=0 reset=0 busy=0`; `0x3c00` was missed by the dump. So **if the GBE is one of these
  nine blocks, its block is not held in reset and its agent is not fenced.** The four entries
  that *are* held — indices 0, 15, 17, 19 — are none of these five, and which blocks they are is
  not decoded either. §8 question 7.
- **Against.** ~~**Nothing in the ROM ever writes the GBE's handle**~~ — **corrected 2026-10-04:
  the walk above asserts `0x20` and `0x74`. What is true is that no GBE *driver* code touches
  them and that nothing in the ROM ever *releases* them, on any path.** The bullet's actual
  argument is unaffected and is the half that held up: **our RX DMA works** —
  74 frames, interrupt-driven. RX DMA is a Sonics initiator transaction, and a device normally
  has one initiator port for both directions, so if the GBE's agent were isolated or its block
  held in reset, RX would not work either. On the straightforward reading this array is **not**
  the TX gate. That is a real negative and is not being talked around.
- **Why it is still the next read anyway.** It is a pure read, it is the only identified
  reset/clock-gating surface for this device, and it has never been read once. The two
  sub-hypotheses it would settle — that the GBE has two handles like USB and SATA and a kexec
  left one of the pair asserted, or that the bank split is read-path/write-path rather than
  controller/PHY — are **speculation**, flagged as such, and the measurement does not depend on
  either being right. Doc §8 class 3 has the experiment and what all-zero would close.

## Clean negatives, each recorded as closed

1. **No code outside the driver reaches the GBE by PCI identity.** Every 2-byte occurrence of
   `0x90d7`–`0x90de` in the 7.4 MB of text was swept and filtered to those Capstone confirms are
   an instruction immediate. **`0x90d8` occurs exactly once: `cmp ecx, 0x90d8` at `0xde1655ff`,
   inside `mtsc_probe`.** (`0x90db`, func 4, occurs in seven probe routines — the drivers that
   attach behind the glue — which **confirms the func-number ↔ device-id mapping**, hence
   `0x90d8` = `00:14.1` = the GBE.)
2. **There is no MAC↔PHY port-mode or interface-clocking step in this firmware to be missing.**
   Image-wide string sweep for `rgmii`/`RGMII`/`sgmii`/`trgmii`/`gmii`/`GMII`/`mdio`/`MDIO`/
   `txclk`/`refclk`/`clk_`/`clock_en`/`pwrgate`/`power_domain`/`deassert`/`rstn`: **zero hits
   relating to Ethernet.** The only `MII` string in the image is `"MII bus"` at `0xde754063`,
   referenced once from `0xde0ecf00` — FreeBSD's generic `miibus` description, which the Baikal
   driver does not use (it has its own `phy_read`/`phy_write`). Every `PLL`/`CLK`/`clock` literal
   belongs to the GPU display path, SDHCI or the GC clock tree. **Eliminated by an image-wide
   string sweep, not by argument** — and the hypothesis was the dispatching agent's own.
3. ~~**Nothing in the ROM asserts or releases a reset handle for the GBE.**~~ **WRONG IN ONE
   DIRECTION — corrected 2026-10-04.** The ICC power walk *does* assert the GBE's handles,
   offsets `0x20` and `0x74`, at `0xddff88f7` and `0xddff8906`, immediately after calling the
   GBE's own BAR0 quiesce routine `0xde1653f0`. The two accurate statements are:
   **(a) no GBE *driver* code asserts or releases them** — the assertion lives in
   `icc_power.c`'s walk, not in `mtsc`/`mts`; and **(b) nothing in the ROM ever *releases*
   them, on any path.** (b) is the half that was load-bearing — it is what answers "is there a
   release step we are missing", and the answer is still no. The correction is what *identified*
   the GBE's index, so it reads as a finding rather than a retraction; corrections log has the
   row. The complete set of `+0x180000` writes is 22 call sites: 3 SATA, 4 USB, 14 the ICC
   power walk, 1 the SATA tail — plus 4 reads, for 26 access sites in all.
4. **Glue `+0xc000` (EFUSE, BAR 4) has no write accessor anywhere in the ROM.**
5. **`sys/dev/scesb/icc/icc_device_power.c` is the Blu-ray drive**, not a device power table —
   its strings are `bd_drive_operable` / `bd_drive_inoperable` / `bd_drive_suspend_handler` /
   `icc_device_power_control(%d->%d) failed`. Not a candidate.
6. **The other `+0xc000` EFUSE readers are SATA and USB, not a shared GBE path.** `0xde4455a0`
   reads `0x44`/`0x50`/`0x54`/`0x6c` for USB, `0xddfcc9a0` reads `0x48`/`0x6c` for SATA.
   `mts_init`'s `0x5c`/`0x60`/`0x64`/`0x68`/`0x6c` does not overlap the USB set and overlaps SATA
   only on `0x6c` — consistent with `0x6c` being the shared EFUSE-valid word, as the driver
   already assumes (`EFUSE_VALID = 0xC06C`).

## New: `SceGbeMtsPhyCtrl` reads one byte of per-console NVS

**Decoded fact.** `SceGbeMtsPhyCtrl` calls `icc_nvs_read(block = 4, offset = 0x38, length = 1,
dst = &stack_byte)` at **`0xde169a42`**. The callee is `0xde0cf310` in
`sys/dev/scesb/icc/icc_nvs.c` (own string `"icc_nvs"`; first argument range-checked `<= 4`). It
is the only non-generic external callee in the whole `mts` range that was not already known — the
range's complete external-callee set is 64 targets, all the rest FreeBSD infrastructure plus the
known EFUSE read, the MRRS pair and the Sony chipset-id family (`0xde064520` =
`chipset_word & 0xff0000`, `0x30000` = Baikal).

**Inference, labelled as such.** A **PCB trace-length or board-variant code** is the obvious
reading for a single byte consumed by a PHY-control thread, by analogy with the two other PHY
paths in the same ROM: the USB path prints `"[USB] NVS is not written trace length"`,
`"SS trace length:%d HS trace length:%d"` and selects among `"[USB] SS trace 1-4 inch setting"` /
`"5-7 inch"` / `"8-11 inch"`, and the SATA path prints
`"Baikal SATA PHY Trace length : %d"`. **What the byte gates is not decoded.**

**Why it is recorded even though it is not a TX candidate** (it is in the link-repair thread, not
the TX path): it is **an external input this port does not read and cannot emulate**. The ROM's
PHY thread has a per-console input we do not have. Nothing in this repo mentioned NVS before.

## Process note — a finding that was demoted on checking the notes

The sweeping agent independently decoded `0xde3a87c0` / `0xde3a88c0` as
`pci_get_max_read_req` / `pci_set_max_read_req` and was about to report the MRRS bump as its
headline. It then found that "One thing Orbis does that Linux does not" above already records it
**and** says it is already shipped via `pcie_set_readrq()`, and demoted its own finding. Recorded
because "check the notes before reporting a discovery" keeps paying, and because the reverse — an
agent re-deriving a known fact and filing it as new — is how a repo grows three copies of one
register.
