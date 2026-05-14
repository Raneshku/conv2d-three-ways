# conv2d-three-ways

A 2D convolution implemented three ways: C (reference), CUDA (GPU), and SystemVerilog (RTL). Same operation, three implementations, one set of comparable measurements. The point is to make the design-space tradeoffs across CPU, GPU, and fixed-function silicon concrete.

## What it is

Sobel edge detection on a 256x256 grayscale image, three implementations with a single golden-output check.

### 1. C reference (golden model)

Software reference in straight C. Source of truth that the other two implementations get diffed against. Design choices here (PGM image I/O, fixed-point math, saturation behavior) lock in what the RTL has to match.

### 2. CUDA: naive and tiled

- Naive: straightforward GPU kernel, one thread per output pixel.
- Tiled: shared-memory tiling that amortizes the redundant global-memory reads of the naive version.

Demonstrates the GPU memory-bandwidth-vs-compute tradeoff.

### 3. SystemVerilog streaming pipeline

- Two line buffers
- 3x3 window register array
- 9 multipliers + adder tree (MAC tree), pipelined
- Fixed-point: 8b x 8b product = 16b, 9-term sum needs 20b, saturate to 8b

A streaming RTL implementation: pixels in one per cycle, results out one per cycle after the pipeline fills.

## Verification

cocotb testbench (Python driver) feeds the same input image as the C reference, captures DUT output, and diffs pixel by pixel against the golden.

## Repo layout

```
conv2d-three-ways/
├── README.md
├── reference/
│   ├── conv2d.c
│   └── Makefile
├── cuda/
│   ├── conv2d_naive.cu
│   ├── conv2d_tiled.cu
│   ├── benchmark.cu
│   └── Makefile
├── rtl/
│   ├── conv2d.sv
│   ├── line_buffer.sv
│   ├── window.sv
│   └── mac_tree.sv
├── tb/
│   ├── test_conv2d.py
│   ├── golden_gen.py
│   └── Makefile
└── images/
    ├── input.pgm
    └── expected.pgm
```

## Toolchain

- `gcc` for the C reference
- `nvcc` for CUDA
- Verilator + GTKWave for RTL simulation and waveform viewing
- `cocotb`, `cocotb-test`, `pytest` for the Python testbench

## Status

Work in progress. Environment setup complete; C reference next.

## Results

To be filled in once measured.

| Metric | C (CPU) | CUDA naive | CUDA tiled | Verilog (sim) |
|---|---|---|---|---|
| Throughput (MP/s) | TBD | TBD | TBD | 1 px/cycle @ 100 MHz |
| First-pixel latency | low | medium | medium | ~5 cycles |
| Programmability | trivial | medium | hard | hard, fixed-function |
| Flexibility | recompile | recompile | recompile | resynthesize |
| Resource cost | 1 core | 1 GPU | 1 GPU | ~9 DSPs, 2 BRAMs |

## Tradeoff discussion

To be written after measurements. Topics:

- Why tiled CUDA is faster than naive (memory bandwidth, not compute)
- Why the RTL has the lowest latency but the least flexibility
- When silicon makes sense vs. GPU vs. CPU
- What changes if the kernel grows (5x5, 7x7) and how each implementation scales

## References

- *Programming Massively Parallel Processors* (Hwu/Kirk/El Hajj): convolution and shared-memory tiling chapters
- NVIDIA CUDA Best Practices Guide
- Verilator and cocotb documentation
