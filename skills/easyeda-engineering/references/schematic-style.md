# Schematic Engineering Style

Read this reference when drawing, reorganizing, or evaluating an EasyEDA schematic. These are portable defaults; explicit project rules may override them.

## Default drawing frame

When the project has no explicit sheet rule, use this default logical drawing boundary:

| Position | Coordinate `(x, y)` |
| --- | --- |
| Top left | `(0.1, 8.15)` |
| Bottom left | `(0.1, 0.1)` |
| Top right | `(11.6, 8.15)` |
| Bottom right | `(11.6, 0.1)` |

Keep all schematic content within `0.1 ≤ x ≤ 11.6` and `0.1 ≤ y ≤ 8.15`.

Reserve the lower-right annotation/title area as a hard no-draw zone by default:

- Top-left corner: `(4.6, 1.9)`
- Bottom-right corner: `(11.6, 0.1)`
- Inclusive region: `4.6 ≤ x ≤ 11.6` and `0.1 ≤ y ≤ 1.9`

Do not place components, wires, labels, text, or other schematic primitives in that region. If the active sheet uses a different coordinate system or title-block geometry, derive the real boundary first and apply the same two invariants: stay inside the sheet and outside the annotation area.

## Layout

- Optimize for human review, fault finding, and real fabrication, not merely for formal electrical attachment.
- Arrange the main signal path from left to right and power or control dependencies from top to bottom unless the circuit naturally requires another direction.
- Group parts by function: external interface, protection, power, controller, logic level, physical layer, and output interface.
- Keep each IC's decoupling, charge-pump, oscillator, reset, and mode-setting components beside that IC.
- Put board-edge connectors near the edge of their functional block and orient them for clear reading and later PCB placement.
- Align related parts to a consistent grid. Leave enough whitespace for values, designators, and pin names to remain readable.
- Keep every primitive inside the repository-defined sheet boundary and title-block keep-out. Use live sheet geometry only when no local boundary is defined.
- On an already-wired sheet, move a functional cluster together with its local wires and markers when the available API can preserve them. Otherwise use small moves and verify pin-to-net topology after every batch.
- Add functional boxes or notes only when they materially improve navigation. They are not mandatory on a compact single-purpose sheet unless repository rules require them.
- Do not reverse the reading direction or fold the same signal path back and forth without a concrete reason.

## Wires and labels

- Use short direct wires within a block when they make the relationship obvious.
- Use a network label only when a wire would cross a substantial distance, leave the block, or reduce readability.
- Do not fan out every pin to a label by default.
- Route wires horizontally and vertically. Use an L path for non-aligned endpoints and keep route channels free of unrelated pins.
- Do not draw through an unrelated pin; EasyEDA may split and electrically join the wire there.
- For a multi-pin local net, prefer a pin-anchored chain or a clearly named port. Do not depend on an unanchored free-space star junction.
- Never emit a zero-length wire. Power and ground flags must sit at the end of a real non-zero stub rather than directly on a bare pin.
- Use exactly one name for a physical net. If two functions are intentionally tied, use one combined name such as `DSR_DCD`.
- Keep active-low notation consistent throughout the project.
- Remove dangling stubs, duplicate markers, orphan ports, old-net remnants, and wire debris after a rename, delete, or replacement.
- Use consistent power and ground symbols instead of synonymous rail labels.
- Name nets by electrical domain and function, such as `USB_D+`, `TTL_TX`, and `COM_NSIN`. Keep spelling, case, prefixes, suffixes, and active-low markers identical everywhere the net appears.
- Do not attach an isolated label to make an unused pin appear handled; use an explicit NC marker.

## Components and annotations

- Choose the actual purchasable device and exact package intended for the PCB.
- Keep standard designators continuous from 1 within each class. Functional connector names are acceptable when repository rules allow them.
- Treat the designator returned after placement as authoritative; EasyEDA can renumber a planned designator when another page already uses it.
- Do not retain duplicate passives without an electrical or layout reason.
- Keep values, part names, designators, and important pin names visible and non-overlapping.
- Place explicit NC markers on intentionally unused pins.
- Orient each IC consistently with the signal flow: inputs toward the source and outputs toward the destination.
- Keep decoupling, charge-pump, crystal, reset, and mode-setting networks visibly associated with their owner IC and connect them locally.
- Place connectors at functional boundaries and orient them to reflect the real mating direction and intended PCB location when possible.
- Align repeated peripheral circuits consistently. Remove functionally redundant parallel parts unless the circuit or physical implementation requires them.
- After adding, deleting, or replacing parts, check each designator class for duplicates, gaps, and meaningless inherited high numbers; reannotate before handoff when required.
- A visually convenient symbol or footprint is not an acceptable substitute for the real connector, package, mechanical pins, or missing-pin pattern.

## Readability review

An engineer should be able to identify the power path, main receive/transmit path, cross-block signals, mode controls, and unused connector pins without searching the entire sheet. If this requires matching many scattered labels, reorganize the drawing before sign-off.

The canvas is a visual-review surface only. Re-export and compare the netlist after any refactor that moved electrical primitives.

## Schematic completion gate

- Run DRC and resolve or explicitly explain every error and warning.
- Export or read the actual generated netlist and inspect power, receive/transmit direction, handshake signals, merged nets, and every external connector pin.
- For critical interfaces, compare each final `designator.pin → net` row with the equipment manual.
- Before PCB synchronization, verify that every fabricated part has the intended footprint and that exposed pads, mechanical pins, shield pins, and missing-pin positions are handled.
- After a delete, rename, or reannotation, repeat the netlist check, remove leftovers, save, and verify persistence.
