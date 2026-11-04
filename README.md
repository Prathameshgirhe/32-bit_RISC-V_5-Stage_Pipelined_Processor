# 32-bit_RISC-V_5-Stage_Pipelined_Processor
<div align="center">

<img src="assets/banner.svg" alt="32-bit RISC-V 5-Stage Pipelined Processor" width="100%"/>

<a href="https://github.com/Prathameshgirhe/riscv-5stage-pipeline">
  <img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=700&size=22&duration=2800&pause=900&color=00E5FF&center=true&vCenter=true&width=700&lines=Designed+in+Verilog+HDL+%F0%9F%94%A7;5-Stage+Pipeline%3A+IF+%E2%86%92+ID+%E2%86%92+EX+%E2%86%92+MEM+%E2%86%92+WB;Data+Forwarding+%2B+Hazard+Detection+%E2%9A%A1;Verified+%7C+Synthesized+on+Xilinx+Vivado+%F0%9F%9A%80" alt="Typing animation"/>
</a>

<br/>

![Verilog](https://img.shields.io/badge/Verilog-HDL-00E5FF?style=for-the-badge&logo=v&logoColor=white)
![RISC-V](https://img.shields.io/badge/RISC--V-RV32I-7C4DFF?style=for-the-badge&logo=riscv&logoColor=white)
![Vivado](https://img.shields.io/badge/Xilinx-Vivado-FF9100?style=for-the-badge&logo=xilinx&logoColor=white)
![QuestaSim](https://img.shields.io/badge/Sim-QuestaSim-00E676?style=for-the-badge)
![Pipeline](https://img.shields.io/badge/Pipeline-5%20Stage-FF4081?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)

<img src="https://img.shields.io/github/stars/Prathameshgirhe/riscv-5stage-pipeline?style=social" alt="stars"/>
<img src="https://img.shields.io/github/forks/Prathameshgirhe/riscv-5stage-pipeline?style=social" alt="forks"/>

</div>

---

## 📑 Table of Contents

[✨ Highlights](#-highlights) · [🧠 Architecture](#-architecture) · [🔄 Pipeline](#-the-5-pipeline-stages) · [⚡ Forwarding](#-data-forwarding-unit) · [🛑 Hazards](#-hazard-detection-unit) · [🧪 Verification](#-verification) · [🏭 Synthesis](#-fpga-synthesis-results) · [📂 Structure](#-project-structure) · [🚀 Run it](#-getting-started) · [🗺️ Roadmap](#%EF%B8%8F-roadmap)

---

## ✨ Highlights

<div align="center">

| 🏗️ **Design** | 🧪 **Verification** | 📉 **Performance** | 🏭 **Synthesis** |
|:---:|:---:|:---:|:---:|
| 32-bit RV32I core<br/>5-stage pipeline | 20+ directed sequences<br/>100+ simulation scenarios | **28 %** fewer stalls<br/>vs. no-forwarding baseline | Xilinx Vivado<br/>Fmax **80–150 MHz** |

</div>

- 🔧 **Micro-architecture:** classic IF · ID · EX · MEM · WB RISC-V (RV32I) core written in synthesizable **Verilog RTL**
- ⚡ **Forwarding unit** resolves RAW data hazards without waiting for write-back
- 🛑 **Hazard detection unit** handles load-use stalls and control-flow flushes
- ✅ **Self-checking testbenches** run in QuestaSim / Vivado Simulator
- 📊 **Measured** stall reduction and FPGA utilization/timing reports included

> 💡 **TL;DR:** Think of it as an assembly line for instructions — while one instruction executes, the next is being decoded and another is being fetched.

---

## 🧠 Architecture

<div align="center">
<img src="assets/pipeline_flow.svg" alt="Animated pipeline flow" width="100%"/>
</div>

```mermaid
flowchart LR
    PC([PC]) --> IM[(Instruction<br/>Memory)]
    IM --> IFID{{IF/ID}}
    IFID --> RF[Register File<br/>+ Imm Gen<br/>+ Control]
    RF --> IDEX{{ID/EX}}
    IDEX --> ALU[ALU]
    FW[[Forwarding Unit]] -.-> ALU
    ALU --> EXMEM{{EX/MEM}}
    EXMEM --> DM[(Data<br/>Memory)]
    DM --> MEMWB{{MEM/WB}}
    MEMWB --> WB([Write Back])
    WB -.-> RF
    HZ[[Hazard Detection Unit]] -.stall.-> PC
    HZ -.stall/flush.-> IFID
    EXMEM -.-> FW
    MEMWB -.-> FW
    style PC fill:#00e5ff,color:#000
    style FW fill:#ff9100,color:#000
    style HZ fill:#ff1744,color:#fff
    style WB fill:#ff4081,color:#000
```

---

## 🔄 The 5 Pipeline Stages

| Stage | Name | What happens | Key modules |
|:---:|:---|:---|:---|
| 🟦 **IF** | Instruction Fetch | Read instruction at `PC`, compute `PC+4` | `pc.v`, `imem.v` |
| 🟪 **ID** | Instruction Decode | Decode opcode, read registers, generate immediate, hazard check | `control.v`, `regfile.v`, `immgen.v`, `hazard_unit.v` |
| 🟧 **EX** | Execute | ALU operation, branch target & compare, forwarding muxes | `alu.v`, `forward_unit.v` |
| 🟩 **MEM** | Memory Access | Load / store from data memory | `dmem.v` |
| 🟥 **WB** | Write Back | Select ALU / memory result, write to `rd` | `wb_mux.v` |

Pipeline registers between stages: `IF/ID` → `ID/EX` → `EX/MEM` → `MEM/WB`.

<details>
<summary><b>📜 Supported RV32I instructions (click to expand)</b></summary>

| Type | Instructions |
|:---|:---|
| **R-type** | `ADD SUB AND OR XOR SLL SRL SRA SLT SLTU` |
| **I-type** | `ADDI ANDI ORI XORI SLTI SLLI SRLI SRAI` |
| **Load / Store** | `LW SW` *(extend to LB/LH/SB/SH if implemented)* |
| **Branch** | `BEQ BNE BLT BGE` |
| **Jump / Upper** | `JAL JALR LUI AUIPC` |

> ✏️ Edit this list to match exactly what your core implements.
</details>

---

## ⚡ Data Forwarding Unit

<div align="center">
<img src="assets/forwarding.svg" alt="Animated forwarding paths" width="100%"/>
</div>

Without forwarding, a dependent instruction must wait until the producer writes back. The forwarding unit **bypasses** results straight from `EX/MEM` or `MEM/WB` into the ALU inputs.

| `ForwardA / ForwardB` | Source | Meaning |
|:---:|:---|:---|
| `00` | ID/EX register | No hazard – use register-file value |
| `10` | EX/MEM `alu_result` | Result from previous instruction |
| `01` | MEM/WB `wb_data` | Result from two instructions ago |

```verilog
// Forwarding logic (EX stage)
always @(*) begin
    forward_a = 2'b00;
    forward_b = 2'b00;

    // EX hazard  (priority)
    if (exmem_reg_write && (exmem_rd != 0) && (exmem_rd == idex_rs1)) forward_a = 2'b10;
    if (exmem_reg_write && (exmem_rd != 0) && (exmem_rd == idex_rs2)) forward_b = 2'b10;

    // MEM hazard
    if (memwb_reg_write && (memwb_rd != 0) && !(exmem_reg_write && exmem_rd != 0 && exmem_rd == idex_rs1)
        && (memwb_rd == idex_rs1)) forward_a = 2'b01;
    if (memwb_reg_write && (memwb_rd != 0) && !(exmem_reg_write && exmem_rd != 0 && exmem_rd == idex_rs2)
        && (memwb_rd == idex_rs2)) forward_b = 2'b01;
end
```

---

## 🛑 Hazard Detection Unit

<div align="center">
<img src="assets/hazard_stall.svg" alt="Animated load-use stall" width="100%"/>
</div>

Forwarding cannot fix a **load-use** hazard (data only exists after `MEM`), so the hazard unit **stalls one cycle**:

```verilog
// Load-use hazard detection (ID stage)
assign stall = idex_mem_read &&
               ((idex_rd == ifid_rs1) || (idex_rd == ifid_rs2));

// On stall: freeze PC & IF/ID, insert bubble (NOP) into ID/EX
assign pc_write   = ~stall;
assign ifid_write = ~stall;
assign idex_flush =  stall;
```

| Hazard type | Solution in this design |
|:---|:---|
| 🔴 RAW (ALU → ALU) | Forwarding (no stall) |
| 🟠 RAW (Load → use) | 1-cycle stall + forwarding |
| 🟡 Control (branch/jump) | Flush `IF/ID` (and `ID/EX`) on taken branch |

---

## 🧪 Verification

- ✅ **20+ directed instruction sequences** (arithmetic, logic, load/store, branch, back-to-back dependencies, load-use, x0 writes)
- ✅ **100+ simulation scenarios** covering hazard and forwarding corner cases
- ✅ Run on **QuestaSim / Vivado Simulator**, results checked against expected register-file / memory state

```text
================ TEST SUMMARY ================
 Directed sequences : 20+  PASS
 Scenarios          : 100+ PASS
 Failures           : 0
==============================================
```

> 📸 *Add your waveform screenshot here:* `![Waveform](docs/waveform.png)`

---

## 📉 Performance: Stall Reduction

Measured on the same program set against a baseline pipeline **without forwarding**:

```text
Baseline  (no forwarding) ████████████████████████████████ 100 %
This core (forwarding)    ███████████████████████░░░░░░░░░  72 %   →  −28 % pipeline stalls
```

<sub>⚠️ Replace with your exact measured value from `[program]` before publishing.</sub>

---

## 🏭 FPGA Synthesis Results

<div align="center">

| Metric | Result |
|:---:|:---:|
| 🎯 Target FPGA | `[FPGA part]` |
| ⏱️ Fmax | **80–150 MHz** |
| 🧩 LUTs | **1,200–2,500** |
| 🔁 FFs | **800–1,500** |
| ⏳ WNS @ 100 MHz | **≈ +0.5 → +3 ns** (positive slack ✅) |

</div>

<sub>⚠️ Replace ranges with your final Vivado utilization & timing report numbers.</sub>

---

## 📂 Project Structure

```text
📦 riscv-5stage-pipeline
 ┣ 📂 rtl
 ┃ ┣ 📜 top.v              # Top-level processor
 ┃ ┣ 📜 pc.v
 ┃ ┣ 📜 imem.v / dmem.v
 ┃ ┣ 📜 regfile.v
 ┃ ┣ 📜 control.v
 ┃ ┣ 📜 immgen.v
 ┃ ┣ 📜 alu.v
 ┃ ┣ 📜 forward_unit.v     # ⚡ forwarding
 ┃ ┣ 📜 hazard_unit.v      # 🛑 stall / flush
 ┃ ┗ 📜 pipe_regs.v        # IF/ID, ID/EX, EX/MEM, MEM/WB
 ┣ 📂 tb                   # Testbenches
 ┣ 📂 programs             # .hex / .mem directed tests
 ┣ 📂 scripts              # QuestaSim .do / Vivado .tcl
 ┣ 📂 docs                 # Waveforms, reports
 ┣ 📂 assets               # README animations
 ┗ 📜 README.md
```

> ✏️ Adjust file names to match your actual repo.

---

## 🚀 Getting Started

```bash
# 1. Clone
git clone https://github.com/Prathameshgirhe/riscv-5stage-pipeline.git
cd riscv-5stage-pipeline

# 2a. Simulate with QuestaSim
vlib work && vlog rtl/*.v tb/tb_top.v
vsim -c tb_top -do "run -all; quit"

# 2b. ...or Icarus Verilog + GTKWave
iverilog -o sim rtl/*.v tb/tb_top.v && vvp sim && gtkwave dump.vcd

# 3. Synthesize in Vivado (batch)
vivado -mode batch -source scripts/synth.tcl
```

---

## 🛠️ Tech Stack

<div align="center">

<img src="https://img.shields.io/badge/Verilog-HDL-00E5FF?style=flat-square"/>
<img src="https://img.shields.io/badge/Xilinx-Vivado-FF9100?style=flat-square"/>
<img src="https://img.shields.io/badge/QuestaSim-Simulation-00E676?style=flat-square"/>
<img src="https://img.shields.io/badge/GTKWave-Waveforms-7C4DFF?style=flat-square"/>
<img src="https://img.shields.io/badge/RISC--V-ISA-FF4081?style=flat-square"/>
<img src="https://img.shields.io/badge/Git-GitHub-white?style=flat-square&logo=github&logoColor=black"/>

</div>

---

## 🗺️ Roadmap

- [x] 5-stage RV32I pipeline
- [x] Forwarding + load-use stall
- [x] Directed verification & Vivado synthesis
- [ ] Branch predictor (static → 2-bit dynamic)
- [ ] Instruction / data caches
- [ ] M-extension (MUL/DIV)
- [ ] CSR + interrupts
- [ ] Run on real FPGA board 🚀

---

## 🤝 Contributing

Issues and PRs are welcome! Fork → branch → commit → pull request.

## 📜 License

Distributed under the **MIT License**.

---

<div align="center">

### 👨‍💻 Author — Prathamesh Girhe

[![GitHub](https://img.shields.io/badge/GitHub-Prathameshgirhe-181717?style=for-the-badge&logo=github)](https://github.com/Prathameshgirhe)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/)

⭐ **If you liked this project, please star the repo!** ⭐

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=6,11,20&height=120&section=footer" width="100%"/>

</div>
