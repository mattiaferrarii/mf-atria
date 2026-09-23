# 3DGS Virtual Tour Platform — Technical Requirements

**Owner:** Mattia (author, platform)
**Primary operator:** Davide (capture, publishing)
**Companion document:** *Working Agreement & Commercial Model v1.0*

**Version:** 1.0 — 19 August 2026
**Status:** Draft for build. Priorities are binding; specifics are open to revision.

**Priority key:** **M** = Must (v1 cannot ship without it) · **S** = Should (needed to sell well) · **C** = Could (later) · **W** = Won't (explicitly out of scope)

---

## 1. Purpose and scope

A platform for publishing photorealistic 3D Gaussian Splatting virtual tours, delivered primarily as **standalone branded links** and secondarily as embeddable iframes.

**Scope:** viewer, tour editor, publishing pipeline, metering.
**Not in scope:** capture, splat training, photogrammetry processing. Those remain Davide's workflow on his own hardware; the platform consumes finished assets.

---

## 2. Product principles

These constrain every decision below. Where a requirement conflicts with a principle, the principle wins.

**P1 — Link first, iframe second.** Italian portals accept a *URL* in a "virtual tour esterno" field, published through the agency's CRM (Getrix, REALsmart, Gestionale Immobiliare). The standalone page is the product; the iframe is a convenience.

**P2 — Mobile is the gate, not a target.** Most property browsing happens on phones over cellular. A tour that fails on a mid-range Android has zero commercial value regardless of desktop quality.

**P3 — The viewer never talks to the database.** It consumes one self-contained JSON payload plus assets. This keeps the viewer fast, cacheable, embeddable, and — critically — **statically exportable** (see C4).

**P4 — Delivery format is disposable; the master is not.** Archive `.ply`. Every delivery artifact is a derived, re-generable file.

**P5 — Build the smallest thing that lets Davide sell.** v1 has no backend. Infrastructure sophistication is not the constraint; Davide's hours are.

**P6 — Proportionate security.** The assets are scans of publicly listed properties. Raise the bar against casual copying; do not gold-plate.

---

## 3. Hard constraints

Non-negotiable. Each has burned projects that ignored it.

### C1 — SharedArrayBuffer is unavailable in third-party iframes

`SharedArrayBuffer` requires cross-origin isolation, which for an iframe needs **all three** of: the top-level document itself cross-origin isolated (COOP `same-origin` + COEP `require-corp`), the parent granting `allow="cross-origin-isolated"`, and the iframe sending COEP. We control only the third. No agency's WordPress site will ever satisfy the first two.

**Consequence:**
- The viewer **must** function correctly with `sharedMemoryForWorkers: false` as the default path.
- On our own top-level domain we *may* set COOP/COEP and take the fast path when available.
- **Feature-detect and degrade. Never assume.**

### C2 — Mobile memory and bandwidth ceilings

iOS Safari terminates tabs under memory pressure with no catchable error. Practical published limits sit around **~1M gaussians on mobile** and ~3M on desktop for progressive formats. Raw captures of 5–15M splats at 200–500MB are not web-deliverable without optimisation.

### C3 — `Referer` is not a security primitive

Trivially spoofable, and legitimately stripped by `Referrer-Policy` on many sites — it will block real customers and stop no one. Use `Content-Security-Policy: frame-ancestors` (browser-enforced) plus short-TTL signed asset URLs.

### C4 — Static export is a legal obligation

The Working Agreement (§7) requires that on termination, every active tour is delivered as a static bundle that runs without the dashboard, and that live tours stay up for 12 months. **This must be an architectural property from day one, not a later migration.** It follows naturally from P3.

### C5 — Infrastructure lives in Davide's accounts

Per the Agreement §5, Cloudflare, R2, domains and DNS are opened and paid by Davide; Mattia holds operator access. The platform must therefore be **deployable into an account it does not own**: no hardcoded account IDs, bucket names, or credentials. This also unlocks the multi-operator licensing path.

---

## 4. Actors and surfaces

| Actor | Surface | v1 | Later |
|---|---|---|---|
| End viewer (buyer, guest, planner) | Public tour page / iframe | ✅ | ✅ |
| Davide (operator) | Editor + publishing | Local HTML tool | Hosted dashboard |
| Agency / venue (client) | Branding, analytics, own hotspots | ❌ | Self-serve portal |
| Mattia (platform owner) | Metering, deploys, monitoring | Manual | Admin console |
| Future operators (other regions) | Full tenant | ❌ | Multi-tenant |

---

## 5. Viewer requirements

### 5.1 Rendering and performance

| ID | Requirement | Pri |
|---|---|---|
| VW-01 | Render a 3DGS scene in-browser via WebGL2, with WebGPU used when available | **M** |
| VW-02 | Operate correctly without `SharedArrayBuffer`; feature-detect and select the sorting path at runtime | **M** |
| VW-03 | Progressive/streamed loading — show something meaningful before the full asset lands | **M** |
| VW-04 | Device-tier detection (memory, GPU, connection) selecting an appropriate LOD variant | **M** |
| VW-05 | Graceful fallback for devices that fail the tier check: poster image + pre-rendered flythrough video | **M** |
| VW-06 | Explicit failure handling — never a blank canvas or a silently dead tab | **M** |
| VW-07 | Multiple quality tiers per tour, generated at publish time | **S** |
| VW-08 | Pause rendering when the tab or iframe is not visible | **S** |

### 5.2 Navigation

| ID | Requirement | Pri |
|---|---|---|
| VW-10 | **Waypoint navigation with free look** — snap-to-point movement, unconstrained camera rotation | **M** |
| VW-11 | Camera position clamped to a bounding volume derived from the capture trajectory | **M** |
| VW-12 | Touch controls tuned for phones: single-finger look, tap-to-move, pinch zoom | **M** |
| VW-13 | Keyboard navigation and focus states | **S** |
| VW-14 | Optional bounded free-roam mode, per-tour, off by default | **C** |
| VW-15 | Minimap or floor indicator for multi-storey tours | **C** |

> **Rationale (VW-10/11):** Matterport's rail-based navigation exists because off-path views of a radiance field look like floating soup. Unconstrained free-roam is the single fastest way to make a beautiful capture look broken. This is a product decision, not a technical limitation.

### 5.3 Content features

| ID | Requirement | Pri |
|---|---|---|
| VW-20 | Hotspots: text, image, video, external link | **M** |
| VW-21 | Client branding — logo, accent colour, optional custom font | **M** |
| VW-22 | Cinematic flythrough from saved keyframes, with a Play control | **S** |
| VW-23 | Respect `prefers-reduced-motion`: never autoplay the flythrough; expose an explicit stop | **M** |
| VW-24 | Verified dimension lines — static, authored, non-interactive (see §10) | **S** |
| VW-25 | Multilingual hotspot and UI content, language selectable | **S** |
| VW-26 | Booking-engine deep links from hotspots (hospitality segment) | **S** |
| VW-27 | Capacity / layout overlays — seated vs standing configurations (venue segment) | **C** |
| VW-28 | Tour variants: seasonal, or construction-progress timeline with version comparison | **C** |

### 5.4 Distribution and sharing

| ID | Requirement | Pri |
|---|---|---|
| VW-30 | Standalone branded page at a stable, human-readable URL | **M** |
| VW-31 | Open Graph + Twitter Card tags with poster image — must preview correctly in WhatsApp | **M** |
| VW-32 | QR code generation for the tour URL (window displays, print) | **M** |
| VW-33 | Iframe embed mode with a generated snippet | **S** |
| VW-34 | `schema.org` structured data on the landing page for SEO | **S** |
| VW-35 | Optional lead-capture form with configurable destination | **C** |

### 5.5 Security

| ID | Requirement | Pri |
|---|---|---|
| VW-40 | `CSP: frame-ancestors` restricting embedding to allow-listed domains | **M** |
| VW-41 | Short-TTL signed URLs for asset delivery; no permanently guessable paths | **S** |
| VW-42 | Optional password protection per tour (pre-market listings, private venues) | **C** |
| VW-43 | ❌ Do **not** use the `Referer` header for access control | **W** |

---

## 6. Editor and dashboard requirements

### 6.1 v1 — local tool, no backend

| ID | Requirement | Pri |
|---|---|---|
| ED-01 | Standalone HTML file Davide opens locally; loads a splat from disk | **M** |
| ED-02 | Click-to-place hotspots via raycasting, with content editing | **M** |
| ED-03 | Record and reorder camera keyframes for the flythrough | **M** |
| ED-04 | Define waypoints and the navigation bounding volume | **M** |
| ED-05 | Set branding: logo upload, accent colour | **M** |
| ED-06 | Export a complete `config.json` | **M** |
| ED-07 | Live preview identical to the published viewer | **S** |

### 6.2 v2 — hosted dashboard (only after Gate 2)

| ID | Requirement | Pri |
|---|---|---|
| ED-10 | Authentication with per-operator accounts | **M** |
| ED-11 | **Multi-tenant data model from the first line of code** — org → operator → client → tour | **M** |
| ED-12 | Direct-to-R2 uploads via presigned URLs, bypassing the app server | **M** |
| ED-13 | Tour lifecycle states: draft → published → archived | **M** |
| ED-14 | Tour class assignment (listing / commercial) — drives metering | **M** |
| ED-15 | Embed snippet and QR generation | **S** |
| ED-16 | Tour versioning with rollback | **S** |
| ED-17 | Asset re-encode job — regenerate delivery formats from the archived `.ply` | **S** |
| ED-18 | Client-facing self-serve portal (branding, own hotspots, analytics) | **C** |

> **ED-11 rationale:** "a private portal for Davide" is cheap now and a rewrite later. Multi-tenancy is required by both the agency self-serve path and the other-region licensing path, which is the only genuine growth route in this project.

### 6.3 Analytics

| ID | Requirement | Pri |
|---|---|---|
| AN-01 | Per-tour: view count, unique visitors, average dwell time | **S** |
| AN-02 | Drop-off point and per-waypoint attention | **S** |
| AN-03 | Device and connection breakdown (also feeds performance tuning) | **S** |
| AN-04 | **Cookieless and aggregate** — no client-side identifiers | **M** |
| AN-05 | Monthly PDF/HTML report Davide can forward to the client | **C** |

> **AN-04 is a hard requirement, not a preference.** If the embedded viewer drops cookies on an agency's site without consent, we make *our customer* non-compliant under ePrivacy. That is a commercial disaster, not a technical detail.
>
> **AN-01/02 commercial note:** for an agent this is the report they forward to the seller to justify their commission. Often more persuasive than the tour itself, and it is the retention argument against churn.

---

## 7. Metering requirements

The royalty in the Agreement (§4.2) is per **active** tour, by class. Metering is therefore a first-class feature, not reporting.

| ID | Requirement | Pri |
|---|---|---|
| MT-01 | Authoritative count of active tours per operator, per class, per month | **M** |
| MT-02 | "Active" = published and publicly reachable. Archived = not counted | **M** |
| MT-03 | Immutable monthly snapshot retained for audit | **M** |
| MT-04 | Operator-visible dashboard showing current billable count — no surprises | **M** |
| MT-05 | Free-allowance logic (first 5 tours) and volume discount above 40 | **S** |

---

## 8. Data model

Minimum entities. Schema is Mattia's IP (Agreement §2.1).

```
Organisation ─┬─ Operator (user)
              └─ Client ──── Tour ──┬── TourVersion
                                    ├── Asset      (ply | ksplat | spz | poster | video)
                                    ├── Hotspot    (position, type, content, i18n)
                                    ├── Waypoint   (position, orientation, order)
                                    ├── CameraPath (keyframes, easing, duration)
                                    ├── Dimension  (endpoints, value, tolerance, date, method)
                                    ├── Branding   (logo, colours, font, domains)
                                    └── Analytics  (aggregate, cookieless)
```

**Tour** carries: `class` (listing | commercial), `state` (draft | published | archived), `published_at`, `expires_at`, `allowed_domains[]`, `default_locale`, `locales[]`.

---

## 9. Architecture

### 9.1 v1 (Gates 0–2)

```
Davide's machine          Cloudflare (Davide's account)
─────────────────         ─────────────────────────────
local editor.html   ──▶   R2 bucket (EU jurisdiction)
  ↓ exports                 /tours/{slug}/config.json
config.json + assets        /tours/{slug}/scene.ksplat
  ↓ sends to Mattia         /tours/{slug}/poster.jpg
  ↓                       Cloudflare Pages (static viewer)
manual publish        ──▶ tour.dominio.it/{slug}
```

No database. No API. No auth. Publishing is a commit.

### 9.2 v2 (after Gate 2)

```
Dashboard (React/Vue) ──▶ API (Node) ──▶ PostgreSQL (EU region)
                             │
                             ├──▶ presigned PUT ──▶ R2 (EU)
                             └──▶ generates ──▶ config.json (static, cached)
                                                     │
Public viewer (Vite + Three.js) ◀────────────────────┘
      └── reads config.json + assets only; never the DB (P3)
```

### 9.3 Stack

| Layer | Choice | Note |
|---|---|---|
| Viewer | Vanilla JS + Vite + Three.js + splat renderer | Evaluate World Labs **Spark**, PlayCanvas, Babylon.js v8 against mkkellogg on mobile before committing |
| Dashboard | React or Vue | Mattia's preference |
| API | Node + Express (or Hono on Workers) | Workers co-locates with R2 |
| DB | PostgreSQL, EU region | |
| Storage | Cloudflare R2, **EU jurisdiction hint set explicitly** | Zero egress |
| CDN | Cloudflare | |

> **Note on hosting region:** R2 serves from Cloudflare's edge, so "EU servers for speed" is not a real performance argument and should not appear in sales material. EU residency is required for **data protection** reasons (§11). The API returns small JSON; its latency is irrelevant.

### 9.4 Asset format strategy

| ID | Requirement | Pri |
|---|---|---|
| AS-01 | Archive the source `.ply` as the master, permanently | **M** |
| AS-02 | Treat every delivery format as a derived artifact, regenerable by one command | **M** |
| AS-03 | Ship `.ksplat` in v1 (mature tooling, progressive loading) | **M** |
| AS-04 | Plan migration to `.spz`; abstract the format behind a loader interface | **S** |
| AS-05 | Generate multiple LOD tiers at publish time | **S** |

> **Rationale:** Khronos published `KHR_gaussian_splatting` and `KHR_gaussian_splatting_compression_spz` in 2025, with the base extension at release-candidate status in early 2026. SPZ is the only format on a formal standards track and gets roughly 90% compression against raw PLY. `.ksplat` works today but is trending legacy. Because of AS-01/02, switching is a batch job rather than a migration.

---

## 10. Metrology requirements

Dimension lines are a *legal* feature as much as a technical one.

| ID | Requirement | Pri |
|---|---|---|
| MG-01 | Dimensions are **authored by Davide**, static, and not user-editable | **M** |
| MG-02 | Every dimension carries: value, **stated tolerance**, capture date, method | **M** |
| MG-03 | Persistent disclaimer: *"Misure indicative — non costituisce documento di rilievo"* | **M** |
| MG-04 | Scale registration recorded (LiDAR / control points), since a splat has no inherent metric scale | **M** |
| MG-05 | ❌ No free-roam ruler tool, ever | **W** |

> A splat is optimised for appearance, not metric accuracy. Measurements mean nothing unless the scene is registered against LiDAR or surveyed control points — which is precisely Davide's professional differentiator, and should be marketed as such. The word "verified" creates a duty of care; MG-02/03 are what make it defensible.

---

## 11. Privacy and compliance

| ID | Requirement | Pri |
|---|---|---|
| PR-01 | All storage and processing in the EU; R2 jurisdiction hint set | **M** |
| PR-02 | Cookieless, aggregate analytics only (see AN-04) | **M** |
| PR-03 | Documented takedown process — remove a tour within 24h on request | **M** |
| PR-04 | Configurable retention; automatic archival at contract end | **S** |
| PR-05 | Audit log of publish, unpublish, and delete actions | **S** |
| PR-06 | Support blur/removal requests on captured personal content | **S** |

**Process requirements sitting outside the platform** (Davide's responsibility, per Agreement §2.3): occupant consent before capture, a property clearance checklist, a DPA with each client, and drone-capture privacy handling for neighbouring properties.

> A 3DGS capture records mail on the counter, medication, documents and faces at high fidelity — and unlike a 2D photo, it cannot easily be blurred after the fact. Prevention at capture time is the only workable control. Budget the cleanup time.

---

## 12. Non-functional requirements

### 12.1 Performance budgets — these are gates

| ID | Budget | Pri |
|---|---|---|
| NF-01 | ≤ 60 MB transferred for the default mobile tier | **M** |
| NF-02 | First meaningful frame < 5 s on 4G, mid-range Android | **M** |
| NF-03 | ≥ 30 fps sustained on a 3-year-old mid-range Android | **M** |
| NF-04 | No tab termination on iOS Safari across a 5-minute session | **M** |
| NF-05 | Desktop tier may exceed the above; it is never the test case | **M** |

**Test device matrix (minimum):** one mid-range Android ~3 years old, one iPhone ~3 years old, one recent Android flagship, one laptop. Throttled to 4G. **Every release is tested on the mid-range Android before it ships.**

### 12.2 Other non-functionals

| ID | Requirement | Pri |
|---|---|---|
| NF-10 | Viewer availability ≥ 99.5% (static assets on CDN make this near-free) | **M** |
| NF-11 | Daily R2 backup with versioning enabled | **M** |
| NF-12 | Uptime and error-rate monitoring with alerting | **S** |
| NF-13 | Cloud spend alerting at defined thresholds | **S** |
| NF-14 | Staging environment separate from production | **S** |
| NF-15 | WCAG 2.1 AA where applicable: contrast, focus, keyboard, reduced motion | **S** |
| NF-16 | Dependency licence inventory maintained (Agreement §2.4) | **S** |

---

## 13. Explicitly out of scope

Say no to these in writing. Each has a reason.

| Item | Why not |
|---|---|
| Free-roam ruler tool | Liability; splats aren't metric (MG-05) |
| Any design depending on SharedArrayBuffer in iframes | Architecturally impossible (C1) |
| VR / headset support | No buyer in these segments asks for it |
| Mesh or point-cloud export from the platform | That's Davide's survey deliverable, priced separately |
| Real-time collaborative editing | One operator |
| Native mobile apps | The web viewer is the product |
| Splat training / photogrammetry pipeline | Davide's workstation, not the platform |
| Google Street View publishing | Requires 360° photospheres, not splats — a **separate capture deliverable**, not a platform feature |
| AI virtual staging | Buy or outsource if a client asks |
| Custom domain per client | Later, if ever |

---

## 14. Delivery plan and gates

Total budget before the first euro of revenue: **~100 hours**. If you reach 100 hours with no revenue, stop and reassess rather than drift.

### Gate 0 — Feasibility (~10 h, 1 week)
Take Davide's best `.ksplat`, host it statically, and open it on a mid-range Android over cellular **with SharedArrayBuffer disabled**.
**Pass:** meets NF-01/02/03. **Fail:** stop, or pivot to a desktop-only in-person presentation tool.
*In parallel, Davide runs 10 agency/venue conversations on price.*

### Gate 1 — Hero PoC (~30 h)
One beautiful tour, hardcoded config, standalone branded link with a fake client logo, OG tags, QR. No backend.
**Pass:** one signed paid pilot or written LOI. Not "they said wow."

### Gate 2 — v1 (~60 h)
Local editor (§6.1), R2 publishing, static viewer, waypoint navigation, hotspots, branding, fallback path, cookieless analytics.
**Pass:** 3+ paid tours after 20 real pitches. **Fail:** keep the PoC as portfolio, stop.

### Gate 3 — Dashboard (only if earned)
Trigger: 10+ tours/month **and** Davide spending >1 h/tour on manual handoff.
Build §6.2 with multi-tenancy and metering from the start.

### Gate 4 — Multi-operator
Trigger: Davide at capacity, or a second operator asking for a seat. Only then does this become a platform business.

---

## 15. Open technical questions

| # | Question | Blocks |
|---|---|---|
| 1 | Which renderer wins on mid-range Android — mkkellogg, Spark, PlayCanvas, Babylon? | Gate 0 |
| 2 | Real transferred size and frame time for Davide's typical capture at target quality? | Gate 0 |
| 3 | Can we hit NF-01 without visible quality loss, or does the pitch need adjusting? | Gate 1 |
| 4 | Splat training turnaround and hardware — does it cap throughput before the platform does? | Gate 2 |
| 5 | Is SPZ tooling mature enough to skip `.ksplat` entirely? | AS-03 |
| 6 | Do Immobiliare.it / Idealista / Casa.it impose constraints on external tour URLs? | Gate 1 |
| 7 | Workers + Hono vs Node/Express for the API? | Gate 3 |

---

## 16. Traceability to the commercial model

| Commercial feature | Requirement |
|---|---|
| Standard residential — link + QR + stills | VW-30, VW-32 |
| Luxury — white-label, cinematic, dimensions | VW-21, VW-22, VW-24, MG-01–04 |
| Agency retainer — analytics report | AN-01, AN-02, AN-05 |
| Hospitality — booking deep links, seasonal re-scan | VW-26, VW-28, ED-16 |
| Venues — capacity/layout overlays, measured plans | VW-27, MG-01–04 |
| Restaurants/wineries — menu hotspots, modules | VW-20, VW-28 |
| Heritage — multilingual, accessibility | VW-25, NF-15 |
| Construction — versioned timeline, comparison | VW-28, ED-16 |
| Royalty metering by tour class | MT-01–05, ED-14 |
| Termination — static export, 12-month continuity | C4, P3 |

---

*Requirements are Mattia's work product and part of the platform IP under Agreement §2.1.*
