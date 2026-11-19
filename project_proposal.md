# Project Proposal: Two-Channel, Two-Band Vibration Analysis Accelerator

**Project team:** Darren Kuo, Andy Deng, Khaliduzzaman Mredul

**Graded course initial plan:** [plans/initial_plan.md](plans/initial_plan.md)

## Goal

Build and simulate a small custom IP accelerator that classifies the dominant vibration-frequency band for each of two independent input channels. The project emphasizes streaming digital signal processing, fixed-point arithmetic, parallel multiply-accumulate units, and modular hardware verification, without relying on a physical FPGA board.

## Initial scope

| Item | Proposed baseline |
| --- | --- |
| Channels | 2, interleaved input samples |
| Bands | Low (example: 20–90 Hz) and high (example: 160–240 Hz) |
| Sample rate used in demo | 1 kHz |
| FIR structure | 16 taps per band; 4 MAC lanes in each of two parallel engines |
| Window | 128 samples per channel |
| Numerical format | Signed 16-bit Q1.15 samples/coefficients, wide accumulators |
| Host I/O | AXI4-Stream input and output |
| Outputs | Low-band energy, high-band energy, dominant band for each channel |
| Demonstration | Python stimulus and golden model + SystemVerilog simulation |

## Basic operation

1. The PS or simulation testbench streams a framed block of interleaved samples for channels 0 and 1.
2. The IP broadcasts each sample to independent low- and high-band FIR engines; each engine maintains separate histories for the two channels.
3. Each filtered sample is squared and accumulated into the appropriate channel/band energy register.
4. After 128 samples per channel, the IP compares the two energies per channel and returns both measured energies and the dominant band.

The fixed-point model is a 16-tap FIR (`y[n] = sum(h[k]*x[n-k])`) followed by windowed energy (`E = sum(y[n]^2)`). See [the initial plan](plans/initial_plan.md) for equations, data formats, module descriptions, and a complete interface table.

## Why hardware?

Each input sample requires repeated multiplications and additions with a regular access pattern. Hardware allows the two filters to run concurrently and four taps in each band to be processed in parallel, while pipelining the energy accumulation and avoiding repeated host reads of prior samples.

## Planned demo (simulation only)

We will generate 50 Hz and 200 Hz example vibrations, plus mixed and silent cases, using Python. An RTL testbench will feed the same quantized input into the IP and compare output energies against a bit-accurate Python model. A final plot/table will show the input signals, measured energy in each band, the predicted band, and any RTL/reference mismatches. Hardware performance metrics will be reported only after actual simulation and synthesis; no speedup is claimed in advance.

## Architectural tradeoff

We considered a single-MAC filter (small but slower), a fully parallel 16-tap filter (fast but larger), and the selected four-lane-per-filter structure (a moderate baseline). The first milestone is a correct single-channel FIR and energy path; the full milestone is two channels with two parallel frequency-band filters and AXI4-Stream framing.

## Project status

**Planning stage.** This document describes proposed functionality; RTL implementation and measured results have not yet been completed.
