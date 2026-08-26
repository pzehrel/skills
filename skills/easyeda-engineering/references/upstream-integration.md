# Upstream Integration Notes

Read this file when maintaining this skill or resolving a conflict between local and upstream EasyEDA workflows.

## Sources reviewed

- [`v0id-byte/easyeda-pro-claude-skill`](https://github.com/v0id-byte/easyeda-pro-claude-skill), `SKILL.md`, MIT license.
- [`zhoushoujianwork/easyeda-agent`](https://github.com/zhoushoujianwork/easyeda-agent), its main skill and relevant schematic workflow, wiring, layout, and automation references, MIT license.

This skill paraphrases engineering practices from those projects. It does not copy or depend on their connector, daemon, CLI, scripts, bundled block library, or this repository's `AGENTS.md`; local instructions are optional overrides rather than hidden dependencies.

## Adopted practices

- Verify health, window, project, document, and document type before mutation.
- Inspect before mutate; use small observable batches, readback, and meaningful save checkpoints.
- Treat a fresh exported netlist as schematic truth and use DRC plus data readback rather than screenshots for PCB judgment.
- Compare a baseline with post-edit state to catch virtual-looking connections, silent net merges or splits, pin-function changes, stale geometry, and persistence failures.
- Do not blindly retry an indeterminate write.
- Preserve a golden pin-to-net and NC map before reorganizing an already-wired schematic.
- Clean stale stubs, orphan ports, dangling wires, and other debris after deletion, renaming, or part replacement.
- Rebuild copper after routing when required and verify saved routing from fresh data rather than a bare success response.

## Deliberate local differences

- The official `easyeda-api` bridge remains the operation layer. The `easyeda-agent` CLI, daemon, and connector are not dependencies.
- Pagination, zone frames, and module notes are optional readability tools unless repository `AGENTS.md` requires them.
- Local schematic style prefers short direct wires and does not fan every pin out to a label.
- Power and ground flags use real non-zero stubs. A direct net-port attachment is accepted only when a fresh exported netlist proves it in the current runtime.
- Detailed PCB constraints come from the current project, datasheets, mechanical requirements, and live design rules; upstream numeric examples are not universal defaults.

## Precedence

Apply rules in this order:

1. User instructions for the current task.
2. Repository `AGENTS.md` and project-local constraints.
3. This skill's engineering invariants and verification gates.
4. Upstream observations only when they match the current API and runtime behavior.

When sources disagree, run the smallest safe live probe and decide from fresh design data, connectivity, and DRC evidence rather than screenshots.
