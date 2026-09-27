# Handoff: Kinfolk Mobile — Lens Rows (family tree viewer)

> Status: **work in progress.** Structure, logic and line semantics are settled. Motion "feel" is still being tuned, so every timing constant is listed below as a named, tweakable value.

## 1. Overview
A mobile family-tree viewer. Each **generation is a horizontal row**; rows stack vertically (oldest on top). The focused generation shows full person cards. Rows further away shrink through a **depth lens**: full cards, then small cards, then round initial avatars. Rows are **synced**: centring a person in one row re-centres their parents above and their children below, so a single straight vertical "spine" always runs through the family being followed.

## 2. About the design files
`Kinfolk Mobile - Lens Rows.dc.html` is an **HTML design reference**: a working prototype, not production code. Open it in a browser with `support.js` beside it. Rebuild it in the target stack (React Native / SwiftUI / Compose / web) using that codebase's patterns. All logic lives in the `<script data-dc-script>` block (`class Component`) and module-level helpers. **That source is the ground truth for every formula below.**

Fidelity: **high** (colours, type, geometry, line semantics). Family data is placeholder.

## 3. Test toggle — couple sizing (in build)
A small pill at the bottom-left ("Couples · unit" / "Couples · free", green dot = unit) switches live between the two sizing variants:
- **unit** (default): a person and their partners share one lens scale, the **max** of their individual scales. Couples stay whole even when cut off at the screen edge.
- **free**: every card or avatar is scaled by its own horizontal lens position.

Implement it as a feature flag (`coupleUnitSizing`) plus a debug-only toggle. It only changes the scale lookup (§6.4).

## 4. Architecture

```
TreeScreen
├─ RelationshipTrail   (top 66px, chip scroller: You › parent › … › focused)
├─ TreeCanvas          (402×714 reference, all people + SVG lines, gesture surface)
│   ├─ FocusBand       (follows focused row)
│   ├─ LinesLayer      (single SVG, z below people)
│   ├─ Row × N         (generation rows; items = person | ghost)
│   │    └─ PersonNode (card ⇄ avatar crossfade), BranchBadge
│   ├─ AddAffordances  (+ child / + parents)
│   ├─ GenerationRail  (year-span pills, left)
│   └─ Edge fades      (top 40px, bottom 110px → #F4F1EC)
├─ ZoomControl         (bottom-right, always open: − ••••• +)
├─ SizingToggle        (bottom-left, debug)
├─ Toast
└─ GrowSheet           (bottom sheet: add partner/child/sibling/parents)
```

**Render model.** One animation loop (rAF) advances all animated values, then a pure layout function computes every node's position, scale and opacity and every SVG path from state. Nothing is laid out by the platform's layout engine; everything is absolutely positioned from that function. Keep this split: **state + animator → layout(state) → draw**.

## 5. Data & relationship engine
- Person: `{ id, name, birth, death?, sex }`. Couples: list of pairs, plus a set marking exes. Families: `[[parentIds], [childIds]]`.
- Derived maps: `PARENTS`, `CHILDREN`, `SIBS` (same family), `PARTNERS`.
- `ancestry(id)`: BFS up through parents, giving `{ancestorId: depth}`.
- `blood(a, b)`: the common ancestor minimising `da+db` (tie → smaller `min(da,db)`).
- `bloodName(da, db)`: You / Parent / Grandparent / Great-grandparent / Child / Grandchild… / Sibling / Aunt-uncle / Niece-nephew / *N*th cousin *M*× removed.
- `relName(id)`: relation to the ego. Falls back to in-law: Sibling-in-law, Parent-in-law, or "*X* by marriage" via a partner who is blood.
- `relPath(target)`: ego → common ancestor → target chain (sibling hops collapsed, partner appended for in-laws). Drives the trail chips. `stepRel(a, b)` labels each hop (parent/child/sibling/partner/ex-partner).

## 6. Layout logic

### 6.1 Rows & items
- **Root** = top ancestor of the current branch. Rows below the root's generation are built recursively:
  - **Root row**: root + root's siblings (by birth), then an "Add sibling" ghost.
  - **Each lower row**: for every *core* person in the row above (in order), their children sorted by birth form a **group**.
- Each core person is followed by their **current partners**, then **ex-partners** (kind `mate`, `ex`). The root row also gets an "Add partner" ghost when the root has no partner.
- Horizontal item spacing (unscaled units, card width CW = 128):
  - same group: 22
  - core → partner: 6
  - partner → partner: 32
  - after the last partner: +30
  - between groups: 52.8 (22 × 2.4)
  - before a ghost partner: 14

### 6.2 Vertical lens (generation depth)
Parameters per zoom level: `z` base scale, `k` decay, `v` spacing, `cm` card/avatar threshold, `db` avatar boost.
- Row scale at generation distance `d`: `s = max(0.3·clamp(z/.6, .8, 1.1), z·e^(−k|d|))`.
- Row centre: `cy = 330 + 250·v·GD(d)`. GD is the integral of the scale with a linear tail once `s` hits its floor (`GD()` in source). Rows stay evenly readable at depth instead of collapsing.
- **Vertical fit**: the whole branch's extent (top of first row − 34, bottom of last + 24) is centred in the band y 36–628 if it fits, else clamped so the focused row stays visible.

### 6.3 Horizontal lens (per row)
- Row-space position `u` → screen `x = 201 + FE((u + rowOffset)·hc)`, where `hc = max(s, (avatarD + 9)/150)` (avatar rows pack tighter).
- `FE`: linear within ±110px of centre. Beyond that it compresses as `110 + 220·(a−110)/(a−110+220)`.
- Item scale `SC = s · q²`, where `q = 220/(a−110+220)` outside the linear zone.

### 6.4 Person scale & form
- **Unit sizing** (§3): `USC(item) = max(SC)` over the core and their mates. Free: `SC(item)`.
- **Card** when `sc ≥ cm`, **avatar** below it. They **crossfade** across `cm ± 0.045` using smoothstep; both render inside that band.
- Avatar diameter `D = clamp(sc·125, 20, 46)·db`.
- **Partner offset**: partners sit slightly lower: `+0.03·CH·sc` (cards) or `+0.07·D` (avatars). In avatar mode they tuck behind the spouse: centre offset = spouse radius + 0.24·D (0.30·D between two partners), with z-index below the spouse.
- Row half-height used for line anchors blends smoothly from avatar to card: `RHALF(s)`.

### 6.5 Line construction (SVG, round caps and joins; width × clamp(s, .45, 1.1))
- **Couple bond**: from the spouse's bottom-centre down to `yb`, then horizontal to the partner, then up into the partner. `yb` = partner bottom + max(8·s, 4). Solid green `#3D6B5A` 1.8px; exes dashed `5 4` and lower (+max(15·s, 8)).
- **Sibling bar**: horizontal at `rowTop − max(12·s, 5)` across a group's cores, with drops to each core. Two layers: a faint base `#CDBDA8` 1.6px always, plus a brown `#7A5A40` 2.4px overlay at opacity `a` (the group's activation, §7.3).
- **Parent → children spine**: only for activated groups. A straight vertical from the midpoint of the parents' couple bond (or the single parent's bottom) down to the children's sibling bar. If the spine x falls outside the bar, the bar extends horizontally to meet it. **Never elbows.**
- **Inactive group stub**: short vertical (max(12·s, 6)) up from the bar's middle at opacity `1−a`, meaning "has parents above".
- Placeholders (add sibling/partner/child/parents): dashed `#C7B79F` / `#B09A7E`.

### 6.6 Centring & sync (`propagate`)
- Selecting a person centres them in their row (`rowOffsetTarget = −item.x`).
- **Upward**: each row above centres on the parent of the row below's centred core. If a co-parent partner exists, it centres on the **couple midpoint**, so the spine is exactly vertical. A previous choice is kept if it's still a valid parent.
- **Downward**: each row below centres on the eldest child of the row above's centred core, or keeps the current one if it's still a child. If a row has no candidate, it and all rows beyond it are cleared.
- **Reroot** (partner branch badge, or a trail chip outside the branch): rebuild the rows from the new root, set everything instantly, fade the tree in (§7.2), and show the toast "Now showing the *Surname* family".

## 7. Motion & animation
All motion is **time-based exponential smoothing**, frame-rate independent: `value += (target − value) · (1 − e^(−dt/τ))`, with `dt` capped at 50ms. The loop runs only while something is moving.

### 7.1 Time constants (tune these)
| Channel | τ | Notes |
|---|---|---|
| Row horizontal offset | 140ms + 70ms × \|row − sourceRow\| | **Cascade**: the touched row settles first; rows further away follow, so changes ripple through generations |
| Zoom params (z, k, v, cm, db) | 170ms | all animate together |
| Generation focus (vertical) | 190ms | fractional `f` → snaps to an integer |
| Family activation `a` (lines) | 240ms | per group, 0↔1 |
| Hover amount | 90ms | mouse only |
| Branch reveal (after reroot) | linear 0→1 over 420ms, smoothstep applied to opacity | |

`sourceRow` = the row that was tapped, dragged or travelled to.

### 7.2 Fades (no hard pops)
- Card ⇄ avatar crossfade (§6.4). Badges follow their form's opacity.
- "+ child" / "+ parents" fade with row scale: `smoothstep((s − .45)/.12)`.
- Dates and relation pill inside cards fade in with `(sc − .6)/.18`. Name is always shown (full name above sc .7).
- Rerooted tree fades in from 0.

### 7.3 Family activation
Each sibling group has a target of 1 if it contains the row's centred core, else 0. The animated value `a` crossfades the brown overlay, the parent spine and the inactive stub. Changing the followed family therefore dissolves the old lineage and draws the new one.

### 7.4 Hover (pointer: mouse)
- Card: scale × (1 + .035h), border `rgba(180,85,47,.55)`, shadow `0 12px 26px rgba(35,31,27,.16)`, z-index +6.
- Avatar: scale × (1 + .1h), plus a `5px rgba(180,85,47,.42)` ring.
- Cleared as soon as a drag starts.

## 8. Gestures
- **Tap** (pointer moved < 8px): select and centre, then propagate.
- **Horizontal drag on a row**: that row follows the finger 1:1 (÷ row scale), with 0.35 rubber-band past the ends. The nearest person becomes centred live (propagating to other rows). On release it snaps to the nearest person, projected with velocity (clamp ±3 px/ms × 200 ÷ scale).
- **Vertical drag**: moves the fractional focus `f` (drag ÷ row pitch), allowing 0.35 of overscroll. On release it travels to `round(f − v·120/pitch)`.
- **Pinch**: distance ratio > 1.2 zooms in one level, < 0.83 zooms out one level (re-armed after each step).
- **Wheel / trackpad**: vertical = ±1 generation, horizontal or shift = ±1 person in the row under the cursor, ctrl/⌘ = zoom. Throttled 240ms / 260ms.
- The **direction lock** is decided after 6px of movement.

## 9. Zoom levels
| L | Name | z | k | v | cm | db | Shows |
|---|---|---|---|---|---|---|---|
| 1 | Whole tree | .42 | .46 | 1.9 | .46 | 1.4 | everything as larger avatars |
| 2 | Branches | .60 | .52 | 1.4 | .30 | 1.15 | focus row small cards, rest avatars |
| 3 | Family *(default)* | .80 | .60 | 1.06 | .30 | 1 | focus cards with relation; neighbours small cards |
| 4 | Close | 1.0 | .72 | 1 | .30 | 1 | big focus cards; neighbours name-only |
| 5 | Details | 1.15 | .42 | 1 | .30 | 1 | neighbours also show dates + relation |

## 10. Visual spec

**Card** (128×150 × sc, radius 22, padding 10×8, gap 7):
- Avatar disc: 54 → 66px as the card shrinks. Initials 600 at 0.36·size.
- Disc border: **gender colour** 2px, male `#56718E` / female `#B45E78`. Dashed if deceased (disc fill transparent), else `rgba(35,31,27,.06)`.
- Name: 600 Outfit, `max(14, 9.5/sc)`px.
- Dates: 400, `max(11, 8.5/sc)`px, `#6E665E`, "1931–2004" / "b. 1989".
- Relation pill: 600 uppercase, `max(8.5, 7/sc)`px.

Card states:
| State | Background | Border | Shadow |
|---|---|---|---|
| Default | `rgba(255,253,250,.9)` | 1px `rgba(35,31,27,.12)` | `0 4px 10px rgba(35,31,27,.07)` |
| Centred in row | `#FFFDFA` | 1.5px `rgba(35,31,27,.34)` | `0 8px 18px rgba(35,31,27,.1)` |
| Selected | `#FFFDFA` | 2px `#B4552F` | `0 16px 34px rgba(180,85,47,.22)`; pill `#B4552F` / white |
| Partner | `rgba(240,245,242,.95)` | 1.5px `rgba(61,107,90,.45)` (ex: dashed .7) | default |

Partner pill: `rgba(61,107,90,.13)` / `#2F5748`. Neutral pill: `rgba(35,31,27,.07)` / `#5E574F`.

**Avatar** (diameter D, initials 600 at .38·D):
- Fill: selected `#B4552F` with white text, partner `#D3E4DA`, ex `#E6EEE9`, deceased `#EDE7DE`, default `#FFFDFA`.
- Border: gender colour, 2px (2.5 when centred), dashed if deceased.
- Rings: always `0 0 0 2px #F4F1EC` (separator). Centred adds `3.5px #231F1B`. Selected adds `4px #B4552F` plus glow.

**Branch badge** (on partners): 34px circle.
- Partner has their own family: filled `#3D6B5A` with a light branch icon. Tap re-roots to their family.
- No family: dashed `#B09A7E` with a "+" and a brown icon. Tap opens "Add parents".
- Placement: card top-right corner; on avatars ≥ 24px, at 0.44·D scale.

**Chrome:**
- Trail chips: pill 8×12. Label 600 8.5px uppercase with .08em tracking, name 600 12px. Active chip `#B4552F`.
- Rail pills: 600 10px tabular numerals. Focused `#231F1B`.
- Zoom pill: bottom-right 16/20, 40px buttons, dots 6px (active 18×6 `#B4552F`).
- Toast: `#231F1B`, 10.5px, 2.4s.
- Sheet: radius 30, rows radius 19, Newsreader 20px title.
- Focus band: `rgba(255,253,250,.38)` with `rgba(180,85,47,.16)` hairlines. Its half-height tracks the focused row: `RHALF(s) + 26 + 22·s`.
- Page: `#F4F1EC` app. The outside-frame gradient `#EFEAE2 → #E2DBD0` is preview-only.

**Tokens:**
- Ink `#231F1B` · Muted `#6E665E` · Paper `#F4F1EC` · Card `#FFFDFA`
- Accent `#B4552F` / `#8E3F1F`
- Partner green `#3D6B5A` · Lineage brown `#7A5A40` · Faint `#CDBDA8`
- Gender: male `#56718E`, female `#B45E78`
- Fonts: Outfit 300–700, Newsreader 400/500

## 11. State
| Key | Meaning |
|---|---|
| `L` | zoom level 1–5, with animated `zc, kc, vc, cmc, dbc` |
| `f`, `fT`, `fg` | focus (fractional, target, integer) |
| `cen[g]` | centred id per row |
| `xs[g]`, `xsT[g]` | row offsets (current/target) |
| `srcG` | cascade source row |
| `root`, `rootG`, `ver` | branch root and item-cache version |
| `ra[g]` | reveal progress per row |
| `av[group]` | activation per group |
| `hv[id]`, `hov` | hover |
| `sheet`, `toast` | overlays |
| `unitOv` | sizing toggle override |

## 12. Open items (feel)
- Tune the τ values in §7.1 on device, especially the cascade step (70ms) and the zoom τ.
- Decide between unit and free sizing using the toggle.
- Watch avatar stacking at row edges in large families.

## Files
- `Kinfolk Mobile - Lens Rows.dc.html`: prototype (includes the sizing toggle)
- `support.js`: prototype runtime (reference only)
