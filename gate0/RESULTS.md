# Gate 0 — feasibility results

**Subject:** LCC_Ca_di_Bazzone_BeB (hospitality / B&B, §8.5 segment)
**Harness:** https://mf-atria.pages.dev  ·  **Renderer:** mkkellogg GaussianSplats3D 0.4.7 + three 0.185.1
**Date:** 2026-08-28

## Source data

Standard INRIA 3DGS PLY, **SH degree 0** (no `f_rest` properties), 17 float32 props = 68 B/splat.
Bounding box from PLY header: 81.6 × 75.8 × 26.2 m — large site, exterior included.
LOD pyramid supplied by Davide's pipeline (`source K1`, `dataType lcc2`) — exact halving.

| tier | splats | .ply | .ksplat (L1) | + JS gz | total |
|---|---|---|---|---|---|
| lod3 | 1,106,573 | 75.2 MB | 26.6 MB | 0.23 MB | **26.8 MB** |
| lod4 | 552,566 | 37.6 MB | 13.3 MB | 0.23 MB | **13.5 MB** |
| environment | 24,050 | 1.6 MB | 0.9 MB | — | (not loaded by harness) |

Conversion (regenerable, `AS-02`) — `convert.js`, mkkellogg `PlyLoader.loadFromFileData`,
`minimumAlpha=1, compressionLevel=1, SH=0`. Pruned nothing: output splat count == input.
lod3 converts in 1.3 s.

## Environment verified before measuring

- [x] `crossOriginIsolated` false — no COOP/COEP headers served by Pages
- [x] R2 buckets jurisdiction `eu` (invisible to default-jurisdiction listing)
- [x] masters bucket private, tours bucket public
- [x] CORS allows the Pages origin, `range` exposed, `Accept-Ranges: bytes` on assets
- [x] assets served `application/octet-stream`, `cache-control: immutable`

## Harness findings (desktop / headless, before the device test)

**1. Progressive loading is pathologically slow when the renderer is CPU-bound.**
Same asset, same page, only `progressiveLoad` differs:

| | progressive on | progressive off |
|---|---|---|
| viewer ready | 0.3 s | 1.4 s |
| splats built | 55% after **87 s** | **100% after 4 s** |

It starves its own build loop: every frame re-sorts and re-rasterises what is already
loaded, leaving no budget to build the rest. Measured under software rasterisation
(swiftshader), which is a worst case — but *mid-range Android is the gate*, and this is
exactly the shape of failure a weak device produces. **`VW-03` wants progressive load, so
this needs a verdict on real hardware, both ways.** The harness defaults to progressive
OFF and has a `prog:` toggle. This is open question #1's real content, not the renderer beauty contest.

**2. Never frame a camera on min/max of splat centres.** Stray splats (drone floaters,
distant landscape) inflate the bounding box several-fold and aim the camera at empty
space. Same trap as the PLY header comments, which claim 81.6 x 75.8 x 26.2 m while the
actual body of the scene is 18.9 x 28.1 x 7.8. The harness frames on p10..p90.

**3. three.js ESM is split.** `three.module.js` does `import from './three.core.js'`.
Vendoring only `three.module.js` gives a **blank page**: Pages answers the missing file
with its 200 + `text/html` fallback, so the browser rejects it on MIME type and no
404 or request-failure ever surfaces. Watch for this whenever vendoring three.

**4. A module script cannot report its own load failure.** The original error overlay was
registered inside the module that failed, so the page died silently -- the exact thing
`VW-06` forbids. There is now a classic-script guard with an 8 s boot watchdog.

**5. Cross-origin resource timing reports `transferSize: 0`** without `Timing-Allow-Origin`,
which R2 does not send and wrangler cannot set. Splat bytes come from `Content-Length`
instead (exposed via the CORS policy). Any byte budget measured through resource timing
alone will silently read zero.

## Measurements — mid-range Android, cellular, `fresh: on`, cooled between runs

### Run 1 (2026-08-28) — native DPR, cache-contaminated

| (run 1 — native DPR, cache-contaminated) | lod4 (13 MB) | lod3 (27 MB) |
|---|---|---|
| device / OS | Realme GT master edition | Realme GT master edition |
| `crossOriginIsolated` (must be false) | false | false |
| SharedArrayBuffer (expect absent) | Absent | Absent |
| page transferred — `NF-01` ≤ 60 MB | 13.6 MB | 26.6 MB |
| first frame — `NF-02` < 5 s | 0.47 s | 0.34 s |
| fps min / avg — `NF-03` ≥ 30 | 10 / 25 | 9 / 18 |
| frames under 30 | 5432 growing each second | 4156 growing each second |
| visual quality (subjective) | Very low (plus device hot) | Very low (plus device hot) |


### What to record per run

Copy these six rows straight off the HUD, plus one subjective note:

`fps min / avg` · `frames under 30` · `first splats drawn` · `splat download` ·
`page transferred` · `splats built` (must read 100%) · sharpness (your words)

**How to run one:** open the URL, wait until `splats built` hits 100%, then orbit slowly
for ~60 s and read the numbers. fps min/avg auto-resets when loading finishes, so what you
read is genuinely sustained. Let the phone cool before the next run.

Screen is 1080x2400, CSS viewport ~393x873, so DPR sets the pixel load:

| dpr | render pixels | vs native |
|---|---|---|
| 1 | 0.34 MP | 7.6x fewer |
| 1.5 | 0.77 MP | 3.4x fewer |
| 2 | 1.37 MP | 1.9x fewer |
| auto = 2.75 | 2.59 MP | baseline (10/25 fps) |

### Run 2 (2026-08-31) — Davide, Realme GT Master Edition 6 GB, Android 13, Very **5G**

| # | tier | dpr | fps min/avg | <30 | 1st frame | built | sharp 1-5 |
|---|---|---|---|---|---|---|---|
| 1 | lod4 | 1 | **46 / 57** | 25 | 3.88 s | 100% | 2 |
| 2 | lod4 | 1.5 | **36 / 46** | 62 | 2.41 s | 100% | 2 |
| 3 | lod4 | 2 | 22 / 37 | 385 | 3.30 s | 100% | 2 |
| 4 | lod4 | auto 2.75 | 11 / 28 | 921 | 3.43 s | 100% | 2 |
| 5 | lod5 | 1 | **54 / 60** | 5 | 1.43 s | 100% | 1 |
| 6 | lod3 | 1 | 23 / 38 | 524 | 5.21 s | 100% | 2 |
| 7 | lod2 | 1 | 11 / 25 | 724 | 9.61 s | 100% | 3 |
| 8 | lod0 | 1 | crashed — froze after ~230 MB, 3 attempts | | | | |

`splat download` omitted: that column was measuring the harness's own HEAD probe, which shared
the splat's URL so the resource-timing lookup matched it instead of the real GET. Fixed (probe
removed, sizes hardcoded). `first splats drawn` is independent and remains valid.

### Cost model — **there are two ceilings, not one**

Converting min fps to frame time exposes both curves:

| dpr (lod4) | megapixels | ms/frame | | tier (dpr 1) | splats | ms/frame |
|---|---|---|---|---|---|---|
| 1 | 0.34 | 21.7 | | lod5 | 276k | 18.5 |
| 1.5 | 0.77 | 27.8 | | lod4 | 553k | 21.7 |
| 2 | 1.37 | 45.5 | | lod3 | 1.11M | 43.5 |
| 2.75 | 2.59 | 90.9 | | lod2 | 2.22M | 90.9 |

- **Pixels matter, sub-linearly.** 4x the pixels roughly halves fps. Confirms finding #1.
- **Splats matter, linearly, above a knee at ~550k.** 276k -> 553k costs only 15%, but every
  doubling past that doubles frame time exactly. Below the knee a fixed ~15-18 ms floor dominates.
- So **finding #1 was half right.** At native DPR pixels dominate; once DPR is capped, splat
  count takes over. Both budgets must be satisfied at once.

### Verdict against the gates

| | result |
|---|---|
| `NF-01` ≤ 60 MB | **PASS** — 13.3 MB at lod4 |
| `NF-02` < 5 s first frame | **NOT VERIFIED** — measured on **5G**, not 4G. At a realistic 4G 25 Mbps, lod4's 13.3 MB is ~4.3 s of download alone, so first frame lands ~5-6 s and this likely **fails**. Must be redone on real 4G |
| `NF-03` >= 30 fps sustained | **PASS at lod4 / dpr 1.5** (36/46) and lod4 / dpr 1 (46/57). Fails at lod3, lod2, and any dpr >= 2 |
| `NF-04` no iOS tab kill | PASS on iPhone 17 Pro Max (flagship — necessary, not sufficient) |

**Best shippable combination: lod4 @ dpr 1.5** — 13.3 MB, 36/46 fps, 2.41 s first frame on 5G.

### The bind — this, not frame rate, is the real Gate 0 problem

Sharpness was rated **2/5 at lod4 across all four DPR values**. Resolution did not move it, so
perceived quality is limited by **splat density**, not pixels. But density is the term that costs
frame time linearly:

| tier | sharpness | fps min | verdict |
|---|---|---|---|
| lod5 | 1 | 54 | fast, looks bad |
| lod4 | 2 | 46 | **fast enough, still looks poor** |
| lod3 | 2 | 23 | too slow, no sharper |
| lod2 | 3 | 11 | only acceptable-looking tier, unusable ("difficile da navigare, laggoso") |

**The configuration that performs is not the configuration that sells.** Gate 0 passes on
numbers and fails on the thing a client actually looks at.

lod0 crashed (8.9M splats on a 6 GB phone — consistent with `C2`), so **the quality ceiling is
still unknown**. That is now the most important open question: if the full-quality model is also
only ~3/5, the limit is the capture or SH0, and no tier tuning fixes it.

### The density law — lod0 desktop test (2026-09-03) reframes everything

Full-quality lod0 on desktop: **good sharpness, very laggy**. So the capture is fine and the
quality ceiling is real — `2/5` was never a capture defect, only what decimation costs.

Sharpness tracks **splats per m²**. Bazzone's dense core (p1..p99) is 18.9 x 28.1 m = **531 m²**,
so the density that read as sharp is `8,893,769 / 531` = **~16,750 splats/m²**. Hold that constant:

| subject | m² | splats | ksplat MB | est. fps (dpr 1) | |
|---|---|---|---|---|---|
| single room | 25 | 419k | 10.1 | 46 | passes everything |
| small office | 40 | 670k | 16.1 | 38 | passes |
| 2-room apartment | 60 | 1.0M | 24.2 | 25 | marginal |
| standard listing (§8.2) | 120 | 2.0M | 48.3 | 13 | fails |
| Ca' di Bazzone core | 531 | 8.9M | 213.9 | 3 | fails badly |

fps extrapolated from the measured linear regime above ~550k splats (553k->46, 1.11M->23, 2.22M->11).

**Full quality is affordable up to ~40-50 m² per scene.** Ca' di Bazzone was never a fair test:
it asked the phone to draw a 531 m² site as one object.

### Consequence: tours must be per-zone, not per-property

Every segment the commercial model values is larger than 50 m² — §8.2 is "up to ~120 m²",
§8.5 is "reception, common areas, 3-5 room types", §8.6 is "full venue indoor + outdoor".
One scene per property caps out below the cheapest sellable segment.

Per-zone scene loading is **not a workaround** — `VW-10` already mandates waypoint navigation,
so the viewer always knows which space the visitor occupies. Load that zone, swap on transition;
§8.5 literally describes hospitality as discrete room types. At 25-50 m² per zone: 10-16 MB and
38-46 fps **at lod0 sharpness**. It also improves `NF-01`/`NF-02`, since a session downloads one
zone rather than an estate.

**Open process question for Davide, and it is the important one:** can his pipeline split a
capture into zones as a post-process, or does per-zone require capturing room-by-room? If the
latter, it changes his workflow and his pricing before he quotes anyone.

### Next, in order

1. ~~`gpuAcceleratedSort`~~ — **TESTED, DEAD END.** Renders a **black screen** on real GPUs
   (desktop and the Realme), while splats still build normally. Not a software-rendering artifact
   and not our bug: `postMessage` carries no transfer list, so the buffer is not detached; the
   failure is the readback race WebGL warns about — *"READ-usage buffer was written, then fenced,
   but written again before being read back"*. `computeDistancesOnGPU` overwrites the distances
   buffer before the prior read completes, the worker sorts on stale data, every splat resolves to
   the same depth, nothing draws. A library-internal race in 0.4.7, not patchable from a harness.
   **The fastest lever is gone; the tier-vs-sharpness bind stands.**
2. **Confirm the density law.** Ask Davide for a small single space (~25-50 m²) at normal capture
   quality, as `.ply` **with the LOD pyramid**. Prediction to test: ~400-700k splats reads 4-5/5
   *and* holds 38-46 fps. Also request the `Ufficio_k2` master — we hold only its `.ksplat`, so it
   cannot be decimated to test this.
3. **The renderer bake-off this plan called for and never ran.** §3 listed mkkellogg vs Spark vs
   PlayCanvas as open question #1. Only mkkellogg was ever built. Two rounds have now been spent
   tuning one renderer rather than testing whether another sorts better without SAB — PlayCanvas
   SOGS and Spark both warrant a harness before Gate 0 is called either way.
4. **lod0 on a desktop GPU** — the missing quality reference. Cannot run on the phone.
5. **Redo `NF-02` on real 4G** (Davide, one run) — 5G and the SIM-backed router both flatter it.
6. Only then call Gate 0.

### Run 3 (2026-09-04) — Davide, real **4G+** (Very, signal 3/5), gpu sort off

He ran the older four-run sheet with `gpu` switched off manually, so runs 2-3 are bonus data
and runs 1 & 4 repeat the same config — a free reproducibility check.

| # | tier | dpr | fps min/avg | <30 | 1st frame | built | sharp |
|---|---|---|---|---|---|---|---|
| 1 | lod4 | 1.5 | 35 / 48 | 55 | **2.92 s** | 100% | 1 |
| 2 | lod3 | 1 | 20 / 28 | 691 | 6.97 s | 100% | 1 |
| 3 | lod2 | 1 | 13 / 21 | 1373 | 10.35 s | 100% | 2 |
| 4 | lod4 | 1.5 | 35 / 50 | 43 | **3.81 s** | 100% | 1 |

**Reproducibility is excellent**: identical config gave 35/48 and 35/50 fps. First frame varied
2.92 -> 3.81 s, i.e. fps is device-bound and stable while first-frame tracks the network.

**`NF-02` answered on real 4G+**: lod4 **passes** at 2.9-3.8 s. lod3 (6.97 s) and lod2 (10.35 s)
**fail**. The earlier worry that even lod4 would fail on 4G was wrong.

**Sharpness recalibrated downward by one point across every tier** (lod4 2->1, lod3 2->1,
lod2 3->2), same person and device. Treat the absolute values as non-comparable between sessions;
the *ordering* is stable and that is what carries information.

## GATE 0 VERDICT — technically PASS

At **lod4 @ dpr 1.5**, on a 2021 mid-range Android over real 4G+:

| gate | budget | measured | |
|---|---|---|---|
| `NF-01` | <= 60 MB | 13.3 MB | PASS |
| `NF-02` | < 5 s first frame on 4G | 2.92 / 3.81 s | PASS |
| `NF-03` | >= 30 fps sustained | 35 min / 48 avg | PASS |
| `NF-04` | no iOS tab kill in 5 min | survived | PASS |

Hotspot placement (`ED-02`, the unlisted blocker from finding #5) is also solved: the capture
ships a `collision.ply` mesh, so raycasting is ordinary Three.js against invisible geometry.

By `requisiti §14`'s own criteria — "meets `NF-01/02/03`" — **Gate 0 passes.**

### ...and the criteria were measuring the wrong thing

That passing tour was rated **1/5** for sharpness. `§14` sets no quality bar on Gate 0, so a
technically-conformant but unsellable tour clears it. The quality bar lives at Gate 1
("one beautiful tour"), which is precisely what is now at risk.

**The useful output of Gate 0 is therefore not pass/fail but a proven budget envelope:**

> **~550k splats, dpr 1.5, 13.3 MB** clears all four gates on a 2021 mid-range phone over 4G.

Combined with the density that read as sharp (~16,750 splats/m², from the lod0 desktop test):

| quality bar | density | **zone size within budget** |
|---|---|---|
| 4-5/5, lod0-like | 16,746 splats/m² | **33 m²** |
| ~2/5, lod2-like | 4,173 splats/m² | 132 m² |

**~33 m² per zone at good sharpness, everything green.** That is a bedroom, an office, a small
dining room — and a B&B or a hotel is a collection of exactly those. It is the quantitative case
for the per-zone architecture.

Both numbers rest on one subjective judgement of one capture, and on the assumption that
sharpness depends only on splats/m². **The pending small-space scan tests both**, and is the
last thing standing between here and Gate 1.

## iOS Safari — `NF-04`

| | |
|---|---|
| device / iOS version | iPhone 17 Pro Max / iOS 26.6.1 |
| caveat | flagship, not a memory-pressure test — `NF-04` is about weak devices; passing here is necessary, not sufficient |
| tier tested | lod3 |
| survived 5 min without tab termination | Yes |

## Verdict

- [ ] **PASS** — `NF-01/02/03` met on the mid-range Android at some tier
- [ ] **FAIL** — stop, or pivot to desktop-only presentation tool

Which tier becomes the mobile default: ______
Open question #1 (renderer): ______
Open question #2 (real size/frame time for Davide's capture): answered above for this subject.
