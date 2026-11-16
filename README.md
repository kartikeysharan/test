# Part 1 — Reuse Your MAC to Build a Hardware Neuron and Tiny Neural Network

**Digital Logic II | FPGA Cloud AI Acceleration | Individual work**  
**Time:** 4–6 hours  
**Tools:** Questa/ModelSim (or compatible SystemVerilog simulator), Quartus Prime Lite  
**Compilation target:** DE10-Lite MAX 10 (`10M50DAF484C7G`), 50 MHz clock (20 ns)

**Scope:** Part 1 requires simulation and Quartus compilation, **not physical FPGA programming**. In Part 2, you will use the FPGA Cloud client to run ResNet-50 on a **separate, prebuilt Agilex 7 accelerator** and study throughput, latency, batching, and concurrent inference requests. Your neuron RTL does **not** run ResNet-50.

## Overview and learning goals

In Assignment 2, you developed a parameterized **multiply-accumulate (MAC)** unit. Now you will reuse it to construct a neuron, then connect **three neurons** into a **2–2–1 network** (two hidden neurons and one output neuron). You will practice module reuse, signed fixed-point arithmetic, synchronous control, memory access, inter-module handshaking, verification, and FPGA timing/resource analysis.

![Hardware neuron architecture](docs/Neuron_Hardware_Architecture.png)

**Datapath:** weight ROM → time-shared MAC → bias addition → sigmoid lookup-table ROM. A finite-state machine (FSM) sequences the two inputs to each neuron. The diagram illustrates the architecture; the student design uses a clocked weight ROM and sigmoid ROM.

**What you should learn:** an arithmetic module can be reused without changing its interface; binary values represent signed fixed-point quantities; registers and memory reads determine when results become valid; a network layer cannot consume data before the preceding layer finishes; and synthesis/timing reports describe an implementation, not proof of hardware execution.

## 1. Starter files and required work

| Files | Required action |
|---|---|
| **`mac_core.sv` (not included)** | Import **your own Assignment 2 MAC**; complete **TODO 1** (signed mode without breaking unsigned mode) |
| `Neuron.v` | Complete **TODO 2A–2D** (MAC connections), **TODO 3** (bias), **TODO 4** (sigmoid index) |
| `Tiny_NN.v` | Complete **TODO 5A–5C** (output neuron connections) |
| `Weight_Memory.v`, `Sig_ROM.v`, `sigContent.mem`, `weights/*.mem`, `include.v` | Provided memory, sigmoid table, weights, and parameters; **do not edit** |
| `tb/tb_mac_reuse.sv` | Verify old unsigned and new signed MAC modes; add one signed edge case |
| `tb/tb_neuron.v` | Run checks; add **two** self-checking cases, including a negative input |
| `tb/tb_tiny_nn.v` | Run checks; add **two** self-checking cases **and a reset-during-operation test** |
| `Neuron.qpf`, `Neuron.qsf`, `Neuron.sdc` | Supplied Quartus project and 20 ns timing constraint |
| `MAC_IMPORT_INSTRUCTIONS.txt`, `ReLU.v` | Import guidance; optional ReLU reference (**sigmoid is required**) |

The starter is deliberately incomplete. It will not elaborate without your MAC and will not pass all tests until the TODOs are finished. Do not hard-code outputs or submit a placeholder MAC.

## 2. Reuse and extend the MAC (TODO 1)

Copy your **own** `mac_core.sv` from Assignment 2 beside `Neuron.v`, retaining an unchanged backup of the original. Preserve the original ports and the two parameters `W=8` and `ACC_W=32`; add an integer parameter **`SIGNED_MODE` with default `0`**.

```systemverilog
module mac_core #(
    parameter integer W = 8,
    parameter integer ACC_W = 32
    // TODO 1: add the SIGNED_MODE parameter (default 0)
)(
    input wire clk, reset, enable,
    input wire [W-1:0] a, b,
    output logic [ACC_W-1:0] accumulator
);
    // Keep and extend YOUR existing MAC implementation.
endmodule
```

The code above is an interface reminder, **not** a replacement for your implementation. Update the parameter-list punctuation when adding `SIGNED_MODE`.

**Required behavior:**

- **`SIGNED_MODE=0`:** original unsigned behavior for `W=8` and `W=16`; `8'hFF` means 255. Existing callers that omit `SIGNED_MODE` must still work.
- **`SIGNED_MODE=1`:** signed two's-complement multiplication for `W=8`; `8'hFF` means −1 and `8'hE0` means −32.
- Every rising edge with `enable=1` adds one full-width product. `enable=0` holds the result; synchronous active-high `reset` clears it and has priority.
- Sign-extend the signed product or zero-extend the unsigned product into the **32-bit accumulator**. Overflow wraps modulo 2³²; no saturation.

**Hint:** interpret operands as signed *before multiplication*. Making only the accumulator signed is insufficient. Preserve your original single-cycle MAC update.

**Test the MAC before continuing:**

```bash
vlib work
vlog -sv mac_core.sv tb/tb_mac_reuse.sv
vsim -c work.tb_mac_reuse -do "run -all; quit -f"
```

All three configurations (unsigned 8-bit, unsigned 16-bit, signed 8-bit) must pass. The testbench also checks reset, enable-hold, accumulation, negative products, and unsigned overflow. Add one signed edge case and explain its expected result in your report.

## 3. Understand the fixed-point format

Inputs, weights, biases, and sigmoid outputs are signed **8-bit Q3.5** numbers: five fractional bits, so each stored integer represents its value divided by 32.

$$
\text{Real value} = \frac{\text{signed raw value}}{32}
$$

| Real value | Raw integer | Hex |
|---:|---:|---:|
| +2.00 | +64 | `40` |
| +1.00 | +32 | `20` |
| +0.50 | +16 | `10` |
| +0.25 | +8 | `08` |
| 0.00 | 0 | `00` |
| −0.25 | −8 | `F8` |
| −1.00 | −32 | `E0` |
| −2.00 | −64 | `C0` |

Multiplying two Q3.5 numbers produces **Q6.10**, with ten fractional bits. The MAC accumulates raw signed products in 32 bits:

$$
S_{\mathrm{raw}} = x_{0,\mathrm{raw}}w_{0,\mathrm{raw}} + x_{1,\mathrm{raw}}w_{1,\mathrm{raw}}
$$

Before adding the signed **Q3.5 bias**, sign-extend it and shift it **left by five bits** to match Q6.10.

A neuron computes:

$$
 z=x_0w_0+x_1w_1+b,\qquad y=\sigma(z)=\frac{1}{1+e^{-z}}
$$

The provided **32-entry sigmoid ROM** approximates outputs for inputs from **−4.00 to +3.75**, spaced by 0.25. Convert the biased Q6.10 sum to a signed **quarter-step index**: `0` represents 0.00, `4` represents +1.00, `-4` represents −1.00. Use arithmetic right shifting (round toward negative infinity), and clamp the index to **−16 through +15**. The output is quantized Q3.5, not an exact real-number sigmoid.

**Example:** `x0=+1.0`, `x1=+0.5`, weights `+1.0` and `−1.0`, bias `+0.25` give `z=+0.75`. The ROM returns raw **22**, or **0.6875**.

For a negative-product check, `8'hE0` is raw **−32** (−1.0) and `8'h10` is raw **+16** (+0.5). Their signed product is raw **−512** in Q6.10, or −0.5 in real units. The resulting product must be sign-extended, not padded with zeros, when added to the 32-bit accumulator. Use this reasoning when answering Question 2.

## 4. Complete the neuron (TODO 2–4)

Open `Neuron.v`. Its controller moves through:

```text
IDLE -> READ0 -> MAC0 -> READ1 -> MAC1 -> ADD_BIAS -> LOOKUP -> DONE
```

At `IDLE`, it captures inputs and clears the MAC. `READ0/READ1` fetch weights from synchronous memory; `MAC0/MAC1` perform one MAC update each.

1. **TODO 2A–2D:** wire the supplied `mac_core` instance to clock, control, selected registered input, weight, and accumulator result. It is already parameterized for `W=8`, `ACC_W=32`, `SIGNED_MODE=1`.
2. **TODO 3:** after `MAC1`, register **MAC result + scaled bias** exactly once in `biased_sum_q610`.
3. **TODO 4:** form the signed, clamped quarter-step address for `Sig_ROM`.

The MAC completes its second update on the `MAC1` edge; `ADD_BIAS` reads that registered result on the following edge. The ROM samples its address in `LOOKUP`, and `DONE` produces `out` plus a one-cycle `outvalid` pulse. **Do not** place a second multiplier in `Neuron.v`, add bias every cycle, or replace sigmoid with ReLU.

**Think in clock edges:** a synchronous ROM does not instantly produce a newly requested weight in the same edge that requests it. Similarly, the registered MAC result is updated at a clock edge, so downstream logic should read it in a later state. Trace the state and operand signals in a waveform if you see an off-by-one result.

## 5. Connect the 2–2–1 network (TODO 5)

In `Tiny_NN.v`, two hidden neurons process the same inputs **in parallel**. The output neuron consumes their quantized results once both are valid. Complete **TODO 5A–5C** by connecting its `start`, `in0`, and `in1` ports to the supplied control/results signals.

| Neuron | Weight 0 | Weight 1 | Bias |
|---|---:|---:|---:|
| Hidden 0 | +1.00 | −1.00 | +0.25 |
| Hidden 1 | −1.00 | +1.00 | −0.25 |
| Output | +2.00 | −2.00 | −0.25 |

The intended functions are:

$$
\begin{aligned}
h_0 &= \sigma(x_0-x_1+0.25) \\
h_1 &= \sigma(-x_0+x_1-0.25) \\
y   &= \sigma(2h_0-2h_1-0.25)
\end{aligned}
$$

**Important:** use the **quantized Q3.5** hidden outputs when calculating expected network values; don't assume ideal mathematical sigmoid results. The `outvalid` and final `done` signals each pulse for one clock.

## 6. Simulate and verify

Run from the **assignment root** so `$readmemh` finds `sigContent.mem` and `weights/*.mem`.

**Questa / ModelSim:**

```bash
vlib work  # omit if your work library already exists
vlog -sv +incdir+. mac_core.sv Neuron.v Weight_Memory.v Sig_ROM.v Tiny_NN.v \
    tb/tb_neuron.v tb/tb_tiny_nn.v
vsim -c work.tb_neuron -do "run -all; quit -f"
vsim -c work.tb_tiny_nn -do "run -all; quit -f"
```

Run `tb_mac_reuse` separately using Section 2. The testbenches print PASS/FAIL and include timeouts.

**Required evidence:**

- **MAC:** all three modes PASS, with your additional signed edge case.
- **Neuron:** original checks plus two added cases (at least one negative input). Show `mac_clear`, `mac_step`, `mac_sum_q610`, `biased_sum_q610`, `lut_code`, and `outvalid` in a waveform or transcript.
- **Network:** original checks plus two added cases and an active-inference reset test. Demonstrate hidden-result handoff and final `done` pulse.

A testbench compiling successfully does **not** mean it passes. Fix functional failures before running Quartus.

**Optional alternative — Icarus Verilog:**

```bash
iverilog -g2012 -I . -s tb_mac_reuse -o mac_sim \
    mac_core.sv tb/tb_mac_reuse.sv
vvp mac_sim
iverilog -g2012 -I . -s tb_neuron -o neuron_sim \
    mac_core.sv Neuron.v Weight_Memory.v Sig_ROM.v Tiny_NN.v tb/tb_neuron.v
vvp neuron_sim
iverilog -g2012 -I . -s tb_tiny_nn -o network_sim \
    mac_core.sv Neuron.v Weight_Memory.v Sig_ROM.v Tiny_NN.v tb/tb_tiny_nn.v
vvp network_sim
```

**Troubleshooting:** If a module is missing, confirm that `mac_core.sv` is in the assignment root and the names/ports match. If ROM values appear unknown, run from the root directory and verify the `.mem` paths. If your output is one clock late, review synchronous ROM and nonblocking-assignment timing. If sigmoid is always the same value, inspect the signed `lut_code` conversion and clamp logic.

## 7. Compile and analyze timing in Quartus

The supplied project has **`Tiny_NN`** as top-level, targets MAX 10 `10M50DAF484C7G`, and constrains top-level `clk` (`core_clk`) to **20 ns / 50 MHz**. It references your imported `mac_core.sv`. No pin assignments are included: **this is not a physical board demo**.

```bash
quartus_sh --flow compile Neuron
```

Or open `Neuron.qpf` and choose **Processing → Start Compilation**. Record values from your own reports:

| Measurement | Result |
|---|---|
| Compilation success; compilation time; significant warnings | |
| Target family/device; clock period | MAX 10 / `10M50DAF484C7G`; 20 ns |
| Core-clock Fmax (MHz) and worst setup slack (ns), if reported | |
| Did the design meet 50 MHz? | |
| Logic resources and registers | |
| DSP/multiplier resources; ROM/memory usage | |

Report `N/A` and explain missing fields. Use the **user/core clock**, not JTAG/TCK. Totals may cover the whole network, ROM, and controllers. Quartus compilation is evidence of implementation/timing analysis, **not physical board execution**.

When evaluating the 50 MHz requirement, check the setup timing analysis for the specified clock and any unconstrained paths or warnings. A positive Fmax margin alone does not replace checking whether relevant timing paths were constrained. Include enough report evidence for someone else to locate your measurements.

## 8. Report questions

Answer briefly, using calculations or measured results where appropriate.

1. How did you add signed support to the old MAC while preserving its unsigned operation? What tests confirm this?
2. For `a=8'hE0` and `b=8'h10`, find their signed raw values, 16-bit product, Q6.10 real product, and 32-bit extension.
3. Why are `READ0`, `MAC0`, `READ1`, and `MAC1` separate FSM states with synchronous weight memory?
4. Why must the MAC be reset for each new inference?
5. Why must the Q3.5 bias be shifted before adding to Q6.10? Give an example.
6. Why use a sigmoid ROM? Explain quarter-step indexing and clamping.
7. How do the hidden neurons start the output neuron? Why are `outvalid` and `done` needed?
8. What logic, registers, DSP resources, and **core** Fmax does Quartus report? Does the network meet 50 MHz, and what can the report prove?
9. How might additional parallel MACs affect hardware cost, latency, and throughput?
10. **Bridge to Part 2:** How is instantiating more hardware MACs different from increasing inference requests (`nireq`) on the prebuilt Agilex 7 accelerator?

## 9. Submit to GitHub

Push to your assigned individual repository directory:

- Modified **`mac_core.sv`**, completed **`Neuron.v`** and **`Tiny_NN.v`**, and required support files.
- Enhanced neuron/network testbenches and MAC compatibility testbench.
- PASS evidence for all three testbenches, plus a neuron-stage/network-handoff waveform or labeled transcript.
- Quartus compilation, utilization, and core-clock timing reports.
- A concise **PDF report** with the Section 7 table, answers to all ten questions, one hand-worked fixed-point example, and a 2–2–1 network block diagram.

**Do not submit credentials, tokens, private keys, or passwords. Do not upload an `.sof` for Part 1.**

### Final checklist

- [ ] Own Assignment 2 MAC imported; original backed up; signed mode added without breaking unsigned mode.
- [ ] Neuron MAC, bias, and sigmoid TODOs completed; three-neuron network connected.
- [ ] All tests PASS, including added cases and reset-during-operation.
- [ ] MAX 10 compilation and **core-clock** timing/resource results recorded.
- [ ] RTL, evidence, and PDF report committed to GitHub; no credentials included.
