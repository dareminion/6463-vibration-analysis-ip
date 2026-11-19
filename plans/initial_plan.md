# Initial Plan: Two-Channel, Two-Band Vibration Analysis Accelerator

## Project Team

- Darren Kuo
- Andy Deng
- Khaliduzzaman Mredul

## IP Definition

### What the IP does

Our team will design a custom hardware IP block that analyzes two streams of vibration-sensor samples and determines whether **low-frequency** or **high-frequency** vibration is dominant in each channel. Possible applications include basic machine-condition monitoring and identifying changes in a vibration source. The IP is an **analysis engine**, not a physical sensor interface: the processing system (PS) supplies digitized samples, and the IP returns energy measurements and a band classification for each channel.

We will begin with **two channels, two frequency bands, 16-tap FIR filters, and windows of 128 samples per channel**. For a demonstration, the PS will be emulated by a simulation testbench and Python will generate signals sampled at 1,000 samples/second. Example test tones are 50 Hz (low band) and 200 Hz (high band). Bandpass FIR coefficients will be generated offline in Python for example bands of approximately 20–90 Hz and 160–240 Hz. These are initial demonstration parameters, not adjustable hardware requirements.

### Mathematical operations

For channel `c` (0 or 1), band `b` (low or high), and sample index `n`, the FIR filter computes:

\[
y_{c,b}[n] = \sum_{k=0}^{15} h_b[k] \, x_c[n-k].
\]

Here, `x_c` is the signed input sample and `h_b` is the set of 16 coefficients for band `b`. Each channel has its **own 16-sample history**; samples from different channels must never be mixed. Each new frame begins with zero-filled filter history.

Over a window of `N = 128` filtered samples, the energy accumulator computes:

\[
E_{c,b} = \sum_{n=0}^{127} y_{c,b}[n]^2.
\]

The output classification for channel `c` is `LOW` when `E[c,low] > E[c,high]`, `HIGH` when the reverse is true, and `TIE` when they are equal. The IP returns **both energy values as well as the classification**, so the PS can inspect the actual measurements.

The first implementation will use **signed 16-bit Q1.15 inputs and FIR coefficients**. Products are accumulated at higher precision (at least 40 signed bits), then rounded, shifted by 15, and saturated back to signed 16-bit filtered samples. Squared filtered samples accumulate into **unsigned 48-bit energy registers**, wide enough for 128 maximum-magnitude 16-bit samples. The Python golden model will reproduce these same integer operations, including rounding and saturation.

Algorithmic reference (one frame; `fir_q15` includes the defined fixed-point rounding and saturation):

```python
history = [[0] * 16 for _ in range(2)]
energy = [[0, 0] for _ in range(2)]  # [channel][low/high]

for channel, sample in interleaved_samples:  # 128 samples per channel
    history[channel] = [sample] + history[channel][:-1]
    for band in (0, 1):
        y = fir_q15(history[channel], coeff_q15[band])
        energy[channel][band] += y * y

classification = [
    "LOW" if e[0] > e[1] else "HIGH" if e[1] > e[0] else "TIE"
    for e in energy
]
```

### Why these operations suit hardware acceleration

FIR filtering is a repeated multiply-accumulate operation with a fixed, regular memory-access pattern. **The low-band and high-band filters can run concurrently**, and the four coefficient multiplications in each of our planned MAC groups can run in parallel. The energy stage is also a regular multiply-and-accumulate pipeline. Channel-local delay-line memory allows samples to be reused rather than fetched repeatedly from the PS. These properties make the processing deterministic and amenable to pipelining, while freeing a host processor from repeatedly filtering each sample in software.

### How the IP interacts with the PS

The host sends a **32-bit AXI4-Stream input frame** containing a header (`frame_id`, operation, and window length) followed by 256 digitized samples, interleaved as channel 0, channel 1, channel 0, channel 1, and so on. Each sample word carries a signed 16-bit sample and a 1-bit channel ID; `TLAST` marks the last sample of the frame. The initial design supports a fixed window length of 128 samples per channel and fixed coefficients held in on-chip ROM/BRAM. No external DDR access or live ADC is needed.

After processing, the IP returns **two result records** on a 32-bit AXI4-Stream output, one per channel. Each record carries the `frame_id`, `channel_id`, dominant-band classification, and the low- and high-band energies (each split over two 32-bit words). `TLAST` marks the end of the response frame. A controller handles valid/ready backpressure and frame boundaries. In simulation, a SystemVerilog testbench acts as the PS; an optional future FPGA integration could connect these streams through an AXI DMA.

## IP Architecture

### Sub-modules

1. **Input/command decoder and channel router.** Parses the AXI4-Stream frame header, checks the input length/channel IDs, and tags samples with `frame_id`, `channel_id`, and window boundaries. It broadcasts each accepted sample to both filter paths via local FIFOs. It only removes a sample from the input stream when **both** downstream FIFO paths can accept it, so neither band loses data.

2. **Low-band FIR engine.** Stores 16 fixed low-band coefficients in local ROM/BRAM and maintains a separate 16-sample delay line for each channel. Four parallel multiply-accumulate lanes process four taps per cycle, taking four MAC groups to cover all 16 taps (plus any pipeline/control overhead). It outputs a signed filtered sample and the associated channel/frame tag.

3. **High-band FIR engine.** Has the same architecture but different stored coefficients targeting the higher band. It runs independently of the low-band engine, enabling both bands to process the same sample concurrently. It also keeps separate histories for the two channels.

4. **Energy accumulation units (one per band).** Each receives filtered samples from its FIR engine, squares the signed 16-bit value, and adds the result into a separate 48-bit energy register for each channel. After 128 samples for a channel, it emits the completed `{frame_id, channel_id, band_id, energy}` token and clears that channel's accumulator for the next frame.

5. **Band comparator and output formatter.** Collects the low- and high-band energy tokens for each channel, compares them, and packages the two channel result records into the 32-bit AXI4-Stream response. It waits for the matching `frame_id` and `channel_id` energies so data cannot be paired incorrectly.

### Interfaces

| From → To | Interface | Information carried |
| --- | --- | --- |
| PS / testbench → Input decoder | **AXI4-Stream**, 32-bit `TDATA`, `TVALID`, `TREADY`, `TLAST` | Frame header `{opcode, frame_id, window_length}` followed by interleaved `{channel_id, signed_sample[15:0]}` words; `TLAST` on final sample |
| Input decoder → Low-band FIR | **FIFO / ready-valid stream** | `{frame_id, channel_id, sample, start_of_frame, end_of_channel_window}` tokens |
| Input decoder → High-band FIR | **FIFO / ready-valid stream** | Identical copy of the sample and boundary tokens, with synchronized acceptance into both FIFOs |
| FIR engines ↔ local delay lines and coefficient storage | **Local shared memory / registers** | Two per-channel 16-sample histories and one 16-coefficient table per band; no PS memory transactions during filtering |
| Each FIR engine → its energy accumulator | **FIFO / ready-valid stream** | `{frame_id, channel_id, filtered_sample, window_end}` |
| Each energy accumulator → comparator | **FIFO / ready-valid stream** | `{frame_id, channel_id, band_id, energy[47:0]}` once per channel/window |
| Comparator → PS / testbench | **AXI4-Stream**, 32-bit `TDATA`, `TVALID`, `TREADY`, `TLAST` | Per-channel `{frame_id, channel_id, classification, E_low, E_high}` result records; `TLAST` on final result word |
| Input decoder → processing stages | **Control signals / frame-boundary tokens** | Reset per-frame delay lines and energy state, and propagate validity/error status |

This split keeps the I/O protocol, FIR arithmetic, energy reduction, and final decision logic independently testable. It also exposes parallelism **between bands** and **within the four MAC lanes** without requiring four fully separate 16-tap filters.

### Design alternatives considered

- **Single reused MAC per FIR:** simplest and lowest arithmetic-resource use, but approximately 16 tap cycles per incoming sample per band.
- **Fully parallel 16-tap FIR per band:** potentially much higher sample throughput, but requires more parallel multipliers and adders.
- **Chosen baseline — four MAC lanes per FIR:** balances moderate complexity and resource use with parallel filtering of both frequency bands. If time permits, we can compare it against one of the other choices in synthesis.

### Verification and demonstration plan

We will first validate each FIR engine against a fixed-point Python golden model, then verify the energy accumulator and result formatter separately. An integrated SystemVerilog simulation will send known 50 Hz and 200 Hz signals, mixed-frequency inputs, silence, and frame-boundary/backpressure cases. The demo will plot the generated inputs and compare hardware-computed energies/classifications with the Python reference. We will report measured cycle counts and, if synthesis is available, estimated clock timing and LUT/FF/DSP/BRAM use. **No physical FPGA board is needed** for this initial demonstration.

The implementation details above are initial architectural targets. We will refine exact register packing, cycle timing, and coefficient quantization in the later detailed design plan.
