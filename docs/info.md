<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

This design is a small programmable dense neural-network layer: 4 neurons, each with 8 inputs. Each neuron computes

y[n] = clip( max(0, sum_i W[n][i] * x[i]) >> shift , 0, 127 )

that is, a signed multiply-accumulate, followed by ReLU, a right-shift requantisation and clipping to 7 bits.

**Activation-zero skipping.** When the current feature x[i] is zero, the MAC operand registers are held and the
accumulators are not updated, so the multipliers and adders do not toggle for that product. A counter records how
many products were skipped. The result is identical to the dense computation; only switching activity is reduced.

**Precision modes.** Weights are stored internally as signed 8-bit values. In INT8 mode each loaded byte is one
weight. In INT4 mode each byte carries two signed 4-bit weights (low nibble first), which are sign-extended on load.

**Datapath.** Features are broadcast to all neurons one per cycle through a 3-stage pipeline (operand register,
8x8 signed multiply, saturating 20-bit accumulate). One inference takes about N_IN + 4 cycles.

**Controlled baseline.** The same RTL with `ZERO_SKIP=0` and `INT4_EN=0` gives a plain INT8 dense layer for
area/activity comparison.

### Pin usage

| Pins | Function |
|---|---|
| `ui_in[7:5]` | Opcode |
| `ui_in[4:3]` | Precision: `01` = INT4, `10` = INT8 (`00`/`11` raise an error) |
| `ui_in[2]` | VALID (byte strobe, accepted on rising edge) |
| `ui_in[1]` | LAST (marks the final byte of a load) |
| `ui_in[0]` | START (resets the load/read pointer) |
| `uio[7:0]` | Data bus: host writes bytes; chip drives read data in READ/DIAG only |
| `uo_out[0]` | READY |
| `uo_out[1]` | BUSY |
| `uo_out[2]` | DONE |
| `uo_out[3]` | ERROR (sticky until CLEAR) |
| `uo_out[4]` | RD_VALID |
| `uo_out[5]` | SATURATED (an accumulator saturated) |
| `uo_out[6]` | INT4 mode active |

### Opcodes

| Code | Name | Action |
|---|---|---|
| `000` | NOP | No operation |
| `001` | CONFIG | `uio_in[3:0]` = output shift, `uio_in[4]` = zero-skip enable; precision from `ui_in[4:3]` |
| `010` | LOAD_W | Weight bytes, neuron-major order (W[0][0..7], W[1][0..7], ...) |
| `011` | LOAD_X | Feature bytes x[0..7], signed INT8 |
| `100` | RUN | Start one inference |
| `101` | READ | Each VALID returns the next byte: y[0..3], skip count, cycle count |
| `110` | CLEAR | Clear accumulators, outputs, flags and pointers (weights/features kept) |
| `111` | DIAG | Drives fixed ID byte `0xA5` on `uio_out` |

ERROR is set by: a command other than READ/DIAG/NOP while busy, loading past the end of a buffer, LAST on the
wrong byte, an invalid precision code, or READ before DONE.

## How to test

1. Reset the design (`rst_n` low for a few cycles), keep `uio_oe` released by the host.
2. Send `CONFIG` with the desired shift and zero-skip enable, and the precision on `ui_in[4:3]`.
3. Send `LOAD_W` with START on the first byte and LAST on the final byte, one byte per VALID pulse
   (hold VALID high 2 cycles, low 2 cycles).
4. Send `LOAD_X` the same way with the 8 feature bytes.
5. Send `RUN` and wait for DONE (`uo_out[2]`).
6. Stop driving `uio`, select `READ` (START on the first pulse) and pulse VALID to read y[0..3], then the skip
   count and cycle count from `uio_out`. Wait 2 cycles after leaving READ before driving `uio` again.

Quick sanity check: select `DIAG` and confirm `uio_out` reads `0xA5`.

The cocotb testbench in `test/` checks the outputs against a Python golden model for random dense and sparse
inputs in both precision modes, plus the corner cases listed in the project specification.

## External hardware

None. The design is driven from the Tiny Tapeout demo board inputs or a microcontroller on the bidirectional bus.