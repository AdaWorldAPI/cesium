# Cesium as a Generic Renderer — Cornerstone (v1)

> **Status:** CORNERSTONE / architecture-framing doc. Design-spec; no code here.
> **Authored:** 2026-06-10. Branch: `claude/medcare-gaussian-splat-8z76jc`.
> **Public face:** *a generic OpenStreetMap 3D rendering improvement.*
> **What it actually is:** one rendering substrate that paints **OpenStreetMap
> geography**, **graph nodes-and-edges** (Neo4j / OWL knowledge graphs in 3D),
> and **Gaussian-splat volumes** — all through codecs and a shader **we wrote
> ourselves**, running on `fable`.

---

## 0. Provenance (read first — first-party, clean-room)

Everything below the Cesium streaming envelope is **our own work**, authored in
this workspace:

- **Our codec stack** — BGZ17 (golden-ratio recoverable sampling + Base17),
  PhiSpiral256 (residual-location atoms), CAM-PQ (semantic basin), PolarQuant
  (magnitude), Palette256 + HHTL (ADR-024 universal compression), and the
  `helix` placement/residue encoder. None of these are third-party.
- **Our Gaussian-splat pipeline** — the `Gaussian3D` carrier, the SoA
  `SplatBatch`, the CPU-SIMD splat math in `ndarray` (Cholesky / Mahalanobis /
  opacity-blend / SH-eval / SE(3)), and the splat-fit engines.
- **Our shader** — the **Markov Tile-Pyramid Perturbation Shader** (MTPPS,
  §4) — a stacked-pyramid LOD tile traversal with Markov cascade routing and
  per-Gaussian covariance perturbation, written against our own SoA stream.

We use **Cesium / OGC 3D Tiles** only as the open, standard *external streaming
envelope* (tileset.json, bounding volumes, geometric error, implicit tiling).
The intelligence inside each tile is ours. This is a **clean-room generic
OpenStreetMap rendering improvement** — it ingests open OSM data and renders it
better, on commodity CPUs, with no proprietary geospatial runtime involved.

---

## 1. The one-sentence thesis

**A tile is not a picture — it is a typed packet of geometry + meaning, and the
same packet can carry a city block, a knowledge-graph node, or a splat cloud.**

Cesium streams *where things are and how coarse they are*. Our stack streams
*what they mean, what shape they hold, how confident we are, and how to recover
the residual* — and paints all three payload kinds (geo / graph / splat) with
one shader.

---

## 2. Three payloads, one substrate

The generic claim is that **anything expressible as positioned primitives with
covariance** renders through the same pipeline. Three first-class payloads:

| Payload | Source | What a "primitive" is | How it becomes splats |
|---|---|---|---|
| **OpenStreetMap geography** | OSM PBF (open data) | Way / Node / Relation + tags | building footprint × DEM height → extruded prism → anisotropic Gaussians |
| **Graph (Neo4j / OWL)** | property graph / OWL+DOLCE ontology | node, edge, label, property | node → positioned Gaussian (size = degree, color = label); edge → swept Gaussian tube between endpoints |
| **Gaussian-splat volume** | splat-fit (e.g. ultrasound, photogrammetry) | fitted `Gaussian3D` | already splats — pass through |

The payloads diverge; **the substrate is singular**. All three:
1. land in a Lance/Arrow SoA keyed by a Cesium-TMS quadkey NiblePath,
2. compress through Palette256 + HHTL (ADR-024),
3. carry a `Gaussian3D` (or node/edge primitive that *fits to* `Gaussian3D`),
4. paint through the one shader (§4),
5. report a render-depth certificate (§5).

---

## 3. The two layers (external envelope + internal stream)

### 3.1 External envelope — Cesium / 3D Tiles (open standard, unmodified)

```
tileset.json
  root tile · boundingVolume · geometricError · refine: ADD|REPLACE
  content/contents · children / implicit tiling (subtrees)
```

Answers: *what exists, where, how coarse, whether content/children are
available, how to stream progressively.* This is the public, standards-based
contract — any OGC 3D Tiles client can consume our output.

### 3.2 Internal stream — our SoA (BindSpace4 lanes)

```
bind_source[]   tile/feature/node/edge/splat id · local coordinate
bind_state[]    3×3 covariance · transform · depth interval · graph degree
bind_signal[]   opacity · color · confidence · semantic score
bind_time[]     capture time · LOD phase · provenance epoch
bind_cert[]     certificate id · error bound · pass/fail · reason code
```

Answers: *what each tile/node/splat means, what shape it carries, what it
emits, when it is from, and which proof governs it.* This is ours; Cesium never
has to understand a single lane.

**The rule:** keep the 3×3 SPD covariance spine authoritative for spatial
Gaussians; the 4×4 carrier wraps transform + time + role + provenance +
certificate lanes around it. Do not collapse 3×3 into 4×4 unless the fourth lane
has a certified meaning.

---

## 4. The Markov Tile-Pyramid Perturbation Shader (MTPPS) — ours

The shader the user named. Three words, three mechanisms, all first-party:

### 4.1 "Tile-Pyramid" — the stacked LOD pyramid

The Cesium implicit-tiling subtree IS a quadtree/octree pyramid. We treat each
level as a stacked band:

```
L4 super-block   region / domain graph block        (coarsest)
L3 block-of-blocks  subtree / implicit-tiling group
L2 block         tile content group
L1 carrier       local splat / node / edge / mesh proxy   (finest)
```

A TMS quadkey prefix selects a pyramid sub-cone in O(1) — *"every primitive
inside this tile"* is one NiblePath prefix scan (the HHTL payoff).

### 4.2 "Markov" — cascade routing across the pyramid

Traversal down the pyramid is a Markov cascade (our HHTL cascade; the
substrate's Chapman-Kolmogorov guarantee, `I-SUBSTRATE-MARKOV`): a frontier
vector at level *k* times a transition table routes to the candidate tiles at
level *k+1*. ~95 % of pairs are skipped via the cascade. The hot path is a table
lookup + popcount + palette distance — never a dense matrix.

```
frontier_vector[route_key] × transition_table[route_key → child] → child scores
  → Skip | Attend | Compose | Escalate   (RouteAction)
```

### 4.3 "Perturbation" — per-Gaussian covariance shaping

At the leaf, the splat is painted by perturbing the base Gaussian: the 3×3
covariance is shaped by the payload (anisotropic Σ from PSF for geo/ultrasound;
degree-scaled isotropic Σ for graph nodes; swept Σ for edges), opacity from
amplitude/confidence, color/Doppler from SH coefficients (ℓ≤3). All math is the
CPU-SIMD `ndarray` splat kernels — no GPU vendor lock; WebGPU optional for the
paint step only.

**MTPPS in one line:** *walk the tile pyramid by Markov cascade, skip what the
certificate lets you skip, and paint the survivors as perturbed Gaussians — at
cache speed, on a CPU.*

---

## 5. Render-depth certificate (why a tile was painted)

Cesium screen-space error is extended with our proof lanes. Every tile/block
decision reports:

```
screen-space error + depth uncertainty + ordering uncertainty
+ occlusion confidence + covariance validity + query relevance
+ residual-location pressure (PhiSpiral256)
```

and emits a reason: *why skipped / why refined / why exact hydration required /
which certificate passed / which lane triggered it.* Certificates are not
telemetry — they change skip/refine/hydrate/paint behavior.

---

## 6. The graph payload, concretely (Neo4j / OWL as nodes + edges)

The generic capability that makes this more than a map renderer: a knowledge
graph renders in the same pyramid.

- **Nodes → Gaussians.** Each graph node (a Neo4j vertex, or an OWL/DOLCE class
  resolved through the family-leaf address) becomes one positioned Gaussian.
  Position by a force-directed or ontology-hierarchy layout; Σ-size by degree;
  color by label/class; opacity by confidence/truth-value.
- **Edges → swept Gaussian tubes.** Each edge (a Neo4j relationship, an OWL
  object-property triple) becomes a thin anisotropic Gaussian swept between
  endpoints; SH color encodes edge type.
- **Same identity, same NiblePath.** A node's 128-bit identity (`[SchemaPtr |
  NiblePath | shape_hash | family-leaf]`) IS its tile address — so *"render the
  sub-graph under this OWL class"* is the same prefix scan as *"render the OSM
  Ways in this tile."* Geo and graph are not two renderers; they are two prefix
  cones of one pyramid.

This is why the substrate is generic: **a city and an ontology are both
positioned-primitive-with-covariance fields**, and MTPPS does not care which.

---

## 7. The data model (durable skeleton)

```
Tileset
 └ Tile
   └ Content
     └ Feature / Node / Edge / Asset
       └ FieldBlock / SplatBlock / BindSpace4Block
         └ PhiSpiralAtom · CAM_PQ · PolarQuant · BGZ17Schedule · Palette256
           └ Certificate
             └ Decision
```

Hot path uses SoA sidecar tables (`tiles`, `contents`, `features`, `nodes`,
`edges`, `splat_blocks`, `bindspace4_lanes`, `phispiral_atoms`, `certificates`,
`decisions`) — never node-per-atom graph overhead.

---

## 8. First proof fixture (start tiny)

```
one tileset · one tile · one content payload
one OSM building → extruded Gaussian block          (geo payload)
one graph node + one edge → Gaussian + swept tube   (graph payload)
one synthetic splat block                           (splat payload)
one BindSpace4 L1–L4 stream
one Palette256 + HHTL codec pass (report ρ ≥ 0.99)
one PhiSpiral residual atom
one render-depth certificate + one decision report
```

Render all three payloads in the same viewport, proving the substrate is
payload-agnostic.

---

## 9. Calibration gates (measurable, not aesthetic)

```
same-or-better correctness · fewer exact replays · smaller leaf payload
lower candidate fanout · cache-shaped routing · stable spiral occupancy
low wrong-high-confidence rate · ρ-vs-reference ≥ 0.99 (Palette256/ADR-024)
```

---

## 10. Anti-patterns

- Do **not** make Cesium understand any BindSpace4 lane — the envelope stays
  standard 3D Tiles.
- Do **not** make PhiSpiral256 mean magnitude or semantics (it is residual
  *location* only).
- Do **not** replace 3×3 SPD covariance with 4×4 unless the fourth lane is
  certified.
- Do **not** let certificates become decorative — they must change behavior.
- Do **not** implement every payload before the §8 fixture proves the substrate.
- Do **not** conflate the geo and graph payloads — they are sibling prefix cones
  of one pyramid, not a merged schema.

---

## 11. What this enables

```
generic OpenStreetMap 3D rendering on commodity CPUs (the public face)
Neo4j / OWL knowledge graphs rendered as 3D node-edge fields
Gaussian-splat volumes (photogrammetry, ultrasound, scanned scenes)
certified 3D Tiles with internal proof lanes
query-aware streaming (paint the sub-graph / sub-region a query touches)
one shader (MTPPS) for all three — first-party, CPU-first, runs on fable
```

## Wall sentence

```
Cesium streams the world-tree; our SoA streams the meaning inside each branch;
MTPPS walks the pyramid by Markov cascade and paints geography, graphs, and
splats as one field of perturbed Gaussians — at cache speed, on a CPU, with fable.
```

---

_End of cornerstone v1. Companion design-specs (substrate detail):
`lance-graph/.claude/plans/cesium-osm-substrate-v1.md` (OSM ingest),
`lance-graph/.claude/plans/3DGS-Cesium-BindSpace4-headstone-exploration.md`
(the internal-stream synthesis), `lance-graph/.claude/plans/splat-native-ultrasound-v1.md`
(the splat carrier + CPU-SIMD math)._
