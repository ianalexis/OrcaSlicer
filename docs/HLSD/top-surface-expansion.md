# Top surface expansion — High Level Design

## Purpose and scope

A feature rising from a top surface, such as raised text or a boss, is covered
by the layer above, so the top surface of the layer under it has a hole in its
footprint, and a feature that crosses the surface, such as a rib or a lattice,
cuts it into pieces. The top fill then breaks around every feature and the
feature's walls stand on sparse infill. `top_surface_expansion` grows the top
surfaces of each layer by a distance, so the top solid infill also covers the
material around and under those features. The holes close, the pieces join,
and the pattern runs uninterrupted, which concentric patterns need most. Walls
rising from the surface are anchored on solid infill.

The feature has two parts:

- `PrintObject::detect_surfaces_type()` grows the `stTop` surfaces of every
  layer region. Everything after it in the pipeline sees the grown area as top.
- `PerimeterGenerator` changes how `only_one_wall_top` treats the inner walls
  over a top surface when the expansion is on, so the expanded fill is not cut
  up by walls it would otherwise overlap.

## Settings

All three are region settings in Strength › Top/bottom shells, Advanced mode,
and can be set per object, part and modifier.

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `top_surface_expansion` | float, mm, ≥ 0 | 0 | Distance the top surfaces grow. 0 is off. |
| `top_surface_expansion_margin` | float, mm, 0–10 | 0 | Clearance kept between the added area and the walls. |
| `top_surface_expansion_direction` | enum | `inward_and_outward` | `inward`, `outward` or both. |

The defaults leave slicing unchanged, so profiles and 3MF projects without
these keys need no migration.

The GUI shows the three lines only when the profile has a top fill: top shell
layers above 0 and a top surface density above 0%. The margin and direction
are enabled only when the expansion is above 0.

## Where it runs

The expansion runs in `detect_surfaces_type()` at `posPrepareInfill`, right
after the layer's top is found (the region's slices minus the upper layer, with
membranes that are both top and bottom given to bottom), and before the later
steps process bridges and shells. It works on the region's slices, which include the
wall area. The result reaches the fill only inside the fill boundary the
perimeter generator left (`slices_to_fill_surfaces_clipped()`), so the walls
never change type.

The grown top is a real `stTop`. `discover_horizontal_shells()` propagates it
down `top_shell_layers`, so the solid shell under a feature's footprint also
becomes solid. Vertical shell detection also treats it as top. The cost is more solid infill on the top layer and in the shell under it.

The expansion only adds area: the original top is unioned back in, and bottom
surfaces are subtracted from the result, as bottom wins over top elsewhere in
that function.

It runs only when the expansion is above 0, the region has top shell layers
and a top surface density above 0%, and the layer has a top. With zero shell
layers `prepare_fill_surfaces()` retypes the top as internal, and at 0% the top
layer has no fill. In both cases there is nothing to grow.

## Growing an island

The region's slices are split into connected islands, and each island's top is
grown and clipped on its own. Each island has its own outer wall. Growing one
island's top across a gap into a neighbour, which may have no top at all,
would turn part of that layer into top fill and leave the rest internal. Doing
it per island also keeps the boolean operations proportional to the island:
the layer's tops are clipped to its bounding box first.

For each island:

1. **Infill region.** The island is inset by the wall band: the external
   perimeter width plus `wall_loops - 1` perimeter spacings, since walls are
   laid out on spacing. With `only_one_wall_top` the band is one wall, which is
   all that stays over a top surface (see below).
2. **Seed.** The seed is the layer's top within the infill region. If an
   opening by one top-infill line width removes all of it, meaning no part is
   at least two lines wide, the island is skipped. On a slope or near-vertical
   wall the exposed top lies within the walls or is a sliver, and growing it
   would flood the island's internal solid infill with top fill.
3. **Offset.** The seed grows by the distance with miter joins (limit 2), so a
   square feature leaves a square hole that closes evenly.
4. **Direction.** See below.
5. **Clip.** The result is clipped to the infill region inset by the margin.

The distance has no upper bound. A very large value fills the whole infill
region of every island that has a real top, less the margin.

### Direction

The direction filter uses the seed's filled outline (its contours with the
holes closed) to tell the two kinds of growth apart:

- **Inward** keeps the growth inside the outline: it closes the holes and
  gaps that the top encloses, such as the footprint of text, and leaves the
  outer edge where it was.
- **Outward** drops the growth that falls into those enclosed holes and keeps
  the growth past the outer edge. This joins pieces of a surface that a
  feature has divided, such as the cells of a lattice, since the gap between
  two pieces is a hole in neither of them, and grows the surface toward the
  walls.
- **Inward and Outward** keeps both, which is the default.

### Wall margin

An expanded top can reach walls that the original top did not touch. Top solid
infill shrinks as it cools and pulls on the outer wall it is bonded to, which
leaves a visible line on the outside at that layer. The margin insets the area
the expansion may add from the inner edge of the wall band, so the margin is
the real clearance between the added fill and the walls. Where the original
top already touched the walls it still does, because the margin limits only
the added area.

## Only one wall on top surfaces

`only_one_wall_top` keeps a single wall over a top surface. The perimeter
generator finds its own top: the island minus the upper layer, without bridges
and without parts narrower than `min_width_top_surface`. This top is not
expanded. Without the expansion, the inner walls are generated for the region
that is not top, so they follow the top's boundary. Around a feature they
leave a ring of inner walls on the surface, and they cut the surface into
pieces. An expanded top fill would overlap those walls or be split by them.

`top_fill_replaces_inner_walls()` is true when the region has top shell layers,
a top surface density above 0% and an expansion above 0. In that case the inner
walls over the top are left out and the top fill takes their place:

- **Classic.** The generator lays out the inner walls for the whole island, so
  they follow the real geometry, then drops each inner loop that has a vertex
  over the top. A closed loop can only be kept or dropped whole. A loop with
  less than two perimeter widths over the top is kept, and its band is taken out
  of the top fill. Gap fill over the top is removed. The band each dropped loop
  covered, minus the top, is added to the fill area as internal. With Ensure
  vertical shell thickness set to All that area becomes solid infill, and the
  expansion grows the top over it. The band is ±spacing/2 around the loop's
  centerline, so neighbouring bands and the spacing/2 fill boundary tile
  without overlap. Using ±width/2 added overlap under the kept walls and made
  no visible difference.
- **Arachne.** The generator lays out the inner walls for the whole region
  inside the outer wall, then cuts away the parts over the top. The cut is
  pulled back by half a width, because a bead's rounded end extends that far
  past its endpoint. Clipper treats each loop as an open polyline, so it also
  cuts the loop at its start point; pieces that share an endpoint are stitched
  back together. If a cut would remove less than two widths, the loop is kept
  whole and its footprint is taken out of the top fill. Pieces shorter than one
  width are discarded. The inner walls' `inset_idx` is renumbered to follow the
  single outer wall.

Otherwise, with the expansion at 0 or a density of 0%, the inner walls are
generated for the region that is not top, as before. Without the expansion,
the top fill stops at the unexpanded top's boundary and never takes over the
walls' space. At 0% there is no fill at all, and the walls are the only support
for the layers above. A project that uses `only_one_wall_top` without the
expansion slices as before.

Both one-wall options also need a shell to act on. `only_one_wall_top` needs
top shell layers and `only_one_wall_first_layer` needs bottom shell layers:
without them the surface is retyped as internal, and the single wall would ring
sparse infill. The generator ignores each option in that state, and the GUI
hides it, so a profile that left one enabled does not act behind a hidden
checkbox. A 0% top surface density still leaves a top surface, only an unfilled
one, so it does not turn `only_one_wall_top` off.

## Invalidation

| Change | Steps invalidated |
| --- | --- |
| `top_surface_expansion_margin`, `top_surface_expansion_direction` | `posPrepareInfill` |
| `top_surface_expansion` | `posPrepareInfill`; also `posPerimeters` when the value goes to or from 0 |
| `top_surface_density` | `posInfill`; also `posPerimeters` when the value goes to or from 0% |

Only a change to or from zero switches the wall reduction above, so other
changes of the expansion or the density keep the perimeters.

## Why a plain offset

The expansion is a plain offset of the top, limited by the island, the walls,
the margin and the direction. Approaches that do less have been tried:

- **Shrinking only the holes in a top surface** cannot join pieces that a
  feature divides.
- **Reducing each hole by a percentage of its area** needs a search for each
  hole's offset, and the distance it gives cannot be predicted.
- **Closing (grow, then shrink by the same distance)** or **keeping only the
  overlap of neighbouring tops' growth** joins only gaps narrower than twice
  the distance, and leaves dead-end gaps and the edges of larger features open.
- **Ignoring small features of the upper layer** works only for features
  under a size threshold.

With a plain offset the distance means what it says, and the direction and
margin cover the cases the other approaches handled separately.

## Constraints

- With the defaults nothing changes: both the expansion and the wall reduction
  need an expansion above 0.
- Bottom surfaces, including bridges, are never claimed.
- The wall reduction tests walls one vertex at a time. A wall segment that
  crosses the top with no vertex inside it is missed; this is rare, and
  testing it would mean clipping every wall.
- The offset does not follow the geometry. It covers any material within the
  distance, including under tall features, which is what anchors their walls.

## Verification

`tests/fff_print/test_perimeters.cpp` slices a step whose top ledge surrounds
a column that keeps rising, with both wall generators:

- *Top surface expansion only acts where there is a top fill*: with no top
  shell layers or a 0% density the expansion changes nothing, while a top fill
  does change the slice.
- *Top surface density does not affect a slice without a top shell.*
- *Only one wall on top surfaces drops inner walls only where a top fill
  replaces them*: the ledge layer loses wall length only with both a fill and
  an expansion, and otherwise lies between the plain slice and the one-wall
  slice.
- *Only one wall on the first layer needs a bottom shell.*
- *Only one wall on top surfaces keeps the inner walls of narrow walls away
  from the top surface*, with and without the expansion.
