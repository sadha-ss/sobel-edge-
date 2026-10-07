# Sobel Edge Detection — FPGA/VHDL Submission Package

## 1. Project objective

This project implements the IMT Atlantique Case Study VI Sobel edge-detection accelerator in two domains:

- **Python**: software reference using exact Euclidean gradient magnitude.
- **VHDL**: streaming hardware accelerator using the hardware-friendly Manhattan approximation `|Gx| + |Gy|`.

The architecture follows the case-study decomposition and includes an explicit two-clock arithmetic drain after the last input pixel so the final pipelined result is preserved:

1. two row line buffers,
2. a 3×3 sliding-window extractor,
3. Sobel Gx/Gy computation,
4. Manhattan magnitude computation,
5. a finite-state controller,
6. a top-level integration,
7. a self-checking testbench.

The case study specifies a 100×100, 8-bit grayscale stream and one pixel per clock. The controller drains the two registered arithmetic stages after the final input pixel before completing the frame. It also explicitly asks for comparison of Euclidean Python results against the Manhattan VHDL result using MAE.

## 2. Repository layout

```text
Sobel_Edge_Detection_Submission/
├── rtl/
│   ├── add_sub_n.vhd
│   ├── linebuffer.vhd
│   ├── magnitude_calc.vhd
│   ├── sobel_fsm.vhd
│   ├── sobel_gradients.vhd
│   ├── sobel_pkg.vhd
│   ├── sobel_system.vhd
│   └── window_extractor_3x3.vhd
├── tb/
│   └── tb_sobel_system.vhd
├── python/
│   ├── prepare_image.py
│   ├── compare_results.py
│   └── self_test.py
├── vivado/
│   └── VIVADO_SETUP.md
├── docs/
│   └── REPORT.pdf
├── input_gray.png
├── imagesrc.txt
├── python_ref.txt
├── expected_vhdl_output.txt
├── edges_python.png
├── edges_manhattan_reference.png
├── README.md
└── requirements.txt
```

## 3. Important design decision: valid 3×3 interior

A 3×3 Sobel operator cannot produce a mathematically complete window for the outermost one-pixel border without an explicit border policy.

This implementation therefore emits **98×98 = 9604 valid interior pixels** from the 100×100 frame. In the 100×100 reconstructed comparison image, the outer border is zero.

Mapping:

- hardware output #0 = image pixel center `(row=1, col=1)`
- hardware output #1 = `(1,2)`
- ...
- last output = `(98,98)`

This makes the hardware stream directly comparable to a software Sobel implementation with zero-valued borders.

## 4. Sobel equations

```text
Gx = (p02 + 2*p12 + p22) - (p00 + 2*p10 + p20)

Gy = (p00 + 2*p01 + p02) - (p20 + 2*p21 + p22)
```

Python:

```text
G = sqrt(Gx² + Gy²)
```

VHDL:

```text
G_hw = |Gx| + |Gy|
```

The VHDL result is 11 bits because an 8-bit Sobel gradient can reach ±1020 and the Manhattan sum can reach 2040.

## 5. Why this version is cleaner than the original skeleton

The supplied case-study skeleton intentionally contains unfinished student sections. This submission fills those sections and also fixes integration issues:

- complete line-buffer shifting and reset;
- complete 3×3 horizontal shifts;
- complete Sobel adder trees;
- complete signed absolute-value logic;
- complete magnitude pipeline;
- complete FSM and pipeline-valid alignment;
- complete top-level wiring;
- remove machine-specific absolute file paths from the testbench;
- make the testbench stop only after the complete frame;
- count and assert the expected 9604 hardware outputs;
- include a deterministic Python golden reference;
- include an exact Manhattan-reference comparison in addition to the required Euclidean MAE.

## 6. Exact Vivado procedure

See `vivado/VIVADO_SETUP.md`.

Short version:

1. Open Vivado.
2. Create a new RTL Project.
3. Add every file in `rtl/` as **Design Sources**.
4. Add `tb/tb_sobel_system.vhd` as a **Simulation Source**.
5. Select **VHDL 2008** as the simulation language if required by your Vivado configuration.
6. Set `tb_sobel_system` as the simulation top.
7. Copy `imagesrc.txt` into the XSim working directory, normally:
   `your_project.sim/sim_1/behav/xsim/`
8. Run **Run Simulation → Run Behavioral Simulation**.
9. Let the simulation finish. The testbench reports:
   `PASS: frame complete; output count = 9604`
10. The simulator creates `SobelEdgeVHDL.txt`.
11. Run:

```bash
python python/compare_results.py --vhdl "PATH_TO/SobelEdgeVHDL.txt" --image input_gray.png --expected expected_vhdl_output.txt --outdir .
```

A correct hardware simulation must report:

```text
Exact Manhattan mismatches: 0
```

The Euclidean MAE is normally non-zero because the case study intentionally uses different magnitude metrics in Python and VHDL.

## 7. Regenerating the image files

Install:

```bash
pip install -r requirements.txt
```

Then:

```bash
python python/prepare_image.py --input your_image.png --outdir .
```

For a different image, replace the supplied `imagesrc.txt` and golden reference files with the newly generated ones before running XSim.

## 8. Static sanity check

```bash
python python/self_test.py
```

This checks that all required RTL files exist and that no unfinished student placeholders remain.

## 9. What is and is not claimed

The package contains a completed RTL implementation and a deterministic Python golden model. The final HDL compile/simulation must still be run in the student's installed Vivado/XSim environment because the exact FPGA part, Vivado installation, and simulator are machine-specific.

Do not claim FPGA resource utilization, maximum frequency, or board-level operation unless you have actually run synthesis/implementation on the target FPGA.
