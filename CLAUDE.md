# CLAUDE.md — 3DGS Virtual Tour Platform

Photorealistic 3D Gaussian Splatting virtual tours. Standalone branded links first, iframe embeds second. Author/owner: Mattia. Primary operator: Davide (surveyor, capture + publishing).

Full spec: `docs/requisiti-tecnici.md`. Commercial and ownership terms: `docs/accordo-commerciale.md`.

---

## Rules that override your defaults

These are the ones you will get wrong if you follow the obvious path. Each has a reason.

### Never assume SharedArrayBuffer
Cross-origin isolation is impossible inside a third-party iframe — it needs the *parent* document isolated plus `allow="cross-origin-isolated"`, and no agency website will ever have that. **Default to `sharedMemoryForWorkers: false`.** Feature-detect and take the fast path only when genuinely available. Never write code whose correctness depends on SAB.

### Navigation is waypoint + free look, never free-roam
Snap-to-waypoint movement, unconstrained camera *rotation*, camera position clamped to a bounding volume derived from the capture trajectory. Off-path views of a splat look like floating soup — free-roam is the fastest way to make a good capture look broken. Do not "improve" this into free flight.

### Domain restriction uses CSP, not Referer
`Content-Security-Policy: frame-ancestors` — browser-enforced, unspoofable. **Never use the `Referer` header for access control**: it is trivially forged and legitimately stripped by `Referrer-Policy`, so it blocks real customers and stops nobody.

### The viewer never talks to the database
It reads one self-contained `config.json` plus assets. Nothing else. This is a contractual obligation, not a preference: on termination we owe Davide a static bundle that runs with no dashboard. Any coupling between viewer and API breaks that.

### No browser storage in the viewer
No `localStorage`, `sessionStorage`, IndexedDB or cookies. The viewer is stateless by design. State lives in `config.json` or in memory for the session.

### Never hardcode infrastructure identity
Cloudflare account IDs, R2 bucket names, domains and credentials all come from config/env. The accounts belong to Davide, not to us, and the platform must deploy into accounts it does not own.

### Analytics are cookieless and aggregate
No client-side identifiers, ever. A cookie dropped by our embed makes *our customer* non-compliant under ePrivacy on their own site.

### `.ply` is the master; delivery formats are disposable
Always keep the source `.ply`. Every `.ksplat` / `.spz` is a derived artifact behind a loader interface, regenerable by one command. SPZ is on the Khronos standards track and `.ksplat` is trending legacy — assume we will switch.

### Mobile is the gate, not a target
A change that is fast on a laptop and slow on a mid-range Android is a failed change. See budgets below.

---

## Hard budgets — treat as tests, not aspirations

- ≤ **60 MB** transferred, default mobile tier
- First meaningful frame **< 5 s** on 4G, mid-range Android
- **≥ 30 fps** sustained on a 3-year-old mid-range Android
- **No tab termination** on iOS Safari across a 5-minute session (Safari kills tabs on memory pressure with no catchable error)

Desktop may exceed these. Desktop is never the test case.

---

## Forbidden features

Do not build these, and do not suggest them:

- **Free-roam ruler / measurement tool.** Splats are not metric. Dimensions are authored by Davide, static, with stated tolerance, capture date and method, plus the disclaimer *"Misure indicative — non costituisce documento di rilievo"*.
- **Anything depending on SAB in an iframe.**
- Mesh or point-cloud export (that is Davide's separately priced survey deliverable).
- VR/headset support, native apps, real-time collaborative editing.
- Splat training or photogrammetry pipeline (runs on Davide's workstation).
- Google Street View publishing (needs 360° photospheres, not splats — separate capture deliverable).

---

## Current phase

**v1 has no backend.** Static site + local editor. Do not introduce a database, auth, or an API server until Gate 3 is explicitly reached.

- Local `editor.html` — loads a splat from disk, raycast hotspot placement, camera keyframes, waypoints, branding → exports `config.json`
- Assets and config in R2 (Davide's account, **EU jurisdiction hint set**)
- Static viewer on Cloudflare Pages
- Publishing is a commit

When the dashboard does arrive: **multi-tenant data model from the first line** (org → operator → client → tour). Retrofitting tenancy is a rewrite.

---

## Billing-critical fields

The royalty is per *active* tour by class. These exist in `config.json` from v1 even though nothing reads them yet — reconstructing them later is archaeology.

- `tour.class`: `listing` | `commercial`
- `tour.state`: `draft` | `published` | `archived` ("active" = published and publicly reachable)

---

## Conventions

- Every tour needs: poster image, Open Graph tags (must preview in WhatsApp), QR-able stable URL, and a fallback path (poster + pre-rendered flythrough video) for devices failing the tier check
- Respect `prefers-reduced-motion` — never autoplay the cinematic flythrough
- Fail loudly: never a blank canvas or a silently dead tab
- Comments and identifiers in English; user-facing copy in Italian

---

## When to stop and ask

- Any change that could affect the performance budgets → measure on the mid-range Android before and after
- Any proposal to add a backend, database, or auth before Gate 3
- Any feature on the forbidden list, however reasonable it sounds in context
- Anything touching dimension lines or measurement claims (legal exposure, not just technical)
