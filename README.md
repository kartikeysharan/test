# Part 1 — From Your MAC to a Hardware Neural Network

**Digital Logic II · Individual assignment · Approximately 4–6 hours**  
**Target:** DE10-Lite MAX 10 (`10M50DAF484C7G`) · 50 MHz constraint  
**Tools:** SystemVerilog simulator and Quartus Prime Lite

## Objective

Extend your MAC from the previous assignment and use the **hardware architecture developed in class** to complete a small **2–2–1 neural network**. This part focuses on module reuse, hierarchy, timing, simulation, and FPGA resource analysis—not on training a neural network.

![Hardware neuron architecture](docs/Neuron_Hardware_Architecture.png)

The supplied starter project is intentionally incomplete. **The lecture and starter RTL establish the required architecture, module interfaces, and integration rules.** Follow those materials rather than replacing the design with a different implementation. Be prepared to explain your connections and the order in which the hardware operates.

## Understanding the supplied hardware

Each two-input neuron follows this path:

**Inputs and stored weights → MAC → bias → sigmoid activation → output**

| Building block | What it does, how it works, and why it is provided |
|---|---|
| **Input registers** | Capture inputs on a clock edge so the neuron works with stable values while processing them. |
| **Weight memory** | Supplies stored weights one at a time, allowing one MAC to process two input-weight pairs across multiple clocks. |
| **Controller** | Coordinates reads, MAC updates, and result handling. Sequencing matters because memories and registers do not produce all results in the same cycle. |
| **Bias stage** | Adds the stored bias after accumulation. The provided logic handles the necessary fixed-point scaling. |
| **Sigmoid activation** | Uses a ROM of precomputed values instead of hardware that directly evaluates an exponential function. The provided logic handles lookup addressing and range limits. |
| **Network controller** | Starts the final neuron after the hidden layer is ready, so later stages do not consume unfinished results. |

The arithmetic uses **signed 8-bit Q3.5** values. As a reading example, raw `32` represents `1.0`, and raw `-16` represents `-0.5`. Products have a different scale; **the supplied support modules handle that conversion**. Read their comments and trace the relevant signals in simulation. You are not being asked to redesign the bias or sigmoid circuitry.

## Required work

**A. MAC reuse.** Bring your own Assignment 2 MAC into the starter project. Preserve its earlier unsigned operation and extend it for the signed operands used here. Use the in-class interface and the compatibility checks to verify the behavior.

**B. Neuron integration.** Complete the remaining connections in the supplied neuron wrapper. Use the existing controller and datapath modules as demonstrated in lecture. Do not write a new standalone neuron implementation.

**C. Network hierarchy.** The starter includes one correctly instantiated neuron. Use it and the lecture example to instantiate the other neurons and complete the required two-hidden, one-output topology. The provided network controller handles layer sequencing.

**D. Verification.** Run the supplied testbenches, correct the integration issues, and add **one additional neuron case and one additional network case**. Capture a waveform showing the accumulation stages and the handoff between layers.

**E. FPGA compilation.** Compile the complete network for the provided MAX 10 target. Record logic resources, registers, multiplier/DSP usage, memory usage, core-clock timing, and whether the 50 MHz requirement is met.

Your changes should stay within the supplied project structure. All original support modules and memory initialization files must remain part of the project.

## Running the checks

Run the commands from the project root so the memory initialization files resolve correctly.

```bash
vlib work
vlog -sv mac_core.sv tb/tb_mac_reuse.sv
vsim -c work.tb_mac_reuse -do "run -all; quit -f"

vlog -sv +incdir+. *.v mac_core.sv tb/tb_neuron.v tb/tb_tiny_nn.v
vsim -c work.tb_neuron -do "run -all; quit -f"
vsim -c work.tb_tiny_nn -do "run -all; quit -f"
```

The MAC test checks older unsigned configurations and signed operation. The neuron and network tests check the complete design. All tests must pass with the student-completed RTL.

For Quartus:

```bash
quartus_sh --flow compile Neuron
```

The project already specifies `Tiny_NN` as the top-level design and includes a 20 ns `core_clk` constraint. Report **core-clock timing**, not JTAG timing. This part requires simulation and compilation only; **do not program the board or upload a bitstream**.

## Short report

Include a 2–2–1 block diagram, test results, one useful waveform, and a small table of Quartus measurements. Answer these questions briefly:

1. How is your previous MAC reused inside a neuron, and what changed for signed operands?
2. What roles do the weight memory, bias stage, and sigmoid ROM play?
3. Why does the controller need multiple clock cycles for a two-input neuron?
4. How does the hidden layer connect to the output layer, and why are valid/complete signals needed?
5. What resources and core-clock timing did Quartus report? Did the design meet 50 MHz?

## Submission

Push the **completed starter project** to your assigned GitHub directory, including your adapted MAC, the modified neuron/network files, the provided modules and memory files, the updated testbenches, simulation evidence, Quartus reports, and your **short PDF report**. Do not include credentials, secret tokens, or generated compilation databases.

**Part 2**, assigned separately, uses the FPGA cloud to study inference performance on a prebuilt Agilex 7 accelerator. Your Part 1 circuit is not the accelerator used in Part 2.
