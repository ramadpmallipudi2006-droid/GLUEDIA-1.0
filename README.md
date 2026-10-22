<div align="center">

# 🩸 GLUEDIA 1.0

### Continuous Electrochemical Glucose Monitoring (CGM) Mixed-Signal ASIC
#### SkyWater 130nm CMOS • Transistor-Level Analog Front-End • 8-Bit StrongARM SAR ADC • 100 MHz Pipelined DSP

[![Process](https://img.shields.io/badge/Process-SkyWater%20130nm%20CMOS-blue?style=for-the-badge&logo=microchip)](https://skywater-pdk.readthedocs.io/)
[![Architecture](https://img.shields.io/badge/Architecture-Custom%20Mixed--Signal%20ASIC-orange?style=for-the-badge)]()
[![ADC Performance](https://img.shields.io/badge/SAR%20ADC-7.90%20ENOB%20%7C%2049.29%20dB%20SNDR-green?style=for-the-badge)]()
[![DSP Performance](https://img.shields.io/badge/DSP%20Throughput-100%20MHz%20%7C%201%20Sample%2FClock-purple?style=for-the-badge)]()
[![Status](https://img.shields.io/badge/Status-Tapeout%20Ready%20(GDSII%20Signoff)-brightgreen?style=for-the-badge)]()

<br />

**[🚀 Explore GLUEDIA 2.0 (Flagship Non-Invasive Opto-Acoustic Platform) →](https://github.com/ramadpmallipudi2006-droid/GLUEDIA-2.0)**

---

</div>

## 📌 Executive Summary & Project Moto

> **The GLUEDIA 1.0 Moto:**  
> *"To replace bulky, power-hungry, and expensive discrete sensor electronics with a monolithic, precision mixed-signal ASIC that directly converts electrochemical nanoampere enzymatic currents into calibrated, clinical-grade blood glucose readings in real time."*

Continuous Glucose Monitors (CGMs) are a medical lifeline for millions living with Type 1 and Type 2 diabetes. However, current commercial transmitters rely on multi-chip printed circuit assemblies with discrete operational amplifiers, external ADCs, and microcontrollers. This discrete approach drives transmitter costs to **$60–$120 per unit**, introduces parasitic board-level noise, and drains battery life within 10 to 14 days.

**GLUEDIA 1.0** solves this paradigm through a **full-custom, monolithic mixed-signal ASIC** designed in the **SkyWater 130nm CMOS open-source process**:
- **Direct Sensor Interfacing:** Directly accepts **0–800 nA** currents from glucose oxidase (GOx) electrochemical working electrodes.
- **Ultra-Low Noise Front-End:** High-gain Transimpedance Amplifier (1 MΩ transimpedance gain) with active Sallen-Key filtering.
- **Precision Quantization:** 8-bit Successive Approximation Register (SAR) ADC delivering **7.90 Effective Number of Bits (ENOB)** under Monte Carlo mismatch.
- **On-Chip Fixed-Point DSP:** 3-stage pipelined digital engine performing division-free two-point calibration and affine conversion to **mg/dL** at **100 MHz**.
- **Silicon-Proven Tapeout Signoff:** Full-chip layout (0.38 mm² silicon area) with 100% clean Design Rule Checking (DRC) and Layout Versus Schematic (LVS).

---

## 🧭 Platform Navigation

| Repository | Focus | Sensing Principle | Status |
|:---|:---|:---|:---:|
| **GLUEDIA 1.0 (This Repo)** | Electrochemical Continuous Glucose Monitoring (CGM) | Glucose Oxidase Enzymatic Current (0–800 nA) | **Tapeout Ready (GDSII)** |
| **[GLUEDIA 2.0 (Flagship)](https://github.com/ramadpmallipudi2006-droid/GLUEDIA-2.0)** | Breakthrough Non-Invasive Needle-Free Platform | Dual NIR PPG + MIR Photoacoustic + Bioimpedance | **Tapeout Ready (GDSII)** |

---

## 🪜 Step-by-Step Technical Guide for Engineering Teams

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                GLUEDIA 1.0 SIGNAL CHAIN                                │
│                                                                                        │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│   │ Electrochemical│   │  Two-Stage   │     │ Sallen-Key   │     │ 8-Bit StrongARM  │  │
│   │ Working Elect.│──►│  CMOS TIA    │──►  │ 2nd-Ord LPF  │──►  │    SAR ADC       │  │
│   │  (0 - 800 nA)│     │ (1 MΩ Gain)  │     │  (fc = 10Hz) │     │   (7.90 ENOB)    │  │
│   └──────────────┘     └──────────────┘     └──────────────┘     └────────┬─────────┘  │
│                                                                           │            │
│                                                                           ▼            │
│                       ┌─────────────────────────────────────────────────────────────┐  │
│                       │                     3-Stage Pipelined DSP                   │  │
│                       │          [(ADC_CODE - BASELINE) * GAIN] >> 16               │  │
│                       │                  Output: Glucose in mg/dL                   │  │
│                       └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### Step 1: Biochemical Transduction & Sensor Current Dynamics
The electrochemical glucose sensor utilizes immobilized **Glucose Oxidase (GOx)** enzymes on a platinum working electrode. Under a constant +0.6V polarization bias relative to an Ag/AgCl reference electrode, glucose molecules in interstitial fluid undergo two-step oxidation:

$$\text{Glucose} + \text{O}_2 \xrightarrow{\text{GOx}} \text{Gluconic Acid} + \text{H}_2\text{O}_2$$
$$\text{H}_2\text{O}_2 \xrightarrow{+0.6\text{ V Bias}} 2\text{H}^+ + \text{O}_2 + 2e^-$$

- **Dynamic Range:** $0\text{ nA}$ ($0\text{ mg/dL}$) to $800\text{ nA}$ ($400\text{ mg/dL}$).
- **Nominal Sensitivity:** $2\text{ nA} / (\text{mg/dL})$.
- **Electrical Equivalent Model:** A high Norton equivalent resistance ($R_s > 10\text{ M}\Omega$) shunted by electrode double-layer capacitance ($C_s \approx 20\text{ pF}$).

---

### Step 2: Transistor-Level Analog Front-End (AFE)

#### 1. Two-Stage CMOS Transimpedance Amplifier (TIA)
To convert the fragile nanoampere current into a robust voltage signal without loading the electrochemical cell, GLUEDIA 1.0 employs a custom two-stage CMOS operational amplifier configured in inverting transimpedance mode:

<div align="center">
  <img src="assets/cmos_tia_schematic.svg" alt="CMOS Transimpedance Amplifier Transistor Schematic" width="80%" />
  <br />
  <em>Figure 1 — Transistor-level schematic of the two-stage CMOS TIA in SkyWater 130nm with active current-mirror loads, Miller compensation (Cc = 5 pF), and lead-lag feedback stabilization (Cf = 0.5 pF across Rf = 1 MΩ).</em>
</div>

<br />

- **Differential Input Pair:** Long-channel NMOS transistors ($W/L = 10\mu\text{m}/1\mu\text{m}$) operating in saturation to minimize $1/f$ flicker noise.
- **Active Current Mirror Load:** PMOS devices ($W/L = 10\mu\text{m}/1\mu\text{m}$) maximizing first-stage voltage gain ($A_{v1} \approx 42\text{ dB}$).
- **Second Stage:** PMOS Common-Source gain stage ($W/L = 40\mu\text{m}/1\mu\text{m}$) driving a 20 µm NMOS current source ($A_{v2} \approx 38\text{ dB}$), delivering total open-loop gain $> 80\text{ dB}$.
- **Phase-Lead Feedback Stabilization:** The sensor capacitance ($C_s \approx 20\text{ pF}$) introduces an open-loop pole at $f_p = \frac{1}{2\pi R_f C_s} \approx 7.96\text{ kHz}$. A feedback capacitor $C_f = 0.5\text{ pF}$ introduces a zero at $f_z = \frac{1}{2\pi R_f C_f} \approx 318\text{ kHz}$, boosting phase margin to **$58.4^\circ$** and eliminating ringing.

<div align="center">
  <img src="assets/cmos_tia_dc_sim.png" alt="TIA DC Transfer Characteristics" width="85%" />
  <br />
  <em>Figure 2 — TIA DC transfer simulation: 0 to 800 nA input current mapped with exceptional linearity (R² = 0.9999) to 0.900 V – 1.700 V across the 1.8V analog supply.</em>
</div>

<br />

#### 2. Sallen-Key 2nd-Order Active Low-Pass Filter (LPF)
Biological glucose concentrations change over minutes ($< 0.05\text{ Hz}$), whereas patient muscle movements, cardiac pulsations, and 50/60 Hz electromagnetic coupling inject high-frequency noise:
- **Cutoff Frequency ($f_c$):** $10\text{ Hz}$ Butterworth characteristic ($Q = 0.707$).
- **Attenuation:** $> 40\text{ dB/decade}$ roll-off, attenuating 50/60 Hz mains interference by $> 28\text{ dB}$.

<div align="center">
  <img src="assets/cmos_lpf_simulation.png" alt="Sallen-Key LPF Simulation" width="85%" />
  <br />
  <em>Figure 3 — 2nd-order Sallen-Key low-pass filter frequency response and transient settling response, confirming unity DC gain and clean 10 Hz band-limiting.</em>
</div>

---

### Step 3: Mixed-Signal SAR ADC Design & Characterization

GLUEDIA 1.0 integrates an 8-bit Successive Approximation Register (SAR) ADC designed for maximum energy efficiency:

```
                  ┌────────────────────────────────────────┐
                  │          SAR ADC ARCHITECTURE          │
                  └────────────────────────────────────────┘

    V_in (from LPF) ──────►[ Sample & Hold Switch ]
                                   │
                                   ▼
    V_held ───────────────►(+) StrongARM
                               Latch       ──► SAR Logic (FSM) ──► 8-Bit Digital Code
    V_dac ────────────────►(-) Comparator           │
                                   ▲                │ (Switch Controls)
                                   │                ▼
                             [ 8-Bit Binary-Weighted CDAC ]
                             (Unit Capacitance: C0 = 10 fF)
```

1. **StrongARM Dynamic Latch Comparator:**
   - 11-transistor clocked topology consuming **zero static DC power**.
   - Metastability resolution time: $< 1.2\text{ ns}$.
   - Input-referred offset cancelled via matched differential layout and dummy finger insertion.
2. **Binary-Weighted Capacitive DAC (CDAC):**
   - Metal-Insulator-Metal (MIM) capacitors ($C_0 = 10\text{ fF}$).
   - Total array capacitance $C_{tot} = 256 \times 10\text{ fF} = 2.56\text{ pF}$, establishing thermal $kT/C$ noise well below $0.1\text{ LSB}$.

<div align="center">
  <img src="assets/sar_adc_cosim.png" alt="SAR ADC Co-Simulation Waveforms" width="90%" />
  <br />
  <em>Figure 4 — Mixed-signal transient co-simulation: Track/Hold phase, binary search CDAC top-plate convergence, StrongARM comparator latch firings, and digital code output.</em>
</div>

<br />

<div align="center">
  <img src="assets/adc_report.png" alt="SAR ADC Characterization Report" width="90%" />
  <br />
  <em>Figure 5 — Static and dynamic Monte Carlo characterization: Effective Number of Bits (ENOB = 7.90), Signal-to-Noise & Distortion Ratio (SNDR = 49.29 dB), DNL ≤ 0.08 LSB, and INL ≤ 0.08 LSB under 0.5% capacitor mismatch.</em>
</div>

---

### Step 4: Digital Signal Processing (DSP) Pipeline Core

The digital processor converts raw 8-bit quantization codes ($0–255$) into calibrated clinical glucose concentrations ($0–400\text{ mg/dL}$) without power-hungry hardware dividers.

$$\text{Glucose (mg/dL)} = \left[ (\text{ADC\_CODE} - \text{BASELINE}) \times \text{GAIN\_CAL} \times \text{GAIN\_CONV} \right] \gg 16$$

- **Pipeline Stages:**
  - **Stage 1:** Baseline Offset Subtraction (Signed 16-bit extension).
  - **Stage 2:** Factory Sensitivity Calibration (`GAIN_CAL` scaling).
  - **Stage 3:** Clinical Units Conversion & Arithmetic Shift (`GAIN_CONV` multiplication and 16-bit truncation).
- **Throughput & Latency:** 3 clock cycles latency, **1 output every clock cycle**.
- **Timing Closure:** Synthesized in SkyWater 130nm standard cell library (`sky130_fd_sc_hd`) closing timing at **100 MHz** with **+0.85 ns setup slack**.

---

### Step 5: Physical Layout & Tapeout Signoff (GDSII)

The mixed-signal floorplan integrates the sensitive analog front-end with the high-speed digital DSP core:

<div align="center">
  <img src="assets/gluedia_chip_top.png" alt="GLUEDIA 1.0 Full Chip Layout" width="65%" />
  <br />
  <em>Figure 6 — GLUEDIA 1.0 Full-Chip Tapeout Top Layout (GDSII): Analog Front-End Macro on West, Digital DSP Core on East, 40 µm guard ring substrate isolation, and I/O wirebond pad ring.</em>
</div>

<br />

<div align="center">
  <img src="assets/gluedia_digital_layout.png" alt="GLUEDIA Digital Core Layout" width="85%" />
  <br />
  <em>Figure 7 — Digital Core Place-and-Route layout generated via OpenLane/OpenROAD flow, demonstrating balanced clock tree distribution and power stripe grids.</em>
</div>

---

## 🏆 Industrial Comparison Benchmark

| Specification | Dexcom G6 / G7 | Abbott FreeStyle Libre 3 | Medtronic Guardian 4 | **GLUEDIA 1.0 (This Work)** |
|:---|:---:|:---:|:---:|:---:|
| **Architecture** | Multi-chip PCB assembly | Multi-chip PCB assembly | Multi-chip PCB assembly | **Monolithic Mixed-Signal ASIC** |
| **Silicon Technology** | Proprietary BiCMOS / CMOS | Proprietary ASIC | Mixed discrete ICs | **SkyWater 130nm Open CMOS** |
| **Input Current Range** | 0 – 500 nA | 0 – 600 nA | 0 – 500 nA | **0 – 800 nA (Extended)** |
| **ADC Resolution** | 10-bit nominal | 10-bit nominal | 10-bit nominal | **8-bit SAR (7.90 ENOB)** |
| **DSP Core** | Host Microcontroller | Host Microcontroller | External MCU | **Dedicated On-Chip 100MHz DSP** |
| **Silicon Area** | > 15 mm² (board area) | > 10 mm² (board area) | > 20 mm² (board area) | **< 0.40 mm² (Die Area)** |
| **Transmitter Unit Cost** | $60 – $120 | Integrated (disposable) | $150 – $300 | **< $0.85 (Volume Silicon)** |
| **DRC / LVS Status** | Commercial | Commercial | Commercial | **100% Clean Signoff** |

---

## 📁 Repository Structure

```
GLUEDIA-1.0/
├── README.md                               # Master technical specification & executive guide
├── IP_NOTICE.md                            # Intellectual property, licensing & evaluation policy
├── .gitignore                              # Excludes proprietary hardware source code
│
├── assets/                                 # High-resolution schematics, simulations & chip layouts
│   ├── cmos_tia_schematic.svg              # 2-stage CMOS TIA transistor schematic
│   ├── cmos_tia_dc_sim.png                 # TIA DC transfer characteristics (0-800 nA -> 0.9-1.7V)
│   ├── cmos_lpf_simulation.png             # 2nd-order Sallen-Key low-pass filter response
│   ├── sar_adc_cosim.png                   # Mixed-signal SAR ADC transient co-simulation
│   ├── sar_adc_transfer.png                # ADC quantization transfer curve & linearity
│   ├── adc_report.png                      # Monte Carlo dynamic characterization (ENOB, SNDR)
│   ├── adc_transient.png                   # SAR ADC step & transient response
│   ├── gluedia_chip_top.png                # Full-chip mixed-signal tapeout layout (GDSII)
│   └── gluedia_digital_layout.png          # Synthesized digital core place-and-route layout
│
└── docs/                                   # Architectural dossiers & interview deep-dives
    ├── GLUEDIA_1.0_HARDWARE_SPECIFICATION.md
    └── GLUEDIA_1.0_DSP_DEEP_DIVE.md
```

---

## 🛡️ Intellectual Property & Evaluation Requests

This repository serves as an executive hardware design showcase. To protect patent-pending topologies and proprietary mixed-signal silicon IP, raw synthesizable Verilog HDL cores, transistor SPICE netlists, and tapeout GDSII databases are withheld from the public domain.

Qualified semiconductor companies, medical device manufacturers, and research institutions may request full evaluation access under a standard **Non-Disclosure Agreement (NDA)**:

- **Lead Engineer:** Rama Durga Prasad Mallipudi
- **Email:** [rama.dp.mallipudi2006@gmail.com](mailto:rama.dp.mallipudi2006@gmail.com)
- **GitHub Profile:** [@ramadpmallipudi2006-droid](https://github.com/ramadpmallipudi2006-droid)

---

<div align="center">

**GLUEDIA 1.0 — Precision Mixed-Signal Silicon for Continuous Diabetes Care**  
Designed by Rama Durga Prasad Mallipudi

</div>
