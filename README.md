# TALOS

**Tensor Accelerator for Low-latency On-device Systolic compute**

A parameterizable INT8 systolic-array accelerator, written in SystemVerilog and targeted at a Cyclone V SoC (Terasic Atlas-SoC / DE0-Nano-SoC). A host core issues commands to the accelerator through a small memory-mapped command interface. The host starts as the on-chip ARM Cortex-A9 and is later swapped for a RISC-V core.

> **Status: planning / early development.** Nothing in this repository has been measured on hardware yet. Every performance figure below is a placeholder (`TBD`) until it is filled in with a real result.

---

## Why this project

- Learn how a TPU-style matrix engine works by building one end to end: microarchitecture, verification, bring-up on real hardware and measurement.
- Make the accelerator host-agnostic, so the same RTL and driver contract works with an ARM core and a RISC-V core.
- Report honest numbers: achieved MACs per cycle against peak, a roofline analysis, and resource and fmax results against array size.

## Goals

1. A weight-stationary systolic array with INT8 multiplies and INT32 accumulation, generated parametrically (array size, DSP packing) rather than hardcoded.
2. Tiled matrix multiplication for matrices larger than the array, with double-buffered DMA.
3. Bit-exact agreement with a pure-integer Python golden model.
4. A host-neutral command interface: register map, descriptor format and doorbell.
5. Hardware measurements on the Atlas-SoC with the ARM as host, with the same kernel run on the A9 alone as the baseline.
6. Workloads: an MNIST MLP first, then a layer of a small transformer (e.g. stories15M) as a benchmark.
7. Stretch: a RISC-V controller (an existing open core, not a new one) with a custom instruction, and a formal proof of the sequencer.

## Architecture (draft)

```
 Host (ARM A9 or RISC-V)
   |  command queue (memory-mapped descriptors, doorbell, done/IRQ)
   v
 [Controller / sequencer] <--> [DMA <-> DDR]
   |                             |
   |             +---------------+----------------+
   |             v                                v
   |       [Weight buffer]                 [Activation buffer]
   |             |                                |
   |             v                                v
   +-----> [ N x N weight-stationary systolic array, INT8 MACs ]
                              | INT32
                              v
                  [Accumulator buffer (double-buffered)]
                              |
                              v
         [Post-process: bias, requantize, ReLU / LUT activation]
                              |
                              v
                     [Output buffer -> DMA -> DDR]
```

**Key decisions**

- **Weight-stationary array.** Weights are preloaded into the processing elements, activations flow across and partial sums flow down. The array size `N` is a parameter (start at 4, then 8 and 16).
- **Tiling.** The controller splits large matrices into tiles and accumulates partial results across K-tiles.
- **Double buffering.** The DMA loads tile *i+1* while the array computes tile *i*.
- **Host-neutral interface.** The accelerator exposes a stream or bus interface and a register map. Bus-protocol adaptation (AXI, Avalon, Wishbone) lives in thin wrappers at the edge.
- **Integer-only datapath.** Quantization follows a group-wise INT8 scheme with fixed-point scales, so results can be checked bit for bit.

## Command interface (draft, subject to change)

| Command    | Purpose                                              |
|------------|------------------------------------------------------|
| `LOAD_W`   | DMA a weight tile into the weight buffer             |
| `LOAD_A`   | DMA an activation tile into the activation buffer    |
| `MATMUL`   | Run the array over the loaded tiles                  |
| `ACTIVATE` | Apply bias, requantization and activation function   |
| `STORE`    | DMA results back to memory                           |
| `FENCE`    | Wait for all prior commands to complete              |

The register map and descriptor format will be documented in `docs/interface.md` once drafted.

## Targets

| Board                          | Role                                                         |
|--------------------------------|--------------------------------------------------------------|
| Terasic Atlas-SoC (Cyclone V SE 5CSEMA4U23C6N) | Primary target. ARM host, DDR3 via the HPS, performance measurements. |
| Digilent Basys 3 (Artix-7)     | Early prototyping and a small standalone demo (BRAM only).   |

Resource limits (DSP blocks, embedded memory) must be confirmed from the Quartus and Vivado fitter reports, not assumed.

## Repository layout (planned)

```
rtl/         SystemVerilog sources (PE, array, buffers, sequencer, wrappers)
model/       Python golden model and quantization scripts
tb/          Testbenches (randomized, with scoreboard and coverage)
sw/          C driver and host test programs (ARM and later RISC-V)
fpga/        Quartus / Platform Designer and Vivado project files
docs/        Specification, interface description, writeups
```

## Verification plan

- A pure-integer Python golden model reproduces the quantization, scaling and rounding exactly.
- Randomized testbenches compare the RTL against the golden model bit for bit.
- Corner cases: saturation, partial tiles, dimensions that are not multiples of `N`, random stream backpressure, back-to-back commands.
- Functional coverage over array size, tile shape and stall patterns.
- Stretch: formal properties on the sequencer and handshakes.

## Getting started

*To be written once the first simulation target exists.* Planned contents: required tools, how to run the Python model, how to run the simulation, and how to build and load the FPGA image.

## Results

All values are `TBD`.

| Array size | fmax (MHz) | DSP / logic / memory use | Peak MACs/cycle | Achieved MACs/cycle | Speedup vs A9 alone |
|------------|------------|--------------------------|-----------------|---------------------|---------------------|
| 4 x 4      | TBD        | TBD                      | 16              | TBD                 | TBD                 |
| 8 x 8      | TBD        | TBD                      | 64              | TBD                 | TBD                 |
| 16 x 16    | TBD        | TBD                      | 256             | TBD                 | TBD                 |

Planned plots: roofline (bandwidth against compute), throughput against array size, and the effect of double buffering.

## Roadmap

- [ ] Specification: ISA, tiling scheme, number format
- [ ] Python golden model validated against NumPy
- [ ] Single PE and 4x4 array with randomized testbench
- [ ] Buffers, accumulators and sequencer; tiled matmul in simulation
- [ ] Parameterized array; sweep of size against fmax and resources
- [ ] Board bring-up on the Atlas-SoC with the ARM as host
- [ ] Hardware measurements and roofline analysis
- [ ] MNIST MLP, then one transformer layer as benchmarks
- [ ] Stretch: RISC-V host with a custom instruction
- [ ] Stretch: formal verification of the sequencer

## References

- Jouppi et al., "In-Datacenter Performance Analysis of a Tensor Processing Unit" (ISCA 2017)
- Gemmini, the open-source systolic-array generator from UC Berkeley (for comparison of design choices)
- Cyclone V Device Overview and the Cyclone V HPS documentation (Intel)
- [I Made a TPU From Scratch](https://www.youtube.com/watch?v=21-b68eKe7k&t=385s) — the initial inspiration for this project. TALOS aims to generate its array parametrically instead of hardcoding a fixed size such as 8x8.

## Name

Talos is the bronze automaton of Greek myth that guarded Crete: a machine acting on commands rather than on its own initiative, which suits an accelerator driven by a host core.

## Licence

To be decided.
