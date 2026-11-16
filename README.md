# Part 1 — Reuse Your MAC to Build a Hardware Neuron

**Digital Logic II · Individual assignment · Estimated time: 4–6 hours**  
**Tools:** Questa/ModelSim (or another SystemVerilog simulator) and Quartus Prime Lite  
**Target:** DE10-Lite MAX 10 (`10M50DAF484C7G`), 50 MHz clock (20 ns)

In Assignment 2, you built a **multiply-accumulate (MAC)** unit. In this assignment, you will **reuse your MAC inside a neuron**, then connect three neurons into a small **2–2–1 neural network**. You will learn how arithmetic, registers, memories, and a controller work together to implement a neural network in hardware.

**Scope:** Simulate and compile the design; **do not program a physical FPGA**. Part 2 is separate: you will use the FPGA Cloud client to run ResNet-50 on a **prebuilt Agilex 7 accelerator**. Your Part 1 neuron is not the accelerator running ResNet-50.

## 1. Understand the provided design

![Conceptual hardware neuron architecture](docs/Neuron_Hardware_Architecture.png)

A neuron performs three steps:

1. **MAC:** Multiply each input by its weight and accumulate the products.
2. **Bias:** Add a constant to the MAC result.
3. **Activation:** Use a sigmoid lookup table (ROM) to obtain the output.

The neuron equation is:

$$
y = \sigma(x_0 w_0 + x_1 w_1 + b)
$$

For example, inputs `1.0` and `0.5`, weights `1.0` and `−1.0`, and bias `0.25` give a weighted sum of `0.75`. The provided sigmoid table returns approximately **0.6875**.

### What is already implemented for you?

| Provided component | What it does | Why we use it |
|---|---|---|
| **Weight memory** (`Weight_Memory.v`) | Stores two weights and reads one per clock. | Weights are needed for each multiplication. |
| **Neuron controller** (`Neuron.v`) | An FSM sequences input capture, weight reads, MAC steps, bias, and activation. | A single MAC is reused across two inputs, so operations must happen in order. |
| **Bias calculation** (`Neuron.v`) | Adjusts the bias to match the MAC's number format, then adds it **once**. | The MAC products and stored bias use different fixed-point scales. |
| **Sigmoid ROM** (`Sig_ROM.v`, `sigContent.mem`) | Looks up an approximate activation output. Address scaling and range limiting are provided. | Reading a table is simpler than building an exponential-function circuit. |
| **Network controller** (`Tiny_NN.v`) | Waits until the two hidden neurons finish before starting the output neuron. | Later layers must wait for valid results from earlier layers. |

**A quick note on fixed point:** Inputs, weights, and outputs use signed **8-bit Q3.5** values. The stored integer is 32 times the real value: `32` means `1.0`, `16` means `0.5`, and `−32` means `−1.0`. Multiplication produces a Q6.10 value. **The provided neuron code handles the bias scaling and sigmoid ROM address conversion; you do not need to implement either operation.** Read their comments to understand what they do.

## 2. Your implementation tasks

### Task 1 — Reuse and extend your MAC

Copy **your own** Assignment 2 `mac_core.sv` into this folder, beside `Neuron.v`. Keep your earlier assignment file unchanged.

Add an integer parameter **`SIGNED_MODE` (default `0`)** while preserving the original module ports, parameters `W` and `ACC_W`, and clock behavior:

- `SIGNED_MODE=0`: keep the original **unsigned** multiplication for 8-bit and 16-bit MACs.
- `SIGNED_MODE=1`: support **signed two's-complement** 8-bit multiplication. For example, `8'hE0` represents `−32`, not `224`.
- With `enable=1`, accumulate one product on each rising edge. With `enable=0`, hold the accumulator. Synchronous `reset` clears it and has priority.
- Extend signed products with their **sign bit**, not zeros, before accumulating into 32 bits.

**Hint:** Both operands must be treated as signed when multiplied. Simply declaring the accumulator signed does not change the meaning of the operands. See `MAC_IMPORT_INSTRUCTIONS.txt`.

### Task 2 — Connect the MAC inside the neuron

In `Neuron.v`, complete **TODO 2A–2D** on the `mac_core` instance:

- Connect MAC `reset` to the provided clear signal.
- Connect MAC `enable` to the provided step signal.
- Connect operand `a` to the selected registered input.
- Connect operand `b` to the weight memory output.

**Do not** implement another multiplier or accumulator inside `Neuron.v`. The provided FSM, bias calculation, and sigmoid lookup logic are already complete.

### Task 3 — Connect the 2–2–1 network

In `Tiny_NN.v`, complete **TODO 5A–5C**: connect the output neuron's **start signal** and its **two input ports** to the provided hidden-layer signals.

The two hidden neurons run in parallel. When both results become valid, the network controller starts the output neuron. Its result is reported with a `done` pulse.

## 3. Simulate and check your work

Run commands from the **top-level assignment directory** so the `.mem` files can be found.

**First, test your MAC:**

```bash
vlib work
vlog -sv mac_core.sv tb/tb_mac_reuse.sv
vsim -c work.tb_mac_reuse -do "run -all; quit -f"
```

The testbench checks **unsigned W=8**, **unsigned W=16**, and **signed W=8** behavior, including reset, accumulation, and enable. All should pass.

**Next, test the neuron and network:**

```bash
vlog -sv +incdir+. mac_core.sv Neuron.v Weight_Memory.v Sig_ROM.v Tiny_NN.v \
    tb/tb_neuron.v tb/tb_tiny_nn.v
vsim -c work.tb_neuron -do "run -all; quit -f"
vsim -c work.tb_tiny_nn -do "run -all; quit -f"
```

The provided testbenches check known input/output pairs. **Add one extra `check_case` to each of `tb/tb_neuron.v` and `tb/tb_tiny_nn.v`.** Suggested inputs and expected outputs appear in the comments of each file; you do not need to calculate a new sigmoid table.

Submit PASS output for all three tests, plus **one waveform or labeled transcript** showing the MAC operations and the transition from hidden neurons to the output neuron. If values are unknown, check that you are running from the assignment root and that the `.mem` files are present.

> The starter intentionally has missing MAC connections and **does not include `mac_core.sv`**. Tests will not compile or pass until you import and finish your own implementation.

## 4. Compile with Quartus

The provided Quartus project uses **`Tiny_NN`** as the top-level entity and has a **20 ns constraint on `clk`** (named `core_clk`). Compile with:

```bash
quartus_sh --flow compile Neuron
```

You can also open `Neuron.qpf` in Quartus and select **Processing → Start Compilation**.

Record the following from the reports:

| Measurement | Your result |
|---|---|
| Compilation successful? | |
| Logic resources and registers | |
| DSP/multiplier and memory resources | |
| Core-clock Fmax, if reported | |
| Does the implementation meet **50 MHz**? | |

Check the **core/user clock timing**, not the JTAG clock. If any field is unavailable, write `N/A` and explain. Compilation and timing analysis **do not mean the board was programmed**.

## 5. Short report — answer these five questions

1. What does your MAC do inside the neuron? How did you support signed operands without breaking its original unsigned behavior?
2. Why does a neuron use **weights, bias, and sigmoid activation**? Which parts are provided and which did you connect?
3. How does the provided FSM control weight reads and MAC operations? Why can't both weights be processed in the same clock with this single MAC?
4. How are the three neurons connected? How does the output neuron know when the hidden neurons finish?
5. What logic, register, DSP, and **core Fmax** results did Quartus report? Did the network meet the 50 MHz target?

Your report should also include a simple **2–2–1 network block diagram** and your measured Quartus results. You do **not** need hand calculations of ROM addresses or fixed-point bias shifts.

## 6. Submit through GitHub

Push to your assigned individual submission directory:

- Your **modified `mac_core.sv`**, completed `Neuron.v` and `Tiny_NN.v`, plus the supplied support and memory files.
- The MAC, neuron, and network testbenches, including your **one new case in each** of the neuron and network tests.
- PASS results, a waveform/transcript, and Quartus resource/timing evidence.
- A **short PDF report** with the five answers, diagram, and timing/resource results.

**Do not include cloud tokens, passwords, private keys, or an `.sof` file.**
