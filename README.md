# Two-Channel, Two-Band Vibration Analysis Accelerator

Advanced Hardware Design course project by **Darren Kuo, Andy Deng, and Khaliduzzaman Mredul**.

This repository will contain a custom streaming hardware IP for classifying low- versus high-frequency vibration energy in two input channels. The planned design uses fixed-point 16-tap FIR filters, per-channel energy accumulators, and AXI4-Stream framed I/O. The demonstration will run using Python-generated test signals and RTL simulation; a physical FPGA is not required.

## Planning documents

- **[Initial plan — submit this file](plans/initial_plan.md)**: Complete course initial-plan submission, including mathematics, modules, and PS/IP interface table.
- [Project proposal](project_proposal.md): A shorter overview of the project, scope, and planned demonstration.

## Status

Initial planning only. Hardware source code, testbenches, and performance measurements will be added after the plan is reviewed.

## Course references

- [Official project planning instructions](https://sdrangan.github.io/hwdesign/docs/projects/planning.html)
- [Example initial plans](https://sdrangan.github.io/hwdesign/docs/projects/example_plans/)

## Submitting the initial plan

1. Create a **public** GitHub repository and add these files at the paths shown.
2. Commit and push to the `main` branch.
3. Submit the URL to `plans/initial_plan.md`, for example:

   `https://github.com/YOUR_USERNAME/YOUR_REPO/blob/main/plans/initial_plan.md`

4. Set the course grader timeout to **at least 100 seconds**.

Only replace `YOUR_USERNAME` and `YOUR_REPO` after creating the real repository.
