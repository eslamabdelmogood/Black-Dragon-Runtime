# Black Dragon Runtime
### The world's first Distributed Industrial Nervous System, optimized for Arm

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Track:** Physical AI (edge sensing, embedded inference, real-time control for physical/robotic structures)

Black Dragon Runtime is the engine behind the **Optical Skin** digital twin
below: a distributed sensing + spiking-reflex + cognitive-control stack for
physical structures, built to live on Arm — from a Cortex-M + Ethos-U edge
node up to an Arm64 server. Four pieces, each with real, runnable evidence
behind it:

| Pillar | What it shows | Where |
|---|---|---|
| **Optical Skin demo** | Dense distributed sensing vs. sparse fixed sensors, live, interactive | [`docs/optical_skin_demo.html`](docs/optical_skin_demo.html) |
| **Reflex Kernel Benchmark** | Polling vs. interrupt vs. event-driven reflex kernel — reaction time, not just throughput | [`REFLEX_KERNEL_BENCHMARK.md`](REFLEX_KERNEL_BENCHMARK.md) |
| **Real NPU pipeline** | INT8 → Arm Ethos-U → NPU → latency, compiled with Arm's actual Vela compiler | [`NPU_PIPELINE.md`](NPU_PIPELINE.md) |
| **Arm optimization benchmark** | Naive Python → vectorized → float32 → int8, memory + speed, honestly reported | [`ARM_OPTIMIZATION.md`](ARM_OPTIMIZATION.md) |

## Project Overview

Most structural health monitoring (SHM) today is a handful of fixed point
sensors (10-15 thermocouples/strain gauges/accelerometers) bolted to a
structure, each independently comparing its reading to a fixed threshold.
That's cheap, but it's blind between sensors, slow to react, and can't tell
*where* a problem is with any precision.

Black Dragon Runtime replaces that with a **distributed optical "skin"**: a
dense simulated Fiber Bragg Grating (FBG) mesh reconstructed with
inverse-distance-weighted interpolation, feeding an **event-driven
spiking-neuron reflex layer** (fast, local, always-on anomaly detection —
the kind of workload that belongs on an Arm edge core, not a cloud
round-trip), which in turn feeds a small three-part cognitive system:

- **Bat** — short-horizon forecaster that extrapolates temperature, stress,
  and damage trends to estimate time-to-failure (Remaining Useful Life).
- **Hermit Crab** — a stability evaluator that vetoes actions whose
  long-term cost outweighs their short-term relief (e.g. it always vetoes
  "ignore and hope").
- **Squid** — adaptively re-weights Productivity / Safety / Energy /
  Structural-Integrity objectives as risk rises and falls, driving the
  actuator's final decision (reduce speed / redistribute load / isolate
  zone / do nothing).

This is a **simulation-first digital twin**: there's no physical panel, but
every layer — the finite-difference thermal/stress/fatigue physics, the
sensor models, the spiking reflex kernel, and the cognitive layer — is real,
runnable code, not a mockup.

**What makes it interesting, and why it should win:** this isn't "AI for
physical systems" as a slogan — it's a full nervous system, from sensing
skin to reflex to cognition, run through Arm's actual toolchain at every
layer that matters:

- The reflex kernel is compiled with Arm's own **Ethos-U Vela compiler**
  into a real NPU artifact — 100% NPU-resident, 42.9 µs/inference,
  compiler-verified, not estimated (`NPU_PIPELINE.md`).
- It's benchmarked not just for throughput but for **reaction time** —
  polling vs. interrupt vs. reflex kernel — because for a physical safety
  system, *how fast you notice* matters as much as *how many tokens/sec*
  (`REFLEX_KERNEL_BENCHMARK.md`).
- It's honestly optimized for **size** (8x memory reduction via int8
  quantization, 99.8% decision agreement with the float64 reference) with
  no inflated speed claims where the hardware to back them up isn't
  present (`ARM_OPTIMIZATION.md`).
- And it's demonstrable, live, in a browser, with no backend
  (`docs/optical_skin_demo.html`).

## Functionality / Output

Running the pipeline produces:

1. **`outputs/metrics.csv`** — a 12-row comparison (4 fault scenarios x 3
   systems: BHS / fixed-point-sensor baseline / unmitigated) across 8
   metrics: detection time, predictive lead time, localization error (in
   grid cells), false/true alarm counts, RUL prediction error, damage
   prevented (%), reaction latency, and simulation wallclock cost.
2. **`outputs/arm_benchmark.json`** — the reflex-kernel benchmark: ms/step,
   resident state memory in bytes, and detection-agreement rate, for each
   of the four implementations described above.
3. **`docs/dashboard.html`** — a single static HTML file (no server, no
   external assets) that bakes both of the above into readable tables and
   bar charts. Open it directly in a browser.

The four fault scenarios (`src/bhs/scenarios.py`) are:

| Key | Scenario |
|---|---|
| A | Localized thermal fault |
| B | Crack initiation and propagation |
| C | Mechanical overload |
| D | Combined heat + vibration + stress failure |

Sample result (your numbers will vary slightly run to run; see the CSV for
exact figures): on Scenario A, BHS detects the fault at **t=0.25s** with
**0-cell localization error** and prevents **~69% of the damage** an
unmitigated panel would accumulate; the fixed-sensor baseline detects at
**t=6.15s**, is off by **~12 cells**, and prevents **~36%**.

## Setup Instructions

Runs anywhere Python 3.9+ and NumPy run — including natively on Arm64
(Raspberry Pi, AWS Graviton, Apple Silicon, Arm-based CI runners) with no
code changes.

```bash
git clone <this-repo-url>
cd bhs-optical-skin

python3 -m venv .venv
source .venv/bin/activate        # on Windows: .venv\Scripts\activate

pip install -e .                 # installs the `bhs` package + numpy

# 1. Run the fault-scenario comparison (BHS vs baseline vs unmitigated)
python scripts/run_scenarios.py --steps 2000 --out outputs/metrics.csv

# 2. Run the Arm-optimization benchmark for the reflex kernel
python -m bhs.optimize.benchmark --steps 300 --out outputs/arm_benchmark.json

# 3. Bake both into the static dashboard
python scripts/build_dashboard.py
open docs/dashboard.html         # or just double-click it

# 4. Reaction-time benchmark: Polling -> Interrupt -> Reflex Kernel
python scripts/reaction_time_benchmark.py --trials 30 --noise-std 7.0

# 5. Real NPU pipeline: INT8 -> Ethos-U Vela compiler -> NPU performance report
pip install -r requirements-npu.txt
python scripts/build_npu_model.py --accelerator ethos-u55-256

# 6. Open the live Optical Skin demo directly in a browser (no build step)
open docs/optical_skin_demo.html

# 7. Run the test suite
pip install pytest
pytest tests/ -v
```

### Validating on an Arm-powered device

```bash
# On a Raspberry Pi / AWS Graviton instance / any arm64 host:
git clone <this-repo-url> && cd bhs-optical-skin
python3 -m venv .venv && source .venv/bin/activate && pip install -e .
python -m bhs.optimize.benchmark --steps 500 --out outputs/arm_benchmark_arm64.json
```
Compare the printed `platform.machine()` field (should read `aarch64` /
`arm64`) and the resulting ms/step + state-bytes numbers against
[`ARM_OPTIMIZATION.md`](ARM_OPTIMIZATION.md). Full instructions for Docker
buildx cross-builds and Arm Performix integration are in that document too.

## Repository layout

```
src/bhs/
  physics.py         # finite-difference thermal/stress/fatigue substrate
  sensing.py          # OpticalSkin (FBG+IDW) and PointSensorBaseline
  reflex.py           # vectorized LIF spiking reflex kernel (reference)
  cognition.py        # Bat / Hermit Crab / Squid
  scenarios.py         # fault scenarios A-D
  simulate.py          # per-scenario, per-architecture simulation runner
  metrics.py           # 8-metric comparison + CSV writer
  optimize/
    naive_reflex.py     # unoptimized reference implementation (benchmark "before")
    quantized_reflex.py # int8-quantized Arm-edge implementation
    benchmark.py         # naive vs float64 vs float32 vs int8 benchmark harness
scripts/
  run_scenarios.py           # CLI: run all scenarios -> metrics.csv
  build_dashboard.py         # CLI: bake metrics + benchmark into dashboard.html
  reaction_time_benchmark.py # CLI: Polling vs Interrupt vs Reflex Kernel reaction time
  build_npu_model.py         # CLI: INT8 -> Ethos-U Vela compiler -> NPU report
tests/                 # pytest suite
docs/
  dashboard.html            # generated results dashboard (open in a browser)
  optical_skin_demo.html    # live interactive demo: dense skin vs sparse sensors
  npu_artifacts/            # committed sample Ethos-U-compiled models + Vela reports
ARM_OPTIMIZATION.md          # Python -> NumPy -> float32 -> int8 optimization write-up
NPU_PIPELINE.md               # INT8 -> Ethos-U -> NPU -> latency, real compiler output
REFLEX_KERNEL_BENCHMARK.md    # Polling -> Interrupt -> Reflex Kernel reaction-time study
```

## Notes on scope

This is a simulation of a distributed SHM system and its optimization
pipeline, not a certified aerospace/industrial monitoring product — the
physics (fatigue law constants, thermal diffusivity, etc.) are illustrative,
not calibrated to a real material. The Arm-optimization work (vectorization,
float32, int8 quantization, and the benchmark harness) is real, runnable,
and the numbers in `ARM_OPTIMIZATION.md` are exactly reproducible with the
commands above.
