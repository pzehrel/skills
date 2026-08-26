# EasyEDA Verification Loop

Read this reference for any live edit, schematic-to-PCB handoff, audit, or sign-off.

## Before mutation

1. Verify bridge health and the active EasyEDA window, project, document, and document type.
2. Read the affected components, primitives, pins or pads, nets, geometry, and rules.
3. Capture the domain baseline:
   - Schematic: exported netlist, explicit NC state, pin-to-net mapping, packages, and DRC/check summary.
   - PCB: pad-to-net mapping, component and routing geometry, unrouted state, pours, rules, and DRC summary.
4. Record actual designators and primitive IDs. For visual refactoring, the affected topology becomes a golden baseline.

## During mutation

- Read the exact `easyeda-api` signature, types, units, enums, and remarks before each new API operation.
- Make one coherent batch at a time and read back what EasyEDA actually created.
- Treat a timeout, connector drop, or success without readback as an unknown write state. Inspect before retrying.
- After deletion or replacement, inspect all affected primitive types for leftovers.
- Save after a coherent verified batch that would be costly to reconstruct.

## Schematic gate

Export the netlist again and verify:

- Every changed pin is on the intended net or explicitly NC.
- The target net gained exactly the intended pins; the source net lost exactly the pins that moved; unrelated nets did not change.
- No intended nets merged or split, no conductor has conflicting labels, and no duplicate marker, orphan port, dangling or zero-length wire, or stale old-net stub remains.
- Designators, real parts, symbol functions, footprints, and exposed or mechanical pins remain correct.

Run DRC and structural checks again. The netlist proves electrical membership; the canvas judges readability only.

## PCB gate

Read the affected board data again and verify:

- Pads, tracks, vias, pours, and components retain the intended nets and geometry.
- Routed and unrouted connectivity changed only as intended.
- The board outline, keep-outs, component clearances, layer usage, copper state, and design rules remain valid.
- Copper has been rebuilt or refreshed when required after routing changes.
- DRC has no unexplained blocking violations.

Save and perform a fresh critical-state read so a transient in-memory result is not mistaken for persisted design data.

## Schematic-to-PCB handoff

- Compare schematic symbol-to-footprint assignments and connector pin tables before synchronization.
- Review the proposed change set; do not blindly accept deletions, footprint swaps, or net remaps.
- After synchronization, compare component count, designators, footprints, pads, and net membership with the approved schematic intent.
- Re-run PCB connectivity and DRC checks before placement or routing proceeds.

## Sign-off report

Report the project and documents checked, primitives or nets changed, interface mappings verified, gate and DRC results, saved checkpoints, deliberate exceptions, remaining warnings, and anything not verified. Do not claim schematic, PCB, or manufacturing readiness while critical discrepancies remain.
