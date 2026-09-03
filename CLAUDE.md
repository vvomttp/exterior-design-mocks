# exterior-design-mocks

Exterior-to-interior UX prototypes for CoStar — LoopNet (commercial) and
Homes.com (residential). These explore how someone moves from an aerial capture
of a site into an interior tour of one space inside it.

They are design artifacts, not production code. Read them as arguments about
interaction, and change them the way you would change a drawing.

## Ground rules

- **Single-file HTML, no build step.** Each prototype is one `index.html` with
  inline CSS and one inline `<script>`. Open it in a browser; that is the whole
  toolchain. Do not add bundlers, package files, or split into modules.
- **Three.js r128 from cdnjs**, loaded as a plain `<script src>`. `THREE` is a
  global. r128 predates a lot of the modern API — notably `CapsuleGeometry`
  (r142) does not exist. Check before reaching for anything recent.
- **Comments explain the reasoning, not the mechanics.** The existing comments
  say why a number is what it is and what went wrong before it got there. Match
  that. A comment that restates the line below it is noise.
- **No emoji, no decorative headers in code.**

## Layout

```
index.html              landing page — five cards, two-up grid
commercial/a/           multi-entry:  office campus, two towers, floors per tower
commercial/b/           single-entry: one building, one pin
homes/a/                multi-entry:  residential community, floors per home
homes/b/                single-entry: one listing, one pin
homes/a-hifi/           fidelity study — homes/a with better materials and geometry
PROTOTYPE-RATIONALE.md  STALE. Predates several rounds of changes. Verify before trusting.
```

`homes/a-hifi/` is a copy of `homes/a/` that diverged. Changes to shared systems
usually need to land in **both**, and the two files drift — always grep both
rather than assuming.

### The A / B distinction

**A (multi-entry)** — the capture contains several separately tourable spaces.
Pins mark assets inside a parcel. Picking one opens a floor list; picking a floor
sections the building and opens that level.

**B (single-entry)** — the capture has one tourable space. One pin, no floor
stack. Deliberately simpler: the argument for B is lower engineering effort.

## Architecture inside a prototype

Two scenes in one renderer, cross-faded rather than cut:

- `exterior` — the aerial capture. Orbit rig, POI pins, the section-cut system.
- `interior` — the tour. Walk / dollhouse / floorplan views.

`enterInterior()` fades out, swaps scenes under cover, and fades in. `savedAerial`
holds the aerial rig state so leaving the tour returns you to where you stood.

### Camera rig

One `rig` object holds both current and **goal** state (`gTheta`, `gPhi`,
`gTarget`, `gRadius`). Every camera change writes the goal; the animation loop
eases toward it with frame-rate-independent damping:

```js
const k = 1 - Math.pow(0.0016, dt);
```

So `flyTo(focus)` never moves the camera itself — it sets a destination. Write
new camera behaviour as a focus function returning `{target, theta, phi, radius}`.

`phi` is measured **off vertical**. Larger = closer to level. `1.30` is a
near-level elevation; `0.86` (~40° above the horizon) is the plan-reading tilt
used for a sectioned floor.

`nearestAngle(cur, goal)` takes the short way round so headings never spin.

### Screen-space framing

Focus functions size the shot from the field of view rather than magic distances.
The pattern: decide what fraction of the viewport the subject should occupy
(`MULTI_SHARE = 0.55`, `FLOOR_SHARE = 0.78`), solve for radius, then push the
look-at point along screen-right so the subject clears the info card:

```js
const off = (1 - SHARE) * radius * Math.tan(hfov*0.5);
target.x += Math.cos(theta) * off;
target.z -= Math.sin(theta) * off;
```

**Aim at the subject, not at `p.frame`.** `p.frame` is a framing anchor that sits
in front of the building; it was tuned when the floor camera was near-level,
where the difference is invisible. Tilted down, a point further from the camera
projects higher in frame, so aiming short throws the subject up past the top
edge. Use `g.getWorldPosition(c)`.

### Section cuts (the x-ray)

Selecting a building makes it solid and hoverable. Hovering a floor previews it
translucent. Committing to a floor **cuts** the building at that floor's ceiling:
no roof, nothing above.

Three pieces of state, deliberately separate:

- `ghosted` — which structure is active (independent of how it looks, so it stays
  raycastable while solid)
- `xrayOn` — translucent preview
- `sectionOn` — committed cut

`claimMaterials(g, on)` clones that structure's materials so it can be styled
without touching the shared palettes, and hands them back on release.
`restyle()` writes opacity and clipping onto the clones.

Clipping is **per-material** (`renderer.localClippingEnabled` +
`material.clippingPlanes`), not global, so only the selected structure is cut.
The plane keeps `y <= constant`:

```js
new THREE.Plane(new THREE.Vector3(0, -1, 0), constant)
```

`CUT_EPS = 0.08` drops the plane just under the ceiling. Without it, the top
floor lands the plane exactly on the wall cap, the eave and the roof base at
once — and a face coplanar with a clip plane flickers per pixel as its
interpolated distance rounds either side of zero.

Cut buildings get **floor plates** (visible only for the cut floor) and
**interior partitions**, or the section reveals an empty shell. Partitions stop
short of the slab above so their tops clear the plane. They have no door
openings; real circulation would need CSG.

`side = DoubleSide` while sectioned, so you see the inside faces of the walls.

Floor lists run **high to low** (Floor 2 above Floor 1) so moving down the list
moves down the building and hover highlighting doesn't invert.

### The ground stack

Site layers are near-coplanar and viewed from 200+ units away, which makes them
fight for the depth buffer — that was the road flicker. Three defences, all
needed:

1. `logarithmicDepthBuffer: true`
2. Named heights in `Y` spaced ~10x further apart than they look
   (`lawn .16`, `lot .34`, `road .52`, `stripe .60`, `walk .68`)
3. Explicit `polygonOffset` per decal material

`M.water` is deliberately excluded from the offset list — it is 3D geometry
(fountain bowls, pool volumes), and slope-scaled offset on curved surfaces makes
them punch through whatever contains them.

Anything new placed on the ground picks a `Y.*` height. Never eyeball it.

### The capture blob

The capture is not a rectangle. `CAPTURE_R` (285 homes / 330 commercial) is a
mean radius perturbed into an organic contour, mimicking a real splat boundary.
Outside it, flat Google-Earth-style context tiles fade out. On load the whole
thing reveals from the center outward.

`startReveal()` collects its targets **at call time**, not at definition time —
trees and cars are added after the block that defines it, and collecting early
made them pop in at full size.

## Brand

- LoopNet red `#C5202B` — commercial
- Homes.com orange `#FF7A10` — residential

POI pins are kite/arrow markers: neutral white at rest, brand color on hover and
when active. The label sits above the kite.

## Verification

Bash runs in a Linux VM and only sees folders mounted at session start. If the
repo is mounted, syntax-check with:

```bash
python3 - <<'EOF'
import re, pathlib, subprocess
for p in pathlib.Path('.').rglob('index.html'):
    for m in re.finditer(r'<script(?![^>]*\bsrc=)(?![^>]*importmap)[^>]*>(.*?)</script>', p.read_text(), re.S):
        f = pathlib.Path('/tmp/chk.mjs'); f.write_text(m.group(1))
        r = subprocess.run(['node','--check',str(f)], capture_output=True, text=True)
        print(p, 'PASS' if r.returncode == 0 else r.stderr)
EOF
```

If bash cannot see the repo, say so rather than claiming the work is verified.
Reading an edit back is not the same as parsing it.

There is no test suite and no linter. The check above plus opening the file in a
browser is the whole safety net, so keep edits surgical and grep for every call
site before renaming anything.

## Known open items

- `PROTOTYPE-RATIONALE.md` is stale — either sync it or delete it.
- Right-drag pan is referenced in the keyboard-shortcuts overlay but not
  implemented.
- `requestFullscreen()` is called without guarding for absence of the API.
- Interior partitions have no door openings.
