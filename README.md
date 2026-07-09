# Ship Cleanup (3D Print Prep)

A Blender add-on that cleans Star Citizen–style game-export ship models and turns them into watertight, manifold meshes ready for 3D printing.

Game exports are built for rendering, not printing: they arrive as loose "panel soup" with hundreds of overlapping shells, interior mechanics, cockpit props, landing gear, decals that are really floating zero-thickness planes, and non-manifold geometry everywhere. This add-on strips out what you don't want to print, keeps what you do, and produces a single sealed solid — while defaulting to a review-first workflow so nothing is deleted without you seeing it.

- **Blender:** 3.6 or newer
- **Location:** View3D → Sidebar (press `N`) → **Ship Cleanup** tab
- **Version:** 2.8.0

---

## Installation

1. In Blender, open **Edit → Preferences → Add-ons → Install…**
2. Select `ship_cleanup_addon_16.py`.
3. Enable the checkbox next to **Ship Cleanup (3D Print Prep)**.
4. Open the 3D viewport sidebar (`N`) and select the **Ship Cleanup** tab.

If you are upgrading from an earlier version, disable and remove the old one first — the internal class names are shared, so two copies cannot be enabled at once.

### Optional: Alpha Wrap support

One of the manifolding methods (Alpha Wrap) uses [`pymeshlab`](https://pypi.org/project/pymeshlab/). If you want it, use the **Install pymeshlab** button in the panel, or install it into Blender's bundled Python yourself. Every other method works without it.

---

## Quick start

1. Open your ship scene. Make sure the ship geometry is what you want in scope (see **Scope** below).
2. Optionally drop any meshes you want to protect into a **keep collection** (a collection whose name starts with `Keep`, `Not `, or `Protect`).
3. Click **Run Full Cleanup**. This runs the enabled initial steps in order and moves everything it flags into `REVIEW_*` collections.
4. **Review the `REVIEW_*` collections.** Anything misclassified can be dragged into a keep collection, and the classifier will leave it alone next time.
5. When you are happy, click **Make Manifold** to produce the final `ShipPrint` solid.
6. Optionally run **Delete All REVIEW_\* Contents** to permanently remove the flagged geometry once you no longer need it.

Nothing is destroyed until you either turn off Review mode or run the purge step. The original ship is also backed up into `REVIEW_original_backup` so Make Manifold can always fall back to a pristine copy.

---

## How it works

The pipeline separates **classification** (what is this mesh?) from **manifolding** (make it printable), with a review buffer in between.

### Scope and Review mode

- **Scope** — run on the **Whole Scene** (default) or only **Selected Objects**.
- **Review mode** (default on) — flagged objects are *moved* into `REVIEW_*` collections instead of being deleted. Turn it off only when you trust the results and want a one-shot destructive run.

### Classification rules

The classifier decides each mesh's fate using layered rules. Names lie in these exports (a mesh called `Body_Front_internalMechanics` can be exterior hull), so wherever possible the add-on trusts **geometry** over names.

- **Keep collections** — anything in a collection matching the keep regex is exempt from *every* rule and never touched (but still goes into the print). Checked first, before all else.
- **Landing gear** — matched by name (`landing gear`, `piston`, `lg_`, plus rigged `BONE_*/LG_*` parts like feet, struts, hinges, wheels). A **gear-exclusion regex** vetoes the match for bay **doors**, **hatches** and articulation **thrusters**, which are exterior hull.
- **Body decals** — islands whose faces use *only* decal-type materials (decal, livery, logo, marking, stencil, HUD, …). These are usually floating zero-thickness planes.
- **Internals** — cockpit-keyword names (seat, joystick, dashboard, screen, ejection, …), plus **library duplicates** (a component parked on the model centreline with copies placed on hardpoints — the centreline copy is removed), plus geometry that fails the **visibility test**.
- **External-hardware / access-feature exemptions** — exterior parts that *look* enclosed to a naive ray test (engine glow recessed in nozzles, gimbal turret parts, and doors/hatches/panels sunk in wells) are exempted from the geometric internal rules — but never when all their materials are interior-namespace, so a genuine interior door still classifies correctly.
- **Ordnance routing** — missiles and torpedoes are sorted by geometry into **externally mounted** (→ `REVIEW_external_missiles`) versus **enclosed in tubes** (→ internals). Tightly-packed rack missiles that shadow each other are handled by removing all ordnance from the occlusion test first.
- **Damage models** — meshes named or materialled as damage/destruction states.

#### The visibility test (inside-out + outside-in)

To tell exterior hull from buried internals, the add-on casts rays.

- **Stage 1 (inside-out):** rays leave each surface outward; enough escaping to open space means the mesh is exterior.
- **Stage 2 (outside-in):** for blocked samples, it probes from far outside back toward the surface and asks whether the *first thing an outside viewer hits* is this object. This is what lets flush doors at the bottom of recessed wells register as visible even though almost no outward ray escapes.

### Loose-geometry cleanup

Within each mesh, disconnected islands are deleted when they are: decal-material only, perfectly planar (zero-thickness fragments), **or** tiny open "micro-fragment" scraps (few verts, small extent, with boundary edges) that would otherwise mushroom into blobs under solidify. Closed solids are never removed by the micro rule. The largest island is protected by default.

### Manifolding

**Make Manifold** duplicates the in-scope meshes (applying modifiers), seals knife-edge seams, joins everything into one object called `ShipPrint`, then makes it printable via one of:

| Method | What it does | When to use |
|---|---|---|
| **Sealed Shells** *(default)* | Seals holes, thickens open shells to a printable wall, joins — no CSG. Slicers union the overlapping closed shells at slice time. Preserves every intake and detail. | Almost always. Fast and print-ready. |
| **Exact Boolean Union** | Sealed shells + an exact CSG self-union into one surface. Slow (minutes to hours). | Only when a downstream tool needs a single united surface. |
| **Alpha Wrap** | CGAL 3D Alpha Wrapping — guaranteed watertight 2-manifold enclosing surface. Needs `pymeshlab`. | When you want a guaranteed-manifold wrap. |
| **Voxel Remesh** | Native OpenVDB volumetric remesh. Robust but high-poly and rounds off detail. | Robust fallback. |

Wall thickness is derived from your **target print length**, **nozzle diameter**, and **minimum perimeters** (e.g. 0.4 mm nozzle × 2 = 0.8 mm walls) so thin panels survive at your intended print scale.

### Pre-manifold repair passes

- **Apply All Modifiers** — bakes every mesh's modifier stack into its data. All the geometric tests read raw mesh data, so this must happen before the loose pass and manifolding, or they measure geometry that isn't what prints.
- **Seal Knife-Edge Seams** — bridges hairline gaps between converging thin walls (matched boundary rims a couple of mm apart). Without it, solidify inflates each wall tip separately and the print grows a "mushroom" lip that doesn't match the model.
- **Mirror-Patch Import Holes** — finds hatches/doors whose mirror twin failed to import (leaving a hull hole), and duplicates + mirrors the surviving side across the ship's symmetry plane into a `MIRROR_PATCHES` collection. If the model has no usable symmetry, the hole is instead capped by Final Seal.
- **Final Seal** — a post-manifold sweep that removes the defect classes which survive sealing (doubled coincident faces, sliver flaps, floating triangles, over-linked edges), caps what remains, and iterates until the mesh verifies at **0 open and 0 over-linked edges**, reporting exact counts.

---

## The panel, box by box

### Steps

The initial-cleanup steps, each with a **checkbox** and a **button**.

- The **checkbox** selects whether **Run Full Cleanup** includes that step.
- The **button** always runs that step on demand, regardless of its checkbox.

Order (this is also the Run Full Cleanup order):

1. **Make All Local** — make linked/library data local and editable.
2. **Clear Empties** — remove empty objects (keeping transforms where needed).
3. **Delete Hidden / Non-Rendered** — drop hidden geometry (keep-collection objects are spared).
4. **Find & Remove Gear / Decals / Internals** — the classifier.
5. **Apply All Modifiers** — bake modifier stacks (added here in v2.8 so the loose pass sees real geometry).
6. **Clean Loose Geometry** — per-mesh island cleanup.

Below the steps, **Delete All REVIEW_\* Contents** permanently purges the review collections. It is destructive and deliberately has **no** checkbox — it is never part of Run Full Cleanup.

### Classifier

All the regexes and geometric thresholds that drive classification: gear / gear-exclusion / keep-collection / external-hardware / access-feature / ordnance / internal / interior-material / decal / damage patterns, plus enclosed-object and centreline-duplicate detection knobs.

### Loose Geometry

Toggles and thresholds for decal-island, planar-fragment, and micro-fragment removal, and whether to always keep the largest island.

### Symmetry Repair

**Mirror-Patch Import Holes.**

### Manifold for Print

Method selection, print scale / nozzle / wall settings, seam-sealing toggle and gap, and the buttons for **Apply All Modifiers**, **Seal Knife-Edge Seams**, **Make Manifold**, and **Seal & Verify Result** (standalone Final Seal). If Alpha Wrap is selected without `pymeshlab`, an **Install pymeshlab** button appears.

---

## Collections the add-on creates

| Collection | Contents |
|---|---|
| `REVIEW_landing_gear` | Flagged landing-gear meshes |
| `REVIEW_body_decals` | Flagged decal / livery planes |
| `REVIEW_internals` | Flagged interior mechanics, cockpit props, enclosed geometry |
| `REVIEW_external_missiles` | Externally mounted ordnance |
| `REVIEW_original_backup` | Untouched copy of the input ship (fallback source for Make Manifold) |
| `MIRROR_PATCHES` | Mirrored hatch/door patches (kept in the print) |
| `PRINT_OUTPUT` | The final joined `ShipPrint` solid |

Keep collections are **yours** — create a collection named starting with `Keep`, `Not `, or `Protect` (e.g. `Not landing gear`, `Not internal`) and drop rescued meshes into it. The purge step will not delete objects that live in a keep collection.

---

## Typical workflow

```
Run Full Cleanup
   └─ make local → clear empties → delete hidden
      → classify → apply modifiers → clean loose

Review REVIEW_* collections
   └─ drag any misclassified mesh into a "Not …" / "Keep" collection

(optional) Mirror-Patch Import Holes   # fix missing mirrored hatches
Make Manifold                          # produce ShipPrint
Seal & Verify Result                   # confirm 0 non-manifold edges
Delete All REVIEW_* Contents           # once you're happy
```

---

## Notes and caveats

- **Review first.** The classifier is good but not perfect on ships it hasn't seen. Always glance at `REVIEW_*` for a new model; the keep-collection override exists precisely for the edge cases.
- **Modifiers are baked.** Apply All Modifiers is destructive to the modifier stack (that's the point). Save first if you want the modifiers back.
- **`ShipPrint` mesh edits are direct.** Some manifold/seal operations write straight to mesh data and are not always undoable — save before Make Manifold if you want a safe restore point. The `REVIEW_original_backup` copy is the built-in safety net.
- **Zero-area degenerate faces** may remain after Final Seal; they are harmless to slicers, and removing them can re-open sealed edges, so they are left in place on purpose.
- **The defaults were tuned** against several real ships (KRIG P52 Merlin, Avenger Titan, RSI Polaris, AEGS Gladius). The regexes and thresholds are all exposed in the panel if your model needs different values.

---

## Version history (summary)

- **2.8** — Apply All Modifiers moved into the initial Steps box; each step now has a checkbox controlling whether Run Full Cleanup performs it.
- **2.7** — Two-stage (inside-out + outside-in) visibility test; stencil decals and micro-fragment loose removal; modifier baking before manifold; knife-edge seam sealing.
- **2.6** — Hull access-feature exemption (flush doors/hatches/panels in recessed wells).
- **2.5** — Gear exclusions for hatches/thrusters; external-hardware exemption; ordnance routing to `REVIEW_external_missiles`; keep-aware purge; Mirror-Patch Import Holes.
- **2.4** — Keep-collection system and gear-exclusion regex.
- **2.3** — Final Seal post-manifold defect sweep.
- **Earlier** — core classifier, loose-geometry cleanup, and manifolding pipeline derived from the Merlin analysis.
