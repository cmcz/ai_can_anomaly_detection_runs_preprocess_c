# preprocess_c

C port of [`preprocess/`](../preprocess) for the NUCLEO-H533RE (STM32H533,
Cortex-M33). Reads raw J1939/FMS CAN frames and produces a 17-signal `float`
row every 100 ms — the input the anomaly-detection model expects — entirely in
firmware, with no Python on the board.

---

## What was ported

The Python `preprocess/` module has two sub-pipelines. Only the inference-time
path is ported; CSV loading and profiling tools stay on the PC.

| C file | Replaces | What it does |
|---|---|---|
| `can_id.h / .c` | `frames/can_id_decompose.py` | Decompose a 29-bit J1939 arbitration ID → `(priority, pgn, source_address)` |
| `spn_decode.h / .c` | `frames/spn_decode.py` | Extract a SPN bit-field from a payload → physical `float`; returns `NAN` for J1939 not-available/error markers |
| `spn_spec.h / .c` | `frames/spn_spec.py` | Compile-time decode table: 9 PGNs, 17 signals, with scale/offset/range |
| `frame_decode.h` | `frames/frame_decode.py` | Inline: decode all SPNs of one PGN in a single call |
| `signal_state.h / .c` | `features/signal_state.py` | Hold-last buffer for 17 signals; `NAN` until first frame of each signal arrives |
| `grid_sample.h / .c` | `features/grid_sample.py` | Event-driven 100 ms grid resampler; calls a callback whenever a complete row is ready |
| `preprocess.h` | — | Single umbrella header; firmware only needs this one `#include` |

**Not ported** (PC-side only):

| Python module | Reason |
|---|---|
| `frames/can_log_loader.py` | No filesystem on the board — frames come from the CAN peripheral |
| `preprocess/profile/` | Data measurement tools, not needed at inference time |

---

## Signal order

The 17-element `float row[]` produced by the grid sampler uses the same column
order as the Python `SIGNALS` list. Models trained on the PC expect this exact
layout.

| Index | Signal | PGN | Unit |
|---|---|---|---|
| 0 | `engine_speed` | EEC1 (61444) | rpm |
| 1 | `driver_demand_torque` | EEC1 | % |
| 2 | `actual_engine_torque` | EEC1 | % |
| 3 | `accel_pedal` | EEC2 (61443) | % |
| 4 | `engine_load` | EEC2 | % |
| 5 | `wheel_speed` | CCVS1 (65265) | km/h |
| 6 | `fuel_rate` | LFE1 (65266) | L/h |
| 7 | `output_shaft_speed` | ETC1 (61442) | rpm |
| 8 | `clutch_slip` | ETC1 | % |
| 9 | `input_shaft_speed` | ETC1 | rpm |
| 10 | `selected_gear` | ETC2 (61445) | gear |
| 11 | `current_gear` | ETC2 | gear |
| 12 | `tachograph_speed` | TCO1 (65132) | km/h |
| 13 | `brake_pedal` | EBC1 (61441) | % |
| 14 | `steering_angle` | VDC2 (61449) | rad |
| 15 | `yaw_rate` | VDC2 | rad/s |
| 16 | `lateral_accel` | VDC2 | m/s² |

---

## How to use it in firmware

Include the single umbrella header and create one `GridSampler` instance.
Feed every CAN frame you receive — in any order, from any ISR or task — by
calling `grid_feed()`.  When the grid has accumulated enough data to fill a
complete 100 ms row, it calls your `on_row` callback synchronously.

```c
#include "preprocess.h"

static GridSampler g_sampler;

/* Called by grid_feed() once every 100 ms of valid CAN traffic. */
static void on_row(const float row[NUM_SIGNALS], uint32_t tick_ms)
{
    /* row[0..16] is ready — pass it to the anomaly-detection model. */
    model_infer(row);
}

/* Call once at startup. */
void app_init(void)
{
    /* period_ms = 100 (10 Hz grid), max_hold_ms = 1000 (reset after 1 s gap). */
    grid_sampler_init(&g_sampler, 100, 1000, on_row);
}

/* Call from the STM32 FDCAN RX interrupt or RTOS receive task. */
void on_can_frame_received(uint32_t arb_id, uint8_t *data, uint8_t dlc)
{
    grid_feed(&g_sampler, arb_id, data, dlc, HAL_GetTick());
}
```

### Parameters

| Parameter | Value used | Meaning |
|---|---|---|
| `period_ms` | 100 | One row every 100 ms (matches the Python 10 Hz grid) |
| `max_hold_ms` | 1000 | If no frames arrive for 1 s, treat it as a recording gap: reset the hold-last state and restart the grid. Matches Python `max_hold = 1.0`. |

### What `grid_feed()` does internally

1. **Gap check** — if the bus has been silent for more than `max_hold_ms`, the
   hold-last values are stale; the state is reset to all-`NAN` and the grid
   is re-armed from the next frame.
2. **Emit elapsed ticks** — for every 100 ms tick that has elapsed since the
   last frame, if all 17 signals have been seen at least once, the current
   hold-last values are copied into a row and `on_row` is called.
3. **Decode and update** — the PGN is extracted from `arb_id`, its SPNs are
   decoded, and any non-`NAN` values overwrite the corresponding hold-last
   slots.
4. **Arm** — if this is the first frame ever (or after a reset), the next tick
   is scheduled for `now_ms + period_ms`.

`on_row` may be called zero times (signals not all seen yet, or no tick elapsed)
or more than once (multiple ticks elapsed between frames) per `grid_feed()` call.

---

## Building and running the host tests

The test suite compiles natively with `gcc` so you can verify correctness on
your development machine without a board.

```bash
cd preprocess_c/test
make test
```

Expected output:

```
=== test_can_id ===
PASS  EEC1 0x18F004E6 → pgn=61444 pri=6 sa=0xE6
PASS  PDU1 0x0C000003 → pgn=0 pri=3 sa=3
PASS  CCVS1 0x18FEF121 → pgn=65265
PASS  EEC1 with bit-31 flag masked out
All CAN ID tests passed.

=== test_spn_decode ===
PASS  engine_speed raw=1024 → 128.0 rpm
...
PASS  total SPN count == NUM_SIGNALS (17)
All SPN decode tests passed.

=== test_signal_state ===
PASS  init: all NAN, not ready
...
PASS  re-init resets all to NAN
All signal_state tests passed.

=== test_grid_sample ===
PASS  T1: no rows before all signals seen
PASS  T2: two rows emitted for t=100 and t=200
PASS  T3: gap > max_hold resets state and resumes
PASS  T4: engine_speed held across non-EEC1 frames (1000.0 rpm)
PASS  T5: frame exactly at tick emits one row
All grid_sample tests passed.

All tests passed.
```

Individual binaries can also be built and run separately:

```bash
make test_can_id   && ./test_can_id
make test_spn_decode && ./test_spn_decode
make test_signal_state && ./test_signal_state
make test_grid_sample  && ./test_grid_sample
```

---

## Cross-compiling for the board

The library has no external dependencies (only `<stdint.h>`, `<stdbool.h>`,
`<string.h>`, and `<math.h>` from the C standard library).  Add
`arm-none-eabi-gcc` flags appropriate for the STM32H533:

```bash
arm-none-eabi-gcc \
  -mcpu=cortex-m33 \
  -mfpu=fpv5-sp-d16 -mfloat-abi=hard \
  -std=c11 -Wall -Os \
  -c can_id.c spn_decode.c spn_spec.c signal_state.c grid_sample.c
```

All arithmetic uses `float` (32-bit), so every operation maps to a single
FPU instruction on the Cortex-M33.  There is no heap allocation; all state
lives in the `GridSampler` struct that the caller owns.

---

## Design decisions

| Choice | Rationale |
|---|---|
| `float` (32-bit) throughout | Matches the ONNX model input dtype; uses the hardware FPU on the STM32H533; all SPN scales fit exactly in float32 |
| `uint32_t ms` timestamps | `HAL_GetTick()` is the natural board clock; 1 ms resolution is more than enough for the 1 s gap detector; unsigned subtraction handles the 49-day wraparound |
| `RowCallback` not a ring buffer | Minimises latency and RAM; the callback runs synchronously inside `grid_feed()` |
| `NAN` as "not yet seen" sentinel | One-to-one with Python's `None`; `isnan()` compiles to a single FPU test |
| `signal_index` embedded in `SpnDef` | Avoids any string-keyed lookup at runtime; the column ordering matches the Python `SIGNALS` list at compile time |
| Linear scan over 9 PGNs | Negligible cost (≤ 9 comparisons per frame); no hash table needed |

---

## Next steps

- [ ] **Cross-compile verification** — build with `arm-none-eabi-gcc` for
      Cortex-M33 and confirm zero warnings, then check the `.map` file for
      flash/RAM usage.
- [ ] **On-board golden test** — hard-code a short sequence of real CAN frames
      (e.g. the first 200 frames from one log) into firmware flash.  Call
      `grid_feed()` for each, collect rows over UART, and compare them
      byte-for-byte against the Python `resample()` output on the same frames.
- [ ] **FDCAN HAL integration** — replace the test stub with a call from the
      STM32 `HAL_FDCAN_RxFifo0Callback` or equivalent RTOS task.
- [ ] **Port the rules layer** — `rules/instant/` (9 stateless checks) and
      `rules/rate/` (1 rate-of-change check) are the natural next C port.
      They operate on the same `float row[17]` that `grid_sample` produces.
- [ ] **End-to-end smoke test** — run the full pipeline (preprocessing → rules
      → model inference) on the board with live CAN traffic and compare scores
      against the PC ONNX Runtime output.

---

## Notes

- `can_log_loader.py` is deliberately not ported.  On the board, frames arrive
  from the CAN peripheral, not from CSV files.  If you need to replay a log
  for testing, send its frames over UART from the PC and read them back with
  a thin receive wrapper.
- The `profile/` sub-module (PGN counts, intervals, classify) is PC-side data
  analysis only and has no place in firmware.
- `grid_feed()` is **not re-entrant**.  If you call it from an ISR and also
  from a task, protect the `GridSampler` with a mutex or critical section.
- The `on_row` callback is called from within `grid_feed()`, so it must not
  block.  If the model inference takes longer than one CAN frame interval
  (~1 ms at 850 frames/s), move it to a lower-priority task and post the row
  to a queue from the callback.
