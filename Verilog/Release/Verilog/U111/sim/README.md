# U111 bus sizing testbench

`tb_u111_split.v` drives `U111_CYCLE_SM.v` through long word reads from a
16 bit off-board port, the cycles U111 splits into two word reads, and
checks what the CPU would make of them.

The mainboard is modelled as a one clock `_TACK` pulse launched `TCO_MB`
after a bus clock edge (rising for U409/U712, falling for U110 when
`NEGEDGE=1`) and delayed `TRACE` on the way to U111, driven high one clock
later and released after another, the way `U110_CYCLE_TERMINATION.v` does
it. `PULSE` widens the pulse for a rise that is slower than the fall.

The CPU model starts its cycles back to back, samples `_TA` from the end of
C2 on and only accepts it when it is low from 8ns before to 2ns after the
edge (MC68040 specs 22a and 23 at 40MHz). U111's outputs reach the CPU
after `TA_PATH` (pad to pad) and `U111_TCO` (register to pad). Each read
must be acknowledged only after both halves were, with the assembled long
word stable through the CPU's data setup and hold window. The second half
of the run puts an on-board read, terminated by a U400 style `_TA` pulse
that U111 forwards to `_TACK`, in front of every off-board read.

    iverilog -g2012 -o tb tb_u111_split.v ../U111_CYCLE_SM.v && vvp tb
    ./run_sweep.sh

`U111_CYCLE_SM.v` has its registers declared ahead of their first use so
that Icarus Verilog accepts it; Synplify did not mind the original order.

## What the sweep shows

Rising edge acknowledges (U409, U712) reach the CPU `TCO_MB + TRACE +
TA_PATH` after the edge and have to make the CPU's 8ns setup at the next
edge: the sum must stay under 17ns, and the bench passes with the default
6 + 2 + 8. Past that the CPU misses the pulse and the cycle never
terminates.

Falling edge acknowledges (U110) have half a clock in hand at the CPU and
pass for every `TCO_MB` except two. At `TCO_MB + TRACE` = 12.5 the pulse
edges coincide with U111's own sampling edges, a simulation race rather
than a design fault, but it marks the alignment at which U111's sampling
of the pulse has no margin. At the fast end the pulse must not be gone
before the CPU's 2ns hold: `TCO_MB + TRACE + TA_PATH` has to stay above
14.5ns, and the bench fails at 4 + 2 + 8.

The bench was written to test a `_TACK` stretch in U111 that held `_TA`
low for an extra clock. With `NEGEDGE=1 PULSE=1.5` and `TCO_MB` from 9.25
to 10 that stretch spanned the release of the split cycle's second half
and terminated every long word read early with half the data; the plain
pass through does not.
