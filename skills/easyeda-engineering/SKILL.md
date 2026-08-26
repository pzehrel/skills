---
name: easyeda-engineering
description: Engineer, refactor, and review EasyEDA designs across schematics and PCB with safe live editing and evidence-based verification. Use for circuit design, schematic cleanup, connector audits, part substitution, schematic-to-PCB handoff, PCB placement or routing review, DRC, and pre-manufacturing sign-off; pair with easyeda-api for live EasyEDA operations. Do not use for generic API lookup or extension development alone.
---

# EasyEDA Engineering

Produce EasyEDA designs that are readable, electrically correct, physically realizable, and verified from design data rather than screenshots. This skill supplies the engineering workflow and sign-off gates; use `easyeda-api` for documented live EasyEDA calls.

This skill is self-contained and must remain usable outside this repository. Apply a repository `AGENTS.md` as a local override when present, but never assume that the user or another project carries the rules from this workspace.

This skill synthesizes `easyeda-pro-claude-skill` and `easyeda-agent` without adding either project's CLI, daemon, or connector as a dependency. Read [references/upstream-integration.md](references/upstream-integration.md) when maintaining the skill or resolving a rule conflict.

## Route the task

- For schematic drawing, cleanup, or visual refactoring, read [references/schematic-style.md](references/schematic-style.md).
- For connectors, headers, transceivers, DTE/DCE wiring, active-low controls, or substitute parts, read [references/interface-review.md](references/interface-review.md).
- For PCB placement, routing, copper, mechanical review, or board sign-off, read [references/pcb-engineering.md](references/pcb-engineering.md).
- For any live edit, handoff, audit, or sign-off, read [references/verification-loop.md](references/verification-loop.md).
- For a read-only review, inspect and report; do not mutate unless the user also requests changes.

## Project scope

- Determine the active project boundary from repository instructions, workspace structure, or the selected EasyEDA project before reading or writing design data.
- If a workspace defines each second-level folder as an independent project, select exactly one such folder and never mix, move, or modify design data across sibling projects.
- When the target remains ambiguous, stop before mutation and ask the user to identify the project.
- Project-local rules may tighten or replace the default drawing geometry and conventions in this skill; otherwise use the self-contained defaults in `schematic-style.md` and `pcb-engineering.md`.

## Engineering invariants

- Confirm one target project boundary before touching project design data. Never mix data across project boundaries.
- Verify bridge health, active window, project, document, and document type before mutation.
- Inspect before mutate. Read the current components, primitives, nets, rules, and document geometry relevant to the task.
- Preserve user edits and design intent. Make the smallest coherent change that satisfies the request.
- Do not trust planned designators, primitive IDs, pin coordinates, pad numbers, or geometry after placement, synchronization, page switching, or reload. Read back what EasyEDA actually created.
- Treat an exported schematic netlist as schematic connectivity truth. Treat PCB DRC, pad-to-net membership, routed/unrouted connectivity, geometry readback, and rebuilt copper state as PCB truth. Screenshots and bare API success are not proof.
- Use documented API signatures and enums from `easyeda-api`; do not guess parameters, units, layer IDs, or return shapes.
- Apply small observable batches. If a write times out or its result is indeterminate, inspect before retrying to avoid duplicate or conflicting primitives.
- Save at meaningful verified checkpoints, then confirm that the saved state persists.
- Require explicit authorization immediately before destructive or materially broader actions such as clearing a page, replacing a board, removing routing, changing the layer stack, or bulk deletion.

## Workflow

1. Confirm project scope, bridge health, active window, project, document, and document type.
2. Capture a task-sized baseline: relevant parts and packages, pin/pad-to-net mapping, NC state, geometry, rules, DRC/check results, and current artifacts.
3. Restate the intended electrical and physical result. Build an explicit pin table for external interfaces before wiring or footprint approval.
4. Plan the change around signal flow, functional grouping, mechanical constraints, routing channels, and repository conventions.
5. Apply one coherent batch through documented APIs. Re-read affected primitives and actual IDs immediately afterward.
6. Run the domain gate: exported-netlist comparison for schematic work; connectivity, geometry, copper, and DRC checks for PCB work.
7. Resolve unexpected net merges or splits, remapped pins or pads, leftovers, collisions, clearance failures, or stale reads before continuing.
8. Save, re-read the critical state, and report verified results, remaining warnings, deliberate exceptions, and anything not checked.

## Stop conditions

- Do not guess a datasheet variant, connector viewing direction, symbol pin, footprint pad, layer rule, board outline, or mechanical requirement.
- If the active document changes or the bridge disconnects, stop live edits and re-establish context.
- If a post-edit schematic netlist or PCB connectivity result contradicts intent, do not proceed or claim completion.
- If a substitute part changes peripheral values, control wiring, footprint, thermal requirements, or layout constraints, treat it as a design variant rather than a chip-only replacement.
- If reads appear stale, save, re-establish the active document, and perform one fresh read. Do not enter a blind reload/retry loop.
