# Two-Channel, Two-Band Vibration Analysis Accelerator

Advanced Hardware Design course project by **Darren Kuo, Andy Deng, and Khaliduzzaman Mredul**.

This repository will contain a custom streaming hardware IP for classifying low- versus high-frequency vibration energy in two input channels. The planned design uses fixed-point 16-tap FIR filters, per-channel energy accumulators, and AXI4-Stream framed I/O. The demonstration will run using Python-generated test signals and RTL simulation; a physical FPGA is not required.

## Planning documents

- [Initial plan](plans/initial_plan.md): Project definition, mathematical operations, hardware architecture, and PS/IP interfaces.
- [Project proposal](project_proposal.md): Overview of the project, scope, and planned demonstration.
