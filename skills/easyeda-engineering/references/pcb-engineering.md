# PCB Engineering

Read this reference for PCB placement, routing, copper, schematic synchronization, mechanical review, or board sign-off.

## Baseline before mutation

- Verify the active PCB document and read the board outline, copper-layer count, layer roles, design rules, component positions and bounding boxes, pads and nets, tracks, vias, pours, keep-outs, and current DRC/connectivity state.
- Confirm which schematic and PCB document form the intended pair. Do not create or bind a replacement board without explicit authorization.
- Preserve a baseline for affected nets and components so placement or routing changes can be compared rather than judged from a screenshot.

## Placement

- Place fixed mechanical items first: board outline, holes, connectors, switches, antennas, displays, and other edge-constrained parts.
- Confirm connector mating direction, component side, pin 1, and mechanical clearance from the actual footprint and equipment drawing.
- Place primary ICs and critical functional blocks next; keep decoupling, crystal, bootstrap, charge-pump, feedback, termination, and protection parts close to the pins they serve.
- Keep RF, high-current, switching, differential, analog, and thermal constraints explicit. Do not let generic compactness override a critical placement rule.
- Read back actual component and pad geometry. Verify that bodies and courtyards remain inside the board and outside mechanical or RF keep-outs.

## Routing and copper

- Read the current design rules before selecting widths, clearances, vias, layers, or differential constraints. Use documented enums rather than guessed layer numbers.
- Route critical power, clock, differential, analog, high-current, and sensitive feedback nets deliberately before ordinary signals when their constraints require it.
- Treat removing existing tracks, changing the layer stack, replacing pours, or running a full autorouter as destructive or materially broad; require explicit authorization.
- After any routing or copper mutation, read back the affected tracks, vias, pads, net membership, and unrouted state before continuing.
- Rebuild or refresh copper after the last routing change when the EasyEDA workflow requires it. A stale pour can create false DRC results or hide a real clearance issue.
- A successful route command is not proof that the intended pads are connected or that a track survived save/reopen.

## PCB sign-off

Verify at least:

- Schematic-to-PCB component and net synchronization is intentional and complete.
- Every external connector pad matches the approved pin table.
- No unintended unrouted connection, short, wrong-net pad, duplicate footprint, or off-board component remains.
- DRC has no unexplained blocking errors; remaining warnings are reviewed and reported.
- Copper is current, plane and pour nets are correct, and critical clearances are respected.
- Silkscreen, polarity, pin-1, connector, and test-point markings are visible after assembly and do not overlap exposed copper.
- Board outline, holes, keep-outs, edge clearances, and component heights match the mechanical intent.
- The document is saved and critical geometry and routing remain present on a fresh read.

Use screenshots only to judge visual organization and mechanical plausibility. Use PCB data, connectivity, and DRC to prove the board state.
