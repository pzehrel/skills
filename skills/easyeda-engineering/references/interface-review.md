# Interface and Pin Review

Read this reference before wiring or approving connectors, headers, transceivers, active-low handshake signals, or substitute ICs.

## Build a pin table first

For every external connector, record:

| Field | Meaning |
| --- | --- |
| Physical pin | Number defined by the target equipment manual |
| Equipment signal | Exact signal name in that manual |
| Direction | Input or output from the equipment's perspective |
| Source and sink | Which device drives the line and which device receives it |
| Schematic endpoint | Symbol reference and pin name/number |
| Footprint pad | Physical PCB pad number |
| Net name | The single intended schematic network name |
| Disposition | Connected, shared, pulled, or intentionally NC |
| Verification | Fresh netlist evidence and PCB pad-to-net evidence |

Do not wire or approve the connector until every used pin has an unambiguous row.

## Connector orientation

- Confirm whether the manual shows the mating face, cable side, component side, or solder side.
- Verify pin 1, row numbering, odd/even columns, keying, missing pins, mirroring, and connector rotation independently.
- Compare symbol pin numbers with footprint pad numbers; do not infer either from screen position.
- After placement, replacement, or schematic-to-PCB synchronization, read back the actual symbol pins, footprint pads, net membership, and designator.

## Direction-sensitive serial interfaces

- Determine whether each endpoint is DTE or DCE before mapping signals.
- Connect a driver output to a receiver input. Same-name connections are not automatically correct in DTE-to-DTE or null-modem wiring.
- Treat active-low suffixes as logic semantics, not decoration.
- When one physical signal feeds several receiver inputs, use one unambiguous net name and list every member pin.
- When both endpoints expose only inputs and no device drives the line, leave it NC unless the design explicitly requires a fixed level.

## Substitute parts

Verify symbol functions, package pads, supply and logic thresholds, peripheral values, enable or shutdown behavior, exposed-pad requirements, data rate, loading, ESD assumptions, thermal needs, and PCB layout constraints.

If any of these change, document an alternate population or circuit variant. After a swap, prove that the target net gained the intended new pin or pad and that the old net no longer contains it.
