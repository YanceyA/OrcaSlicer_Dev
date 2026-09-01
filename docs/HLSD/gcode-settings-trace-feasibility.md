# G-code Settings Trace — Feasibility Analysis

Date: 2026-09-01
Branch: `claude/orcaslicer-gcode-settings-trace-tz4vri`
Code references are against commit `e8115658`. Line numbers drift; function names are the durable anchor.

## 1. Problem and agreed scope

After slicing, the preview shows *what* each toolpath does (speed, flow, fan, width) but not *why*.
A user who sees a segment printing slower, wider, or with more fan than expected has no way to
discover which setting caused it, so they cannot tell which setting to change.

The minimum viable feature: for a printed line segment, show the chain of settings that moved each
attribute away from its base value.

Decisions already taken for this exploration:

| Question | Decision |
|---|---|
| Exhaustive list vs decision trace | **Decision trace**: only the steps that changed the value. |
| Attributes, tier 1 | Line width, speed, flow, fan, acceleration, temperature. |
| Attributes, tier 2 (later) | Seam, z-hop, pressure advance, layer height, jerk. |
| UI | Hover-in-3D is the eventual goal but out of scope. A panel bound to the existing preview slider position is the target. |
| Setting provenance | Name the source (global preset, object, modifier, layer range, paint). Level of detail to be tuned by trying it. |
| Geometry-driven causes | Include the measured value that triggered a rule where cheap. |
| Granularity | Per G1 line. Grouping unchanged runs is a later deliverable. |
| Gating | Strictly opt-in. |
| Deliverable | This analysis. No prototype. |

## 2. Verdict

**Feasible, moderate effort, low risk to existing behaviour when disabled.** Every decision that
changes one of the six tier-1 attributes was located and is instrumentable. The recommended
architecture reuses the mechanism OrcaSlicer already uses to move information from the G-code
generator to the preview: comment tags in the G-code stream, parsed by `GCodeProcessor` into
per-move data, displayed by the sequential-view widgets in `GCodeViewer`.

Three findings shape the design more than any other:

1. **Decisions are spread over three pipeline stages that cannot see each other.** Path generation
   sets roles, widths and flow. `GCode::_extrude` sets speed, acceleration, jerk and applies flow
   ratios. Post-processors that operate on raw text (`CoolingBuffer`, `PressureEqualizer`,
   `FanMover`) decide the final feedrate and the fan speed. The fan value in effect for a G1 line is
   never known by the code that emits that line.
2. **G1 line identity is not preserved through post-processing.** `PressureEqualizer` and
   `FanMover` both split one G1 into several. `CoolingBuffer` deletes some `G1 F` lines. A trace
   keyed on line number assigned early will break. A trace carried as sticky comment-line state
   (the same model as `;TYPE:` and `;WIDTH:`) survives all three, verified by reading each stage's
   comment handling.
3. **Setting provenance is recoverable in memory but is absent from the exported file.** The
   `PrintObjectRegions` structure that survives slicing keeps the volume, modifier chain, layer range
   and paint links for every region. `GCode::m_config` is a flattened copy that has lost them, and
   the config block written into the G-code is the global preset only.

The rest of this document gives the evidence, the design options, a recommended phasing, and the
decisions that need a human answer before a prototype starts.

## 3. What exists today

### 3.1 Per-move data in the preview

`GCodeProcessor` parses the generated file into one `MoveVertex` per G-code line that moves
(`src/libslic3r/GCode/GCodeProcessor.hpp:215-249`). Each vertex already carries: role, extruder,
position, ΔE, feedrate, actual feedrate, width, height, mm³/mm, fan speed, temperature, pressure
advance, acceleration, jerk, time, layer id, object label id and the **source line number**
(`gcode_id`). These are converted into `libvgcode::PathVertex` in
`src/slic3r/GUI/LibVGCode/LibVGCodeWrapper.cpp:191` and rendered by the libvgcode viewer.

The values are derived by *re-parsing the G-code*, not by remembering decisions. Width is
back-solved from ΔE using three different cross-section models, then clamped
(`GCodeProcessor.cpp:4975-4991`); mm³/mm comes from ΔE over distance (`:4943-4956`); fan comes from
the last `M106` seen (`:6103-6114`); temperature from the last `M104`/`M109` (`:6090-6096`).

### 3.2 Widgets bound to the slider position

- **Position window** (`GCodeViewer.cpp:325`, `Marker::render_position_window`): an ImGui window
  showing the current vertex's type, role, width, height, and the value for the active view type.
  This is the natural host for a "why" section.
- **G-code window** (`GCodeViewer.cpp:789`, `GCodeWindow::render`, toggled with the `C` key,
  preference `show_gcode_window`): memory-maps the G-code file and lists the lines around the
  current one, colouring comments separately. It **truncates every line at 55 characters**
  (`:791-807`), so long trace comments would be cut off there.
- The viewer keeps `const GCodeProcessorResult* m_gcode_result` (`GCodeViewer.hpp:186`), so a
  side table stored in the processor result is reachable from the widgets without new plumbing.

### 3.3 Tags: the generator-to-preview channel

`GCodeProcessor::process_tags` (`GCodeProcessor.cpp:4126`) recognises comment lines that start
with a reserved tag. The tag arrays (`:59-103`) have two spellings, one for Bambu printers
(`; FEATURE: `, `; LINE_WIDTH: `) and one for everyone else (`;TYPE:`, `;WIDTH:`), selected by
`reserved_tag()` (`GCodeProcessor.hpp:519`). Tags are **sticky state**: `;WIDTH:0.45` applies to
every following move until the next `;WIDTH:`. The generator emits them only on change
(`GCode.cpp:8284-8294`).

A second family of **internal markers** carries information from the generator to post-processors
and is stripped before the file is written: `;_EXTRUDE_SET_SPEED`, `;_EXTERNAL_PERIMETER`,
`;_WIPE`, `;_EXTRUDE_END`, the seven `;_*_FAN_START/END` pairs, `;_FORCE_RESUME_FAN_SPEED`,
`;_EXTRUSION_ROLE:` and `;PA_CHANGE:`. The consumers are `CoolingBuffer.cpp:418-544`,
`PressureEqualizer.cpp:257-296`, `FanMover.cpp:422-436` and `AdaptivePAProcessor.cpp`.

### 3.4 Verbose G-code

The `gcode_comments` option (`PrintConfig.cpp:4355`, "Verbose G-code") already annotates some
decisions in prose: `; Ramp down-non-variable` around overhang speed changes
(`GCode.cpp:8456-8474`), `| Old Flow Value: … Length: …` when small-area flow compensation
changed E (`:8526-8528` and three sibling sites), and `; adjust acceleration` / `; adjust jerk` in
`GCodeWriter.cpp:379,431`. It is unstructured and partial, but it proves that per-line annotation
is accepted in this codebase and shows the injection points.

### 3.5 Post-processing pipeline order

`GCode::process_layers` (`GCode.cpp:4256-4352`, sequential-print copy at `:4358-4451`):

```
generator → [spiral_mode] → [pressure_equalizer] → cooling → fan_mover → [pa_processor] → output
```

Note that the adaptive-PA filter is dropped entirely when spiral vase is on (`:4343-4351`), and
`fan_mover` runs before the PA processor.

## 4. Decision inventory by attribute

Each subsection lists the steps that can move the attribute, in execution order, with the config
keys and the code location. This is the content a trace record would carry. "Stage" is
G = path generation, X = `GCode::_extrude` and callers, P = text post-processor.

### 4.1 Speed

Base value is chosen per role in `_extrude` (`GCode.cpp:7995-8032`), except skirt, brim and support
which arrive with a pre-decided speed from their callers (`:5064`, `:5134`, `:7640-7654`).

| # | Stage | Step | Keys | Location |
|---|---|---|---|---|
| 1 | X | Role base speed (`inner_wall_speed`, `outer_wall_speed`, `bridge_speed`, `internal_bridge_speed`, `sparse_infill_speed`, `internal_solid_infill_speed`, `top_surface_speed`, `ironing_speed`/`filament_ironing_speed`, `initial_layer_infill_speed`, `gap_infill_speed`, `support_speed`, `support_interface_speed`) | as listed | `GCode.cpp:7995-8032` |
| 2 | X | Scarf joint clamp on walls | `scarf_joint_speed` | `:7999`, `:8005` |
| 3 | X | Small perimeter (loop length ≤ threshold) | `small_perimeter_threshold`, `small_perimeter_speed` | `extrude_loop`, `:7262-7268` |
| 4 | X | Small support perimeter | `small_support_perimeter_threshold`, `small_support_perimeter_speed` | `:7640-7654` |
| 5 | X | Auto speed (role speed 0 → volumetric limit / mm³ per mm) | `filament_max_volumetric_speed` | `:8041-8042` |
| 6 | X | First layer / over-raft overwrite | `initial_layer_speed`, `initial_layer_infill_speed` | `:8045-8053` |
| 7 | X | Slow-down layers ramp | `slow_down_layers` | `:8054-8079` |
| 8 | X | Skirt speed | `skirt_speed` | `:8080-8084` |
| 9 | X | Volumetric clamp (adaptive variant folds in a fitted curve) | `filament_max_volumetric_speed`, `filament_adaptive_volumetric_speed`, `volumetric_speed_coefficients` | `:8034-8039`, `:8095-8098` |
| 10 | X | Resonance avoidance (can raise as well as lower) | `resonance_avoidance`, `min_/max_resonance_avoidance_speed` | `:8100-8133` |
| 11 | X | Dynamic overhang speed, per point, relative to a separately computed reference speed | `enable_overhang_speed`, `overhang_1_4_speed` … `overhang_4_4_speed`, `slowdown_for_curled_perimeters`, `bridge_speed` | `:8135-8207`, `ExtrusionProcessor.hpp:421-563` |
| 12 | X | Emission suppression: per-point changes under 1 mm/s are not written | — | `:8724-8731` |
| 13 | P | Extrusion rate smoothing rescales F and splits lines | `max_volumetric_extrusion_rate_slope`, `…_segment_length`, `extrusion_rate_smoothing_external_perimeter_only` | `PressureEqualizer.cpp:503-613`, `:918-953` |
| 14 | P | Layer-time slowdown, min speed, outer wall exemption | `slow_down_for_layer_cooling`, `slow_down_layer_time`, `slow_down_min_speed`, `dont_slow_down_outer_wall` | `CoolingBuffer.cpp:638-692`, rewrite at `:943-1005` |

Non-obvious edges:

- **Flow settings change speed.** The volumetric clamp in step 9 uses the flow-ratio-adjusted
  mm³/mm, so any of the fifteen flow-ratio keys can lower the speed when the clamp binds.
- **Overhang speed uses its own reference.** Step 11 recomputes a reference from the wall speed and
  the volumetric limit, ignoring steps 6, 7, 8 and 10 (`:8143-8153`). A truthful trace needs both.
- No print-level volumetric clamp exists in this fork; the legacy call is commented out
  (`:2969`, `:8086-8093`).

### 4.2 Acceleration and jerk

Acceleration is a strict ladder in `_extrude` (`GCode.cpp:7883-7910`), gated on
`default_acceleration > 0`: first layer → bridge → sparse infill → solid infill → outer wall →
inner wall → top surface → default. There is no support, gap-fill or ironing key. Jerk
(`:7912-7928`) uses a different ladder with no bridge branch. Travel acceleration and jerk are
decided in `travel_to` (`:8867-8932`) with short-travel special cases.

The writer **suppresses repeated values** (`GCodeWriter.cpp:356-358`, `:386`, `:470-471`) and the
value is clamped to machine limits (`:348-351`). Custom G-code can invalidate the writer's memory
(`GCode.cpp:4496-4500`). A trace must therefore record the *decision*, not the emitted command,
or it will show gaps.

### 4.3 Line width

Width is decided at path generation and is **advisory** from then on: the emitted E comes from
`mm3_per_mm`, and `;WIDTH:` only feeds the preview.

| # | Stage | Step | Keys | Location |
|---|---|---|---|---|
| 1 | G | Per-role width with fallbacks (first layer width beats every role; role key 0 → `line_width`; 0 → auto width from nozzle) | `initial_layer_line_width`, `outer_wall_line_width`, `inner_wall_line_width`, `sparse_infill_line_width`, `internal_solid_infill_line_width`, `top_surface_line_width`, `line_width`, `nozzle_diameter` | `PrintRegion.cpp:25-54`, `Flow.cpp:21-36`, `:129-144` |
| 2 | G | Bridge flow: thick (round, diameter × √ratio) or thin (width override then flow ratio, which also changes width) | `thick_bridges`, `thick_internal_bridges`, `bridge_line_width`, `bridge_flow` | `LayerRegion.cpp:31-60`, `Flow.cpp:167-198` |
| 3 | G | Narrow-island external perimeter narrowed by a hard-coded tolerance | `detect_thin_wall` | `PerimeterGenerator.cpp:1371-1377`, `:1457-1470` |
| 4 | G | Overhang split: part of a loop gets the overhang flow | `detect_overhang_wall`, `extra_perimeters_on_overhangs` | `:199-204`, `:484-485`, `:1155-1176` |
| 5 | G | Arachne variable width: bead widths from percent-of-nozzle parameters, spacings converted to widths, quantised at 0.05 mm | `wall_generator`, `min_bead_width`, `initial_layer_min_bead_width`, `min_feature_size`, `wall_transition_*`, `wall_distribution_count`, `precise_outer_wall` | `WallToolPaths.cpp:26-66`, `:516-540`; `VariableWidth.cpp:65-77` |
| 6 | G | Thin walls and gap fill: length-weighted mean width from the medial axis; gap fill uses the **solid infill** flow | `detect_thin_wall`, `filter_out_gap_fill`, `gap_fill_target` | `PerimeterGenerator.cpp:1440-1455`, `:1822-1866`; `VariableWidth.cpp:99-214` |
| 7 | G | Infill width re-fitted to achieved spacing | (pattern-driven) | `FillBase.cpp:145-186`, `FillRectilinear.cpp:3741-3761` |
| 8 | G | Infill combination raises the height, which raises mm³/mm | `infill_combination`, `infill_combination_max_layer_height` | `PrintObject.cpp:4340-4460` → `Fill.cpp:1005` |
| 9 | G | Ironing has a hand-rolled flow bypassing `Flow` | `ironing_flow`, `ironing_spacing`, filament overrides | `Fill.cpp:1806-1827` |
| 10 | G | Support, brim, skirt flows with their own fallback chains | `support_line_width`, `initial_layer_line_width`, `support_filament` | `Flow.cpp:232-271`, `Print.cpp:2101-2139` |

The preview clamps displayed width to `max(2.0, 4 × height)` (`GCodeProcessor.cpp:4991`), so the
displayed value can differ from the decided value.

### 4.4 Flow

The flow cascade in `_extrude` (`GCode.cpp:7938-7993`) is a product of named multipliers, which
makes it the cheapest attribute to trace: a bitmask of applied keys plus the resulting factor is
complete.

| # | Stage | Step | Keys | Location |
|---|---|---|---|---|
| 1 | X | Global multipliers | `print_flow_ratio`, `filament_flow_ratio` | `:7939-7942` |
| 2 | X | Role multiplier, first group (mutually exclusive) | `top_solid_infill_flow_ratio`, `bottom_solid_infill_flow_ratio`, `internal_bridge_flow`, `brim_flow_ratio`, `scarf_joint_flow_ratio` | `:7944-7954` |
| 3 | X | Role multiplier, second group, behind a master gate | `set_other_flow_ratios` + `outer_wall_`, `inner_wall_`, `overhang_`, `sparse_infill_`, `internal_solid_infill_`, `gap_fill_`, `support_`, `support_interface_flow_ratio` | `:7956-7972` |
| 4 | X | First layer multiplier (additive to the above, not skirt/brim) | `first_layer_flow_ratio` | `:7975-7978` |
| 5 | X | Mixed-colour sub-layer scaling | — | `:7981-7988` |
| 6 | X | Small-area infill flow compensation, per line, solid roles only | `small_area_infill_flow_compensation`, `…_model` | `:8522-8529` and three siblings; `SmallAreaInfillFlowCompensator.cpp:91-111` |
| 7 | X | Scarf seam E ratio per segment | `seam_slope_*`, `scarf_*` | `:8557-8563`, `:8763-8765` |
| 8 | X | Z anti-aliasing E ratio per segment | `zaa_*` | `:8530-8547`, `:8741-8757` |
| 9 | P | Spiral vase transition ramps and XY smoothing rescale E | `spiral_starting_flow_ratio`, `spiral_finishing_flow_ratio`, `spiral_mode_smooth` | `SpiralVase.cpp:150-160`, `:185` |

`filament_flow_ratio` is applied and then divided out again (`:7942`, `:7993`); only the
volumetric clamp and the PA tags see it. The pressure equalizer changes F, never E per mm.

### 4.5 Fan

Fan is the hardest attribute because the deciding code (`CoolingBuffer`) works on text one stage
after the emitting code and has no access to paths or config provenance.

| # | Stage | Step | Keys | Location |
|---|---|---|---|---|
| 1 | X | Role markers emitted around overhang, bridge, internal bridge, support interface and ironing paths; edge-triggered, once per role per layer | `enable_overhang_bridge_fan`, `overhang_fan_threshold`, `support_material_interface_fan_speed`, `ironing_fan_speed`, `internal_bridge_fan_speed` | `GCode.cpp:8335-8412`, `:8498-8506`, `:8646-8670` |
| 2 | X | Overhang classification per point from measured overlap | `overhang_fan_threshold` thresholds 0.9/0.75/0.5/0.25/0.05 | `:8345-8369` |
| 3 | P | Layer time estimate (constant-velocity model, no acceleration) | — | `CoolingBuffer.cpp:391-474` |
| 4 | P | First layer hard override | `initial_layer_fan_speed` | `:766-777` |
| 5 | P | Base fan from layer time interpolation | `fan_min_speed`, `fan_max_speed`, `slow_down_layer_time`, `fan_cooling_layer_time`, `reduce_fan_stop_start_freq` | `:784-792` |
| 6 | P | Layer ramp | `close_fan_the_first_x_layers`, `full_fan_speed_layer` | `:795-815` |
| 7 | P | Role fan arbitration: overhang > internal bridge > support interface > ironing > base; overhang wins only if higher than base | keys from step 1 | `:816-832`, `:1026-1052` |
| 8 | P | PWM floor, flavour-specific command | `part_cooling_fan_min_pwm`, `gcode_flavor` | `GCodeWriter.cpp:1262-1298` |
| 9 | P | Fan mover advances increases in time and can kickstart; splits G1 lines | `fan_speedup_time`, `fan_speedup_overhangs`, `fan_kickstart` | `FanMover.cpp:115-211`, `:317-361` |
| 10 | — | Wipe tower and nozzle-change code write raw `M106` outside the model | — | `WipeTower.cpp:1223-1233`, `:3478-3481` |

Fan keys live only in filament and printer presets. **There is no per-object, per-modifier or
per-layer-range fan override anywhere**, so provenance for fan is trivial (which filament preset).
The `;_SET_FAN_SPEED_CHANGING_LAYER` marker emitted at `GCode.cpp:5638` has no consumer and is an
available per-layer anchor.

### 4.6 Temperature

Temperature is nearly static and is the cheapest attribute:

- First layer vs other layers: `nozzle_temperature_initial_layer` then `nozzle_temperature`,
  switched once at layer 2 (`GCode.cpp:5759-5781`); a value of 0 means "keep first-layer value".
- Ooze prevention standby and restore around tool changes (`:389-431`, `:9404`, `:9744`),
  keys `ooze_prevention`, `standby_temperature_delta`, `idle_temperature`.
- Tool change targets and overrides (`:9410-9436`), wipe tower interface temperatures
  (`WipeTower2.cpp:1367-1404`).
- Temperature tower calibration ramp (`:5648-5651`).
- Custom G-code (start, layer change, filament start, role change) is opaque text and can only be
  attributed as "custom G-code at layer N".

No per-object or per-region temperature keys exist. Two fidelity hazards in the preview:
`process_M104` ignores the `T` parameter (`GCodeProcessor.cpp:6090-6096`), and the pre-heat /
pre-cool injector adds `M104` lines after moves are recorded (`:2057-2270`), so
`MoveVertex::temperature` is not ground truth on multi-extruder printers.

## 5. Provenance: where a value came from

### 5.1 How the effective config is built

The effective region config is a five-layer merge in `region_config_from_model_volume`
(`PrintObject.cpp:3838-3884`): global default (or the parent modifier's region) → object config →
volume config → material config → layer-range config. It is a plain key-by-key overwrite; the only
memory it keeps is a six-bit mask for feature filament ids, discarded on return. Regions with
identical merged configs are **interned** into one `PrintRegion` (`PrintApply.cpp:1030-1044`),
and a modifier that changes nothing is recorded pointing at its parent's region (`:1130-1133`).
Object-level keys are a two-layer merge (`PrintObject.cpp:3731`).

Painted MMU regions override exactly six filament-id keys (`PrintApply.cpp:1143-1149`); painted
fuzzy skin overrides exactly one (`:1167`). Painted seam and painted support never become config
at all; they are consumed directly by `SeamPlacer.cpp:721-726` and `PrintObject.cpp:4826-4827`.

At G-code time `GCode::m_config` is a flat `FullPrintConfig` (`GCode.hpp:586`) that is overwritten
in sequence (print → object → region, `GCode.cpp:2965-2967`, `:5503`, `:6414-6415`,
`:7590`, `:7617`). **Any value read from `m_config` has already lost its source.**

### 5.2 What survives slicing

`PrintObjectRegions` (`Print.hpp:236-340`) survives and keeps, per layer range: the layer-range
config pointer, each volume's `ModelVolume*`, its parent chain, its bounding box and its
`PrintRegion*`; plus painted regions with their extruder id. The invariant
`layer->get_region(i)->region().print_object_region_id() == i` (asserted at
`PrintObjectSlice.cpp:930`) means a `LayerRegion` index *is* the index into that structure.
The existing `verify_update_print_object_regions` (`PrintApply.cpp:745-890`) already re-runs
the merge from sources on demand, which is the strongest evidence that a read-only resolver is
tractable.

Two dedup levels matter: `print_object_region_id` (per object) and `print_region_id` (across
objects, `PrintApply.cpp:2013-2031`). The G-code loops use the global one (`GCode.cpp:5987`,
`:7590`), so a hook there must also carry the `PrintObject` to get back to provenance.

### 5.3 Resolver design

A read-only `ProvenanceResolver` computed **once per PrintRegion per export**, not per path:

```
input:  PrintObject, print_object_region_id, print_z, option key
output: {GlobalPreset, ObjectOverride, VolumeOverride(volume id), MaterialOverride,
         LayerRangeOverride(range), PaintedMMU(extruder), PaintedFuzzySkin, DerivedClamp}
```

Implementation: locate the `LayerRangeRegions` for `print_z` with `layer_range_first/next`
(`PrintObjectSlice.cpp:244,258`); find the `VolumeRegion` whose region id matches; walk the
parent chain; for each candidate `ModelConfig` call `has(key)` (`PrintConfig.hpp:2374`) in merge
order and take the last hit. `DerivedClamp` covers the post-merge synthesis at
`PrintObject.cpp:3869-3882` (filament clamps, density clamp, fuzzy skin reset).

Per-extruder keys need an extra answer: which vector index was used, and via which of the three
indexing schemes (`EXTRUDER_CONFIG`, `FILAMENT_CONFIG`, `NOZZLE_CONFIG`, `GCode.cpp:2001-2003`).
The filament and nozzle indices are **layer-dependent** (`:4234-4252`). `CoolingBuffer` indexes
by raw filament id (`CoolingBuffer.cpp:739`), which can disagree with the generator on
multi-nozzle configs; a trace would make this pre-existing inconsistency visible.

Caveats a design must state: interned regions make the answer "volume A or B" when two volumes
have identical overrides; no-op modifiers must not be credited; vector options with per-variant
overrides (`set_variant_override`, `PrintObject.cpp:3815`) have per-element provenance.

## 6. Cross-cutting constraints

1. **Line identity.** Assigning an id per G1 in the generator does not survive
   `PressureEqualizer` (splits, `:517-613`), `FanMover` (splits, `FanMover.cpp:115-211`) or
   `CoolingBuffer` (elides unchanged `G1 F` lines, `CoolingBuffer.cpp:956-960`). Sticky comment
   lines do survive: `CoolingBuffer` copies unknown comment lines verbatim, `PressureEqualizer`
   drops only `;_EXTRUSION_ROLE:` lines (`PressureEqualizer.cpp:257-264`) and passes other
   comments as `GCODELINETYPE_OTHER`, `FanMover` passes comment lines through (`FanMover.cpp:422`).
2. **Decision vs emission.** Writer dedup (acceleration, jerk, fan) and the 1 mm/s suppression for
   overhang speeds mean the emitted stream under-reports decisions. Record decisions.
3. **Three time models.** `CoolingBuffer` (constant velocity), `FanMover` (its own), and
   `GCodeProcessor` (trapezoidal). Fan and slowdown are decided by the first; the preview's layer
   time is the third. A trace that says "layer time 4.2 s" must say which model.
4. **Non-local state.** Fan is a fold over all preceding markers and commands, deduplicated at
   three levels. The trace must carry the resolved value at each change, not just the trigger.
5. **Preview data model.** `MoveVertex` has no region or volume id; `object_label_id` exists only
   when `gcode_label_objects` is on. The libvgcode path split test (`LibVGCodeWrapper.cpp:217-219`)
   keys on type, role, mm³/mm, acceleration and jerk, not fan or temperature.
6. **Two tag spellings.** Any new reserved tag must be added to both `Reserved_Tags` arrays or
   emitted via a spelling-independent constant, like `Flush_Start_Tag` (`GCodeProcessor.hpp:471`).
7. **Backward compatibility.** Trace tags in an exported file must be ignorable by older
   OrcaSlicer versions and by firmware. Comment lines are safe on both counts. 3MF is untouched
   unless the option is stored in the project config, which any new print option is.

## 7. Data channel options

### Option A — In-memory side table only

Record decisions into a structure keyed by output byte offset or line index as the generator
emits, and pass it alongside the text through the pipeline.

- Pro: no file-size impact; typed data, no parsing.
- Con: the TBB pipeline stages exchange `std::string` (`LayerResult`, `GCode.hpp:176-188`); every
  stage that splits or deletes lines would need to remap the table. `PressureEqualizer` and
  `FanMover` do not track source line indices today. High-touch, fragile.

### Option B — Comment tags in the G-code stream

Emit sticky `;_TRACE` comment lines from the generator; let post-processors append their own trace
lines where they change a value; parse everything in `GCodeProcessor::process_tags` into a
compact table in `GCodeProcessorResult`.

- Pro: reuses the exact mechanism that already carries role, width and height. Survives all
  post-processors without modifying their line bookkeeping. Visible in the existing G-code window
  and in any text editor. Debuggable by reading the file.
- Con: file size and parse time grow when enabled. Data must be encoded compactly. The G-code
  window truncates at 55 characters.

### Option C — Hybrid (recommended)

Option B for transport, plus a post-parse step that interns records and builds a per-line index in
`GCodeProcessorResult`; a separate opt-in decides whether the tags remain in the exported file or
are stripped during the existing post-process rewrite (`GCodeProcessor.cpp:1024-1776`, which
already rewrites the file for `M73` and pre-heating).

This keeps the generator and post-processor changes small and local, and moves all the cost into
the processor and viewer, which are the parts a user only pays for when the option is on.

## 8. Recommended design sketch

### 8.1 Records

One record per attribute per change, encoded on its own comment line so it survives every stage.
Illustrative, not final:

```
;_TR S base=outer_wall_speed:200@g vol=filament_max_volumetric_speed:120@f1 slow=slow_down_layers:3/5:96
;_TR W base=outer_wall_line_width:0.42@v17
;_TR F print_flow_ratio:0.98@g outer_wall_flow_ratio:1.02@v17
;_TR A outer_wall_acceleration:5000@g
;_TR L cool: t=4.1s<slow_down_layer_time:5s@f1 factor=0.82 fan=fan_max_speed:100@f1
;_TR N overhang: overlap=0.31 -> overhang_3_4_speed:25%@g ref=200 -> 50
```

Elements: attribute letter; ordered steps of `key:value@source`; `@source` refers to a source
table emitted once in the header (`;_TR_SRC v17 volume "Modifier 1" object "Bracket"`), so
per-path records stay short. Geometry-driven inputs (overlap, loop length, layer time) are
carried as plain numbers. Records are emitted only when the record text differs from the last
one for that attribute, exactly like `;WIDTH:`. Within a region and role most paths share a
record, so the emission rate should be close to the existing `;TYPE:` rate; this must be measured
in a spike, not assumed.

Per-line overrides inside a path (overhang points, small-area compensation, scarf, ZAA) emit a
short record before the affected G1 only when the value changed by more than a display threshold.

### 8.2 Generator hooks

- `GCode::_extrude`: after the speed ladder (`GCode.cpp:8209`), after the acceleration and jerk
  ladders (`:7909`, `:7926`), after the flow cascade (`:7993`), and at the variable-speed segment
  loop (`:8677`). All decisions for speed, accel, jerk and flow are already local variables
  there; the change is to record which branch fired.
- Width: record at `PrintRegion::flow` (`PrintRegion.cpp:31-53`), `LayerRegion::bridging_flow`
  (`LayerRegion.cpp:44-57`), and the variable-width conversion (`VariableWidth.cpp:65-77`,
  `:126-136`). Since `ExtrusionPath` has no room for a trace, the cheapest route is a small
  `uint16_t` index into a per-region record table stored on the path, or re-deriving the width
  record at `_extrude` time from role plus region config (loses Arachne/thin-wall detail but
  costs nothing in path storage). Recommended for MVP: re-derive at `_extrude`, and mark
  variable-width paths as "Arachne/variable" without per-bead detail.
- Fan intent: record the marker predicate outcome (`:8335-8412`) with the measured overlap.
- Temperature: record at the layer-2 switch (`:5759-5781`), ooze prevention, tool change and
  calibration ramp.

### 8.3 Post-processor hooks

- `CoolingBuffer::apply_layer_cooldown`: emit one layer record (layer time under its own model,
  thresholds, stretch factor, base fan chosen, ramp factor) at layer start (`:866`), and on each
  role-fan arbitration (`:1026-1052`) a record naming the winning key. For slowed lines
  (`:963-965`) append a short suffix such as `;_TR c` to the modified line rather than a new line,
  or emit a record when the `slowdown` flag toggles.
- `PressureEqualizer::push_line_to_output`: when the emitted F differs from the input, emit a
  record naming `max_volumetric_extrusion_rate_slope` and the ratio.
- `FanMover`: when it moves or kickstarts a fan command, emit a record naming
  `fan_speedup_time` / `fan_kickstart`. Its split lines inherit the sticky state automatically.
- Spiral vase: emit a record on the transition layers.

### 8.4 Processor and viewer

- `GCodeProcessor::process_tags`: parse `;_TR` into an interned record table
  (`std::vector<std::string>` or a small struct with key ids) and keep current per-attribute
  record ids as sticky state. Add `uint32_t trace_id` to `MoveVertex`, or keep a separate
  `std::vector<std::pair<line_id, trace_ref>>` in `GCodeProcessorResult` searched by the vertex's
  `gcode_id`. The second avoids touching libvgcode and the vertex size.
- `GCodeViewer::SequentialView::Marker::render_position_window`: add a collapsible "Why" section
  listing, per attribute, the resolved steps. The result pointer is already available
  (`GCodeViewer.hpp:186`). Optionally raise the G-code window truncation limit when the trace
  option is on.
- Provenance display: resolve source ids through the header table; show the human name of the
  object, modifier or layer range.

### 8.5 Gating

- New print-preset option, e.g. `gcode_settings_trace` (bool, `comDevelop` or `comAdvanced`),
  registered in `Print::steps_gcode` (`Print.cpp:105-245`) so toggling it re-exports G-code
  without re-slicing. All fan and temperature inputs are already in that set.
- Second option, or a sub-mode, for "keep trace in exported file". Default: strip on export, keep
  for the preview. Stripping happens in the existing post-process rewrite.
- With the option off: no records are built, no tags are emitted, `process_tags` never sees them.
  The only cost is one boolean check per hook.

## 9. Cost estimates

Memory and file size scale with the number of *changes*, not lines, under the sticky encoding.
Numbers to measure in the spike on three reference models (a calibration cube, a 2-hour organic
part with overhangs, and a large 20-hour plate):

| Quantity | Expectation | Why it matters |
|---|---|---|
| Records per 1000 G1 lines | tens, not hundreds | file size and parse time |
| Bytes per record | 60 to 120 before interning | file size |
| Per-move overhead in the processor | 4 bytes if stored on `MoveVertex`, 0 with the side index | preview memory on multi-million-move prints |
| Parse time increase | proportional to record count; `process_tags` is a prefix match | slicing-to-preview latency |
| `_extrude` time increase | negligible when off; string formatting when on | export time |

If measurements show the per-path emission rate is too high on Arachne walls (many short
variable-width paths), fall back to per-role records plus a "variable width" flag.

## 10. Phasing

**Phase 0 — Spike (1 to 2 days).** Emit a single hard-coded `;_TR S` record from `_extrude` on
speed changes, parse it in `process_tags`, print it in the position window. Measure record rate
and file growth on the three reference models. Confirm the records survive the pressure
equalizer, cooling and fan mover on a model with all three enabled.

**Phase 1 — MVP.** Speed, flow and width traces from the generator; cooling slowdown layer
record; the "Why" section in the position window; the opt-in option; strip-on-export. Provenance
at the coarse level only (global vs object vs modifier vs layer range) via the resolver in 5.3.

**Phase 2.** Fan arbitration records from `CoolingBuffer`; acceleration and jerk; temperature;
pressure equalizer and fan mover records; source names in the UI; the G-code window limit.

**Phase 3.** Grouping of unchanged runs; per-bead Arachne width detail; tier-2 attributes (PA,
seam, z-hop, layer height); hover picking in the 3D view.

## 11. Decisions to surface

1. **Where the trace lives in the exported file.** Strip by default and keep only for the preview,
   or keep it when the user asks. Keeping it makes the file self-describing for bug reports.
2. **Record vocabulary.** Machine-readable keys (`outer_wall_speed`) or the localized UI labels.
   Keys are stable, searchable and language-independent; labels are friendlier. Recommend keys in
   the file and labels in the UI, mapped through `PrintConfig` definitions.
3. **Width detail level.** Re-derive at `_extrude` (cheap, loses Arachne per-bead reasons) or
   carry an index on `ExtrusionPath` (exact, touches a core struct copied everywhere).
4. **Fan trace depth.** Layer-level record only (what the base fan was and why) plus the role
   arbitration winner, or full per-line resolution including fan mover time shifts.
5. **Provenance granularity for interned regions.** Report "one of: Modifier A, Modifier B" or
   resolve by bounding-box intersection with the move position.
6. **Option placement and mode.** Process preset vs printer preset; Advanced vs Develop mode.
   It reads like a process option (it changes G-code output) and belongs beside Verbose G-code.
7. **Upstream intent.** If the target is an upstream PR, keep the MVP to Option C with the
   side-index (no `MoveVertex` change) and no new fields on `ExtrusionPath`.

## 12. Risks and open questions

- **Divergence between trace and truth.** The trace records what the generator decided; the file
  contains what post-processors did. Every post-processor that changes a traced attribute must
  emit its own record or the trace will lie. The four writers of F (generator, PE, cooling,
  fan mover split) and the two writers of E outside the generator (spiral vase, PE split) are the
  full list at this commit; a new post-processor added later would silently degrade the trace.
  Mitigation: a regression test that asserts the traced speed equals the parsed feedrate for a
  reference model when no post-processor is active, and a documented contract for post-processors.
- **Bambu vs compatible tag spelling.** Use a single spelling-independent constant.
- **Custom G-code.** Anything set inside custom G-code blocks is attributed only as "custom
  G-code". `custom_gcode_motion_state_changes` shows the generator already reasons about this.
- **Skirt and brim config staleness.** They are generated before the per-instance region reset
  (`GCode.cpp:6407-6415`), so region-keyed values they read may come from the previous instance's
  last region. A trace would expose this; decide whether to fix it first.
- **Layer id mismatch.** `CoolingBuffer` gates on `layer.id()` while marker emission uses
  `m_layer_index` (`GCode.cpp:8386`). Records from the two stages must use one layer identity.
- **Sequential printing** duplicates the pipeline wiring (`GCode.cpp:4358-4451`); both copies
  need the hooks.
- **Localization.** UI strings for step names go through `_L`; the file keeps keys.
- **Untested area.** Whether the G-code window is worth extending or the position window is
  enough can only be judged by using it; both are ImGui and cheap to iterate.

## 13. Verification strategy for the prototype

- **Catch2, `tests/fff_print`.** Slice a cube with the option on and known settings; assert that
  the exported text contains the expected `;_TR S` record for outer walls, that enabling
  `slow_down_layers` adds the expected step, and that disabling the option produces a
  byte-identical file to the baseline.
- **Catch2, `tests/libslic3r`.** Feed synthetic G-code with `;_TR` lines to `GCodeProcessor` and
  assert the per-line lookup; feed a layer through `CoolingBuffer` with a slow layer and assert a
  cooling record is emitted and existing marker stripping is unchanged.
- **Round-trip invariant.** For a model with no post-processors active, the traced final speed
  must equal `MoveVertex::feedrate` for every extrusion move.
- **Manual.** Three reference models, before/after file size, preview load time, and a checklist
  of settings toggled one at a time (overhang speed, small perimeter, volumetric limit, layer
  time slowdown, bridge flow, modifier override) confirming each appears in the panel.

## 14. Appendix: reference index

Pipeline and tags: `GCode.cpp:4256-4352` (pipeline), `GCodeProcessor.cpp:59-103` (tag arrays),
`:4126` (`process_tags`), `:7015-7050` (`store_move_vertex`), `GCodeProcessor.hpp:215-249`
(`MoveVertex`).

Speed: `GCode.cpp:7995-8032`, `:8034-8209`, `:8373-8377`, `:8492`, `:8644`, `:8724-8731`;
`ExtrusionProcessor.hpp:421-563`; `CoolingBuffer.cpp:338-558`, `:638-692`, `:943-1020`;
`PressureEqualizer.cpp:471-720`, `:918-953`.

Acceleration and jerk: `GCode.cpp:7883-7936`, `:8867-8932`; `GCodeWriter.cpp:345-498`.

Width and flow: `Flow.cpp:21-216`, `PrintRegion.cpp:25-54`, `LayerRegion.cpp:21-60`,
`PerimeterGenerator.cpp:111-224`, `:394-547`, `:1371-1377`, `:1822-1866`; `WallToolPaths.cpp:26-84`,
`:516-540`; `VariableWidth.cpp:5-234`; `Fill.cpp:880-1048`, `:1806-1827`; `FillBase.cpp:145-250`;
`GCode.cpp:7938-7993`, `:8271-8294`, `:8521-8563`; `SmallAreaInfillFlowCompensator.cpp:27-111`;
`SpiralVase.cpp:66-216`.

Fan: `GCode.cpp:8335-8412`, `:8498-8506`, `:8646-8670`; `CoolingBuffer.cpp:48-74`, `:733-862`,
`:866-1052`; `GCodeWriter.cpp:1262-1338`; `FanMover.cpp:89-436`.

Temperature: `GCode.cpp:389-431`, `:4630-4715`, `:5759-5781`, `:9410-9436`; `GCodeWriter.cpp:243-339`;
`GCodeProcessor.cpp:6090-6096`, `:6167-6183`, `:1350-1428`, `:2057-2270`.

Provenance: `PrintObject.cpp:3731`, `:3764-3884`; `PrintApply.cpp:977-1200`, `:745-890`,
`:2013-2031`; `Print.hpp:236-340`; `PrintObjectSlice.cpp:244-258`, `:930`, `:1156-1166`;
`GCode.cpp:2001-2003`, `:2965-2967`, `:5987`, `:6414-6415`, `:7590`, `:7617`; `GCode.hpp:586`;
`PrintConfig.hpp:2328-2402` (`ModelConfig`).

Viewer: `GCodeViewer.cpp:325` (position window), `:789` (G-code window), `:1205` (libvgcode
conversion), `GCodeViewer.hpp:186`; `LibVGCodeWrapper.cpp:191`; `libvgcode/include/PathVertex.hpp`,
`Types.hpp:80-102`.
