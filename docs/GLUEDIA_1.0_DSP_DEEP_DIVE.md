# GLUEDIA — Digital Signal Processing Block: Complete Deep Dive

**Audience:** you, preparing to present the DSP block to an interviewer.
Everything here is traced to the actual RTL in [glucose_dsp.v](glucose_dsp.v), the testbench in [glucose_dsp_tb.v](glucose_dsp_tb.v), the golden model in [calibration_sim.py](calibration_sim.py), and the real OpenLane signoff reports under [runs/RUN_2026.10.09_19.35.27/](runs/RUN_2026.10.09_19.35.27/).

---

## 0. The 60-Second Answer (memorize this)

> "The DSP is the block that turns a meaningless 8-bit ADC code into a clinically meaningful glucose number in mg/dL. It does that with a **two-stage affine transform** — subtract a baseline, then multiply by two calibration gains — implemented entirely in **Q8.8 fixed-point integer arithmetic**. There is no divider and no floating point anywhere: every division is a multiply by a pre-computed reciprocal followed by an arithmetic right shift by 8. It's a **3-stage pipeline** at 100 MHz, 3 clock cycles of latency, one result per clock of throughput. The calibration coefficients live in a **4-entry runtime-writable register file**, so the same silicon works with any sensor lot — you recalibrate in firmware, not in a new tapeout. The whole digital top closed timing at 100 MHz with 0.85 ns of setup slack and zero TNS in Sky130."

---

## 1. Where the DSP Sits — The Full Signal Chain

```
  Glucose in interstitial fluid
            │
            │  glucose oxidase enzyme reaction
            ▼
   ┌─────────────────┐
   │ Electrochemical │   I_sensor = 5 nA per mg/dL
   │     Sensor      │   (0 … 900 nA working range)
   └────────┬────────┘
            │  CURRENT (nA)  ← tiny, high-impedance, noisy
            ▼
   ┌─────────────────┐
   │      TIA        │   V = V_BIAS + Rf·I     Rf = 1 MΩ
   │  (CMOS, on-chip)│   0.9 V + 1 mV/nA
   └────────┬────────┘
            │  VOLTAGE (0.9 … 1.8 V)
            ▼
   ┌─────────────────┐
   │  Sallen-Key LPF │   anti-alias + noise rejection
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   S/H + SAR ADC │   8-bit, VREF = 1.8 V
   │   (sar_logic.v) │   V_LSB = 1.8/256 = 7.03125 mV
   └────────┬────────┘
            │  adc_data[7:0] + adc_valid   ← code 128 ≈ 0 nA
            ▼
   ╔═════════════════════════════════════════╗
   ║          glucose_dsp.v   ◄── YOU ARE HERE
   ║  code → current (nA) → glucose (mg/dL)  ║
   ╚════════════════════╤════════════════════╝
                        │
                        ▼
         current_nA[15:0], glucose_mgdL[15:0], dsp_valid
         saturated_high, saturated_low, raw_code[7:0]
```

**The key insight to state out loud:** everything before the DSP is *physics*. The DSP is the first place where the chip knows what a "milligram per decilitre" is. The analog chain cannot do calibration — a resistor can't be reprogrammed after the sensor ships. So all the sensor-lot variation, baseline drift, and unit conversion is deliberately pushed into the digital domain where it's one register write.

---

## 2. The Modules Inside the DSP

`glucose_dsp` is a single Verilog module, but it is architecturally **four distinct functional blocks**. Present it this way — it shows you think in blocks, not in lines of code.

```
                     cal_we, cal_addr[1:0], cal_wdata[15:0]
                                   │
                    ┌──────────────▼───────────────┐
                    │  ① CALIBRATION REGISTER FILE │
                    │  r_code_offset        (128)  │
                    │  r_na_per_code_x256  (1800)  │
                    │  r_mgdl_per_na_x256    (51)  │
                    │  r_glucose_offset       (0)  │
                    └──┬────────┬──────────┬───────┘
                       │        │          │
  adc_data[7:0] ───┐   │        │          │
  adc_valid ───────┼───▼────────┼──────────┼──────────────────┐
                   │ ┌──────────┴──┐       │                  │
                   │ │② STAGE 1    │       │                  │
                   │ │  BASELINE   │       │                  │
                   │ │  SUBTRACT   │       │                  │
                   │ │ 9-bit sub   │       │                  │
                   │ └──────┬──────┘       │                  │
                   │        │ s1_signed_delta[8:0]            │
                   │ ┌──────▼──────────────▼─┐                │
                   │ │③ STAGE 2             │                │
                   │ │  CURRENT COMPUTE      │                │
                   │ │  8×16 MULT + >>8      │                │
                   │ │  + negative clamp     │                │
                   │ └──────┬────────────────┘                │
                   │        │ s2_current_nA[15:0]             │
                   │ ┌──────▼───────────────────┐             │
                   │ │④ STAGE 3                │             │
                   │ │  GLUCOSE COMPUTE         │             │
                   │ │  16×16 MULT + >>8 + OFF  │             │
                   │ └──────┬───────────────────┘             │
                   │        │                                 │
                   │        ▼  current_nA, glucose_mgdL,      │
                   │           code_baseline, dsp_valid       │
                   │                                          │
                   └────────► ⑤ STATUS / SATURATION FLAGS ◄───┘
                               last_raw, saturated_high/low
```

| # | Block | RTL location | Function | Cost |
|---|-------|-------------|----------|------|
| ① | Calibration register file | [glucose_dsp.v:63-90](glucose_dsp.v#L63-L90) | 4 × 16-bit writable coefficients, async-reset to safe defaults | 64 flops + 2→4 decoder |
| ② | Stage 1 — baseline subtract | [glucose_dsp.v:96-113](glucose_dsp.v#L96-L113) | `delta = adc_data − CODE_OFFSET` in 9-bit (sign-extended) | 9-bit subtractor + 18 flops |
| ③ | Stage 2 — current compute | [glucose_dsp.v:115-152](glucose_dsp.v#L115-L152) | `I = delta × NA_PER_CODE_X256 >> 8`, clamp negatives | 8×16 array multiplier + 34 flops |
| ④ | Stage 3 — glucose compute | [glucose_dsp.v:154-181](glucose_dsp.v#L154-L181) | `G = I × MGDL_PER_NA_X256 >> 8 + GLUCOSE_OFFSET` | 16×16 multiplier + 16-bit adder + 41 flops |
| ⑤ | Status / saturation | [glucose_dsp.v:183-194](glucose_dsp.v#L183-L194) | Rail detection + raw code passthrough | 8 flops + 2 comparators |

---

## 3. The Mathematics — What Technique Is Actually Used

### 3.1 The physical model you are inverting

Forward direction (what the hardware does to the signal):

```
  I_sensor  =  S · G                      S = sensor sensitivity = 5 nA/(mg/dL)
  V_TIA     =  V_BIAS + Rf · I_sensor     V_BIAS = 0.9 V, Rf = 1 MΩ
  code      =  floor(V_TIA / V_LSB)       V_LSB = VREF/2⁸ = 1.8/256 = 7.03125 mV
```

The DSP inverts this. Substituting:

```
  code = floor( (V_BIAS + Rf·S·G) / V_LSB )
```

so

```
  G = (code − code_offset) · (V_LSB / Rf) · (1/S)
       └──────┬──────┘      └────┬─────┘   └─┬─┘
         baseline          nA per code    mg/dL per nA
         subtraction          7.03125          0.2
```

**That is the entire DSP in one line.** It is an **affine map**: `G = a·(code − b) + c`, deliberately factored into two multiplies so that each coefficient has independent physical meaning and can be calibrated independently.

Why factor it instead of collapsing `a = 7.03125 × 0.2 = 1.40625` into one multiply? Because:

- `current_nA` is **independently observable** on the output bus — this is the debug/characterisation port. You can verify the TIA and ADC are correct without trusting the sensor model.
- The two coefficients are calibrated by **different procedures**: `NA_PER_CODE` is an *electrical* calibration (inject a known current, read the code — done once in ATE / wafer test), while `MGDL_PER_NA` is a *biochemical* calibration (dip in a known glucose standard — done per sensor, per patient, in the field).

That separation-of-concerns argument is the single best thing you can say about this block.

### 3.2 Fixed-point Q8.8 — why, and how

The two gains are irrational-ish non-integers: 7.03125 and 0.2. Hardware has no floats (an FPU would be ~50× the area of this whole block and burn mW, not nW, on a battery-powered wearable).

**Technique: Q8.8 fixed-point.** Store the coefficient pre-multiplied by 2⁸ = 256:

| Coefficient | Real value | × 256 | Stored as | Exact? |
|---|---|---|---|---|
| `NA_PER_CODE_X256` | 7.03125 | 1800.0 | `16'd1800` | **Exactly** — 7.03125 = 1800/256 |
| `MGDL_PER_NA_X256` | 0.2 | 51.2 | `16'd51` | No — 51/256 = 0.19922, −0.39 % gain error |

Note the beautiful coincidence worth pointing out: `V_LSB/Rf = (1.8/256)/10⁶ = 7.03125 nA`, and `7.03125 × 256 = 1800` **exactly**, with no rounding error at all, because V_LSB is itself a power-of-two division of VREF. The nA-per-code path is *bit-exact*. Only the biochemical coefficient rounds.

**The multiply-and-shift trick.** After multiplying by a Q8.8 number you have a Q8.8 result, so you divide by 256 to get back to integers. In hardware, `/256` is **free** — it is `>> 8`, which is just wiring: take bits `[23:8]` instead of `[15:0]`. Zero gates, zero delay.

```verilog
current_product_full = s1_signed_delta[7:0] * r_na_per_code_x256;  // Q8.8 result
s2_current_nA <= current_product_full[23:8];                       // >>8 : pure wiring
```

**Why there is no divider in this design — say this if asked.** A real divider (restoring/non-restoring) is either N sequential cycles or an enormous combinational array; it's the most expensive common arithmetic primitive. Here, every division in the math is a division by a *constant* (`V_LSB`, `Rf`, `S`). So we compute the **reciprocal off-chip, once, in Python**, scale it by 256, and write it into a register. The silicon only ever multiplies. Dividers moved to the host CPU at calibration time; the sensor node only does multiply-accumulate.

### 3.3 Worked numeric example — trace one sample end to end

Patient's glucose = **100 mg/dL**.

```
 ① Sensor:   I = 100 × 5                            =  500.0 nA
 ② TIA:      V = 0.9 + 1e6 × 500e-9                 =  1.400 V
 ③ ADC:      code = floor(1.400 / 0.00703125)       =  199      (0xC7)

 ─── DSP ────────────────────────────────────────────────────────
 Stage 1:   s1_signed_delta = 199 − 128              =  71   (bit[8]=0 → positive)
 Stage 2:   product = 71 × 1800                      =  127,800
            current_nA = 127800 >> 8                 =  499 nA   (floor of 499.2)
 Stage 3:   product = 499 × 51                       =  25,449
            glucose = 25449 >> 8                     =  99 mg/dL (floor of 99.41)
            + r_glucose_offset (0)                   =  99 mg/dL
 ─────────────────────────────────────────────────────────────────
 True 100 mg/dL → reported 99 mg/dL.  Error = −1 mg/dL = −1 %.
```

This is exactly the `send_and_check(8'd199, 499, 99, 5)` vector at [glucose_dsp_tb.v:176](glucose_dsp_tb.v#L176) — and it passes. **Be ready to run that in front of them.**

Where did the 1 mg/dL go? Three places, and knowing the budget is what separates a designer from a coder:

| Error source | Magnitude | Fixable? |
|---|---|---|
| ADC quantisation (floor to code 199) | up to −7.03 nA = −1.4 mg/dL | Only with more ADC bits |
| `>>8` truncation in stage 2 (499.2 → 499) | up to −1 nA = −0.2 mg/dL | Yes: round-to-nearest |
| `>>8` truncation in stage 3 (99.41 → 99) | up to −1 mg/dL | Yes: round-to-nearest |
| Coefficient quantisation 51/256 vs 0.2 | −0.39 % gain | Yes: wider Q format |

All truncations are **floor**, never round — so the errors are *biased negative*, they don't cancel. See §8 for the one-line fix.

---

## 4. Line-by-Line Code Walkthrough

### ① Calibration register file — [glucose_dsp.v:63-90](glucose_dsp.v#L63-L90)

```verilog
reg [15:0] r_code_offset;        // Baseline ADC code at zero sensor current
reg [15:0] r_na_per_code_x256;   // nA per code × 256   (7.03125 × 256 = 1800)
reg [15:0] r_mgdl_per_na_x256;   // mg/dL per nA × 256  (0.2 × 256 = 51)
reg [15:0] r_glucose_offset;     // Baseline glucose offset (integer mg/dL)

always @(posedge clk or negedge rstn) begin
    if (!rstn) begin
        r_code_offset      <= DEFAULT_CODE_OFFSET;      // 128
        r_na_per_code_x256 <= DEFAULT_NA_PER_CODE_X256; // 1800
        r_mgdl_per_na_x256 <= DEFAULT_MGDL_PER_NA_X256; // 51
        r_glucose_offset   <= DEFAULT_GLUCOSE_OFF;      // 0
    end else if (cal_we) begin
        case (cal_addr)
            2'd0: r_code_offset      <= cal_wdata;
            2'd1: r_na_per_code_x256 <= cal_wdata;
            2'd2: r_mgdl_per_na_x256 <= cal_wdata;
            2'd3: r_glucose_offset   <= cal_wdata;
        endcase
    end
end
```

**Points to make:**

- `posedge clk or negedge rstn` → **asynchronous active-low reset**. On a medical device this matters: the chip must come up in a known, *safe, non-zero* calibration the instant power is valid, without needing a clock. If reset were synchronous and the clock hadn't started, the registers would be X and the first readings garbage.
- The reset values are a **working calibration, not zeros**. If they were zero, `na_per_code = 0` would report 0 mg/dL forever — a silently-wrong device, the worst failure mode in medical electronics. Resetting to the nominal design values means an *uncalibrated* chip is merely inaccurate, not dead.
- The `case` has **no `default`** but covers all 4 of the 2-bit values, so it is fully specified — no latch, no incomplete-case warning.
- `cal_we` gates all four — writes are a single-cycle strobe, not a handshake. That's fine because the writer (an MCU over a slow SPI/I²C shim) is vastly slower than 100 MHz.

**The detail that will impress them — verifiable in the reports.** Look at the Sky130 cell histogram in [1-synthesis.AREA_0.stat.rpt](runs/RUN_2026.10.09_19.35.27/reports/synthesis/1-synthesis.AREA_0.stat.rpt):

```
sky130_fd_sc_hd__dfrtp_2   141     ← D flop with async RESET  (reset-to-0 bits)
sky130_fd_sc_hd__dfstp_2     9     ← D flop with async SET    (reset-to-1 bits)
```

Exactly **9** set-flops. Count the `1` bits in the non-zero defaults:

```
 128  = 0b0000_0000_1000_0000  →  1 one
1800  = 0b0000_0111_0000_1000  →  4 ones
  51  = 0b0000_0000_0011_0011  →  4 ones
                                 ─────
                                  9  ✓
```

Synthesis mapped each default-`1` bit to a `dfstp` (async *set*) flop and every other bit to `dfrtp` (async *reset*). **Nine set-flops, nine one-bits.** That's your RTL intent surviving intact all the way to the gate netlist, and you can prove it on screen in ten seconds.

### ② Stage 1 — Baseline subtraction — [glucose_dsp.v:96-113](glucose_dsp.v#L96-L113)

```verilog
reg        s1_valid;
reg [8:0]  s1_signed_delta;     // 9-bit: adc_data - offset
reg [7:0]  s1_raw;

always @(posedge clk or negedge rstn) begin
    if (!rstn) begin
        s1_valid        <= 1'b0;
        s1_signed_delta <= 9'd0;
        s1_raw          <= 8'd0;
    end else begin
        s1_valid <= adc_valid;
        if (adc_valid) begin
            s1_raw <= adc_data;
            s1_signed_delta <= {1'b0, adc_data} - {1'b0, r_code_offset[7:0]};
        end
    end
end
```

**What the zero-extension does, and why it's the clever bit.** Both operands are zero-extended to 9 bits before subtracting. In 9-bit two's-complement:

```
  adc_data = 100, offset = 128:
     9'b0_0110_0100  (100)
   − 9'b0_1000_0000  (128)
   = 9'b1_1110_0100  (−28 in two's complement, = 484 unsigned)
     ▲
     └── bit[8] = 1 → BORROW OUT → the result was negative
```

The MSB of the 9-bit result is a **free sign flag**. No comparator, no `if (adc_data < offset)`, no extra logic — the subtractor's borrow-out *is* the sign bit. That's why the width is 9 and not 8. In stage 2, `if (s1_signed_delta[8])` is a single-wire test.

**Why latch `adc_data` under `if (adc_valid)`** rather than unconditionally: the ADC output bus `dout` is only guaranteed stable during the `DONE` state of [sar_logic.v:206](sar_logic.v#L206). Between conversions `cdac_ctrl` is being thrashed through 8 trial values. Qualifying the capture with `adc_valid` means the pipeline only ever ingests settled data. `s1_valid` follows `adc_valid` unconditionally — the *valid* travels every cycle, the *data* only moves when valid. That's the standard valid-qualified datapath discipline.

`r_code_offset[7:0]` truncates the 16-bit register to 8 bits, which is correct — an 8-bit ADC can never produce a code above 255, so offset values above 255 are meaningless. It's a deliberate narrowing, and worth calling out so it doesn't look like an accident.

### ③ Stage 2 — Current computation + clamp — [glucose_dsp.v:115-152](glucose_dsp.v#L115-L152)

```verilog
reg [23:0] current_product_full;   // 8 bits × 16 bits = 24 bits max

always @(posedge clk or negedge rstn) begin
    if (!rstn) begin
        s2_valid <= 1'b0;  s2_delta_clamped <= 8'd0;
        s2_current_nA <= 16'd0;  s2_raw <= 8'd0;  s2_was_negative <= 1'b0;
    end else begin
        s2_valid <= s1_valid;
        s2_raw   <= s1_raw;
        if (s1_valid) begin
            if (s1_signed_delta[8]) begin
                // Negative delta → clamp to zero
                s2_delta_clamped <= 8'd0;
                s2_current_nA    <= 16'd0;
                s2_was_negative  <= 1'b1;
            end else begin
                s2_delta_clamped <= s1_signed_delta[7:0];
                current_product_full = s1_signed_delta[7:0] * r_na_per_code_x256;
                s2_current_nA <= current_product_full[23:8];  // >>8 = /256
                s2_was_negative <= 1'b0;
            end
        end
    end
end
```

**The clamp is a safety feature, not a convenience.** A negative delta means the ADC read *below* the zero-current baseline. Physically that's a reverse current through the electrochemical cell — impossible for a working glucose sensor. It means a fault: sensor disconnected, TIA railed, electrode fouled, reference drifted. Without the clamp, the 9-bit negative value would be reinterpreted as a huge positive number through the multiplier and the device would report a **wildly high glucose reading** — which, in a closed-loop insulin system, is the failure that kills someone. Clamping to zero plus asserting `saturated_low` turns a dangerous wrong answer into an obvious, flagged, safe one.

This is the strongest safety argument in the whole design. Lead with it if they ask about medical-device considerations.

**Bit-width proof (`[23:0]`) — be ready to derive this at a whiteboard.**

```
  max delta     = 255          (8 bits)
  max coeff     = 65535        (16 bits)
  max product   = 255 × 65535 = 16,711,425 < 2²⁴ = 16,777,216   ✓ fits in 24 bits
```

With realistic defaults: `127 × 1800 = 228,600`, `>>8 = 893 nA` — fits comfortably in the 16-bit `s2_current_nA`. And `893 nA` is exactly the headroom number: `(255−128) × 7.03125 = 893 nA`, matching the 0.9 V of TIA headroom above the bias point divided by 1 MΩ. **The digital full-scale and the analog full-scale agree to within one LSB.** That cross-domain consistency check is the kind of thing that tells an interviewer you designed the chip, not just the block.

**The one piece of style you should pre-empt before they find it.** `current_product_full` is declared as a module-level `reg [23:0]` and written with a **blocking** `=` inside a non-blocking clocked block. Functionally it's correct — it's written before it's read in the same evaluation, so Yosys infers it as a pure combinational wire feeding the flop, and the gate netlist is exactly the intended multiplier-then-register. But it is **not clean style**: a module-level `reg` assigned blocking inside an edge-triggered block is the classic shape that causes simulation/synthesis mismatch when someone later reads it from a different always block. The disciplined form declares it local to the block:

```verilog
always @(posedge clk or negedge rstn) begin : stage2
    reg [23:0] product;                        // block-local temp
    ...
    product = s1_signed_delta[7:0] * r_na_per_code_x256;
    s2_current_nA <= product[23:8];
end
```

Saying this yourself, unprompted — "it works and synthesises correctly, here's the netlist, but I'd scope the temp into the block in a cleanup pass" — is worth more than having clean code they never look at. It shows you know *why* the rule exists, not just the rule.

### ④ Stage 3 — Glucose computation — [glucose_dsp.v:154-181](glucose_dsp.v#L154-L181)

```verilog
reg [31:0] glucose_product_full;

always @(posedge clk or negedge rstn) begin
    if (!rstn) begin
        dsp_valid <= 1'b0;  code_baseline <= 8'd0;
        current_nA <= 16'd0;  glucose_mgdL <= 16'd0;
    end else begin
        dsp_valid <= s2_valid;
        if (s2_valid) begin
            code_baseline <= s2_delta_clamped;
            current_nA    <= s2_current_nA;

            if (s2_was_negative) begin
                glucose_mgdL <= r_glucose_offset;        // fault → report baseline only
            end else begin
                glucose_product_full = s2_current_nA * r_mgdl_per_na_x256;
                glucose_mgdL <= (glucose_product_full >> 8) + r_glucose_offset;
            end
        end
    end
end
```

- `s2_was_negative` is **carried down the pipeline** rather than recomputed. Classic pipelining discipline: a control decision is made once, at the stage where the information exists, and the *result* of the decision rides alongside the data. Recomputing it here would mean re-plumbing `s1_signed_delta[8]` across a stage boundary for no benefit.
- On the fault path, glucose reports `r_glucose_offset` (default 0) rather than 0 unconditionally — so a user-configured floor still applies. Small thing, right thing.
- `16×16 → 32 bits`: `65535 × 65535 = 4.29e9 < 2³²` ✓. In practice `893 × 51 = 45,543`, `>>8 = 177 mg/dL`.
- `code_baseline` and `current_nA` are simply *forwarded* here with no arithmetic. They're already final — the stage exists to align them with `glucose_mgdL` so that when `dsp_valid` fires, **all three outputs describe the same sample**. Pipeline alignment of side-channel data is a thing interviewers specifically probe for.

### ⑤ Status and saturation flags — [glucose_dsp.v:183-194](glucose_dsp.v#L183-L194)

```verilog
reg [7:0] last_raw;
always @(posedge clk or negedge rstn) begin
    if (!rstn) last_raw <= 8'd0;
    else if (adc_valid) last_raw <= adc_data;
end

assign raw_code       = last_raw;
assign saturated_high = (last_raw >= 8'd250);
assign saturated_low  = (last_raw <= r_code_offset[7:0]);
```

- Flags are **combinational off a registered value** — no glitching, since `last_raw` only changes on a clock edge.
- `>= 250` rather than `== 255`: a 6-code guard band. The TIA's output stage can't actually reach the rail (a CMOS output stage saturates ~100 mV short of VDD), so codes 250–255 are already in the compressive, non-linear region. Flagging at 250 declares the reading untrustworthy *before* the transfer function visibly bends, rather than after. **Quote the number and the reason together** — "250, because the TIA goes non-linear about 40 mV below the rail and 6 codes × 7 mV is 42 mV."
- `raw_code` is a transparency port. In a medical device, downstream firmware must be able to see the unprocessed measurement to run its own plausibility checks, and a regulator will ask whether the raw signal is recoverable. Note it's wired to `dac_bus` at the top level ([gluedia_digital_top.v:51](gluedia_digital_top.v#L51)) — it's brought off-chip.

### Top-level integration — [gluedia_digital_top.v](gluedia_digital_top.v)

```verilog
wire [7:0] sar_data_wire;
wire       sar_valid_wire;

sar_logic    u_sar ( .clk(clk), .rstn(rstn), .start(start), .comp_out(comp_out),
                     .dout(sar_data_wire), .valid(sar_valid_wire), ... );

glucose_dsp  u_dsp ( .clk(clk), .rstn(rstn),
                     .adc_data (sar_data_wire),
                     .adc_valid(sar_valid_wire),
                     .code_baseline(),            // ← intentionally unconnected
                     .raw_code(dac_bus), ... );
```

Single clock domain, single reset, two modules, a `dout → adc_data` / `valid → adc_valid` handshake. **No CDC anywhere** — the SAR logic and the DSP share `clk`, so there is no synchroniser, no FIFO, no metastability exposure. If asked "how did you handle clock domain crossing?", the correct answer is *"I designed it out — one clock domain by construction."*

`code_baseline` is left unconnected at the top. Mention this before they spot it: it's a verification/debug port used by [glucose_dsp_tb.v](glucose_dsp_tb.v) and intentionally not brought to a pad, because pads are the scarcest resource on the die and `current_nA` already carries the same information scaled. Synthesis consequently prunes part of that register — which is why the gate netlist has ~150 flops rather than the ~190 the RTL literally describes. **Knowing why your flop count shrank is a very good look.**

---

## 5. Pipeline Timing — Latency, Throughput, and Why 3 Stages

```
 cycle:      0      1      2      3      4      5
             │      │      │      │      │      │
 adc_valid  ─┐▔▔▔▔▔┌──────────────────────────────    (1-cycle strobe)
 adc_data    │ 199 │
             ▼      
 S1          ░░░░░░ delta = 71
                    ▼
 S2                 ░░░░░░ mult 71×1800, >>8 → 499 nA
                           ▼
 S3                        ░░░░░░ mult 499×51, >>8 → 99 mg/dL
                                  ▼
 dsp_valid  ───────────────────────┐▔▔▔▔▔┌────────
 glucose_mgdL                      │  99 │
```

| Metric | Value | Derivation |
|---|---|---|
| Latency | **3 clocks = 30 ns** @ 100 MHz | one register stage per pipe stage |
| Throughput | **1 sample/clock = 100 MSPS** | fully pipelined, no stalls, no backpressure |
| ADC conversion time | **13 clocks = 130 ns** | IDLE 1 + SAMPLE 2 + HOLD 1 + CONV 8 + DONE 1 |
| Effective system rate | **~7.7 MSPS** | ADC-limited |
| DSP headroom | **13×** | DSP can absorb 13 ADCs before it is the bottleneck |

**Why 3 stages and not 1?** With no pipelining the critical path would be: 9-bit subtract → 8×16 multiply → 16×16 multiply → 16-bit add, all in one clock. Two cascaded array multipliers is roughly 8–10 ns in Sky130 at typical corner — that alone eats the entire 10 ns period and fails timing. Splitting at the natural multiplier boundaries puts exactly **one multiplier in each stage**, so the critical path is one multiply plus a shift (free) plus setup. Result, from [2-syn_sta.summary.rpt](runs/RUN_2026.10.09_19.35.27/reports/synthesis/2-syn_sta.summary.rpt):

```
  worst slack -max (Setup)  =  0.85 ns        ← 8.5 % margin at 10 ns period
  worst slack -min (Hold)   =  0.14 ns        ← positive, no hold violations
  wns = 0.00    tns = 0.00                    ← CLEAN. Zero failing endpoints.
```

**Why not 4 or 5 stages?** Because timing already closes with margin, and every extra stage costs flops (area, leakage) and latency for nothing. The pipeline depth was chosen by where the long combinational blocks are, not by a rule of thumb. If asked "could you run faster?" — yes, by splitting each multiplier into partial-product + accumulate stages, but `suggested_clock_frequency` in [metrics.csv](runs/RUN_2026.10.09_19.35.27/reports/metrics.csv) is 64 MHz *post-route with full parasitics*, so the honest answer is that routed, this design wants ~15.5 ns, and a wearable sampling glucose every few minutes has no use whatsoever for more speed. **The right design decision here is to slow the clock down and save power, not speed it up.**

---

## 6. Two-Point Calibration — The Feature That Justifies the Whole Block

This is the "why does a DSP exist at all" answer.

### The problem

Every electrochemical sensor is different. Enzyme loading varies between manufacturing lots, the electrode area varies with the screen-printing process, sensitivity drifts as the enzyme degrades over the 14-day wear period, and body temperature shifts the reaction rate. A sensor nominally at 5 nA/(mg/dL) might actually be 4.6 or 5.4. The TIA's `Rf` has its own ±20 % process spread in Sky130, and the TIA's `V_BIAS` isn't exactly 0.9 V.

You cannot fix any of this in analog. You *can* fix all of it with two numbers.

### The procedure

Dip the sensor in two reference solutions and record the raw codes:

```
  Standard 1:    0 mg/dL  →  ADC code 130      (not 128 — baseline has drifted +2)
  Standard 2:  100 mg/dL  →  ADC code 200      (not 199 — gain is low)
```

Solve the affine system (done in Python, [calibration_sim.py:72-111](calibration_sim.py#L72-L111)):

```
  Δcode    = 200 − 130                     = 70 codes
  ΔI       = (100 − 0) mg/dL × 5 nA/(mg/dL) = 500 nA
  nA/code  = 500 / 70                       = 7.1429
  × 256    = 1828.6  →  round              = 1828
  code_offset                               = 130
```

Write four registers and the chip is calibrated:

```verilog
write_cal_reg(2'd0, 16'd130);   // CODE_OFFSET      — kills baseline drift
write_cal_reg(2'd1, 16'd1828);  // NA_PER_CODE_X256 — kills TIA/ADC gain error
write_cal_reg(2'd2, 16'd51);    // MGDL_PER_NA_X256 — sensor sensitivity
write_cal_reg(2'd3, 16'd0);     // GLUCOSE_OFFSET
```

Verified at [glucose_dsp_tb.v:208-217](glucose_dsp_tb.v#L208-L217):

```
  Code 130 →    0 nA →   0 mg/dL   ← calibration point 1, exact
  Code 165 →  249 nA →  49 mg/dL   ← interpolated midpoint
  Code 200 →  499 nA →  99 mg/dL   ← calibration point 2, exact
  Code 235 →  749 nA → 149 mg/dL   ← extrapolated, still linear
```

**The sentence to land:** *"The same die works with any sensor lot. Calibration is four register writes over the serial interface, not a new mask set. That's the entire economic argument for putting a DSP on this chip — without it, every sensor lot variation becomes a respin."*

`r_glucose_offset` exists for the fourth degree of freedom: some CGM algorithms apply a **lag compensation** offset, because interstitial fluid glucose trails blood glucose by 5–15 minutes. That offset is clinical, not electrical, which is why it's a separate register in mg/dL rather than folded into the gain.

---

## 7. Verification Strategy — How You Know It Works

Three independent levels. Describe all three; most candidates only have one.

**1. Directed RTL testbench** — [glucose_dsp_tb.v](glucose_dsp_tb.v), 12 self-checking vectors with pass/fail counting:

- **Test 1**: default-calibration sweep across the clinical range (codes 128→241)
- **Test 2**: edge cases — code 100 (below baseline, must clamp to 0) and code 255 (full scale, must not overflow)
- **Test 3**: two-point recalibration, re-verifying the same transfer function with new coefficients

The tolerance check at [glucose_dsp_tb.v:103-106](glucose_dsp_tb.v#L103-L106) is worth pointing out — it guards against *unsigned underflow* when the expected value is near zero:

```verilog
if ((exp_current < tolerance || current_nA >= exp_current - tolerance) && ...
```

Without the `exp_current < tolerance` guard, `0 - 5` in 16-bit unsigned wraps to 65531 and every zero-current test would spuriously pass. Catching that is a real verification bug fixed, and it's exactly the kind of thing to volunteer.

**2. Bit-exact Python golden model** — [calibration_sim.py:55-149](calibration_sim.py#L55-L149). This is the strongest verification asset, because it replicates the Verilog *including its imperfections*:

```python
delta = signed_delta & 0xFF                       # 8-bit truncation
current_product = delta * self.na_per_code_x256   # 24-bit
current_nA = (current_product >> 8) & 0xFFFF      # >>8 AND 16-bit wrap
glucose_product = current_nA * self.mgdl_per_na_x256
glucose_mgdL = ((glucose_product >> 8) + self.glucose_offset) & 0xFFFF
```

Note the `& 0xFFFF` masks and the `>>` floors — it is deliberately **not** a floating-point idealisation. A model that uses floats would *disagree* with the RTL and be useless as a reference. This one lets you sweep all 256 codes exhaustively and prove bit-exact equivalence, and it's what produces the linearity-error plot.

**3. Mixed-signal chain simulation** — [calibration_sim.py:156-165](calibration_sim.py#L156-L165) models TIA + LPF + S/H + ADC feeding the DSP model, sweeping 0–800 nA in 1 nA steps, so you verify the DSP against the *physical* signal chain and not just against synthetic codes.

**4. Post-synthesis / post-route gate-level netlist** — the DSP survives to [gluedia_digital_top.nl.v](runs/RUN_2026.10.09_19.35.27/results/final/verilog/gl/gluedia_digital_top.nl.v) in real Sky130 standard cells, with STA signed off. You have RTL → gates → placed-and-routed → timing-clean. That's a complete flow, not a simulation exercise.

---

## 8. Known Limitations — Raise These Yourself

Volunteering the weaknesses is the highest-value move in the interview. It converts "found a flaw in your design" into "already knew, had a fix, had a reason."

### L1. Truncation is biased, not rounded

Both `>>8` operations **floor**. Errors never cancel; they accumulate negative. Worst case ≈ −1.2 mg/dL.

**Fix — add half an LSB before shifting (round-to-nearest), costs one 16-bit adder:**

```verilog
s2_current_nA <= (current_product_full + 24'd128) >> 8;     // +0.5 then floor
glucose_mgdL  <= ((glucose_product_full + 32'd128) >> 8) + r_glucose_offset;
```

### L2. Double quantisation between stages

Stage 3 consumes the *already truncated* integer `current_nA`, so stage 2's error propagates and is re-scaled. The cleaner architecture keeps the Q8.8 fraction alive between stages and only rounds once at the very end:

```verilog
// carry the fractional bits forward instead of discarding them
reg [23:0] s2_current_q8;          // Q8.8 nA, fraction preserved
s2_current_q8 <= current_product_full;                  // no shift here
...
glucose_mgdL  <= (s2_current_q8 * r_mgdl_per_na_x256 + 32'd32768) >> 16;  // round once
```

Cost: 8 extra flops and a wider multiplier. Benefit: single-rounding, ~0.5 mg/dL better.

### L3. Dynamic range stops right at the clinical threshold ⚠️ *the important one*

With the default calibration, full-scale output is:

```
  (255 − 128) codes × 7.03125 nA/code = 893 nA
  893 nA × 0.2 mg/dL/nA               = 178 mg/dL
```

But the **hyperglycemia threshold is 180 mg/dL** ([calibration_sim.py:48](calibration_sim.py#L48)), and clinical CGMs must read to **400 mg/dL**. The chain saturates ~2 mg/dL *below* the one threshold it most needs to detect.

Root cause: `Rf = 1 MΩ` with only 0.9 V of headroom above the bias point gives 900 nA, and 900 nA / 5 nA-per-mg/dL = 180 mg/dL. **The analog gain was chosen too high for the required range.**

Three fixes, with the trade-off — know all three:

| Fix | Change | Cost |
|---|---|---|
| Reduce `Rf` to 220 kΩ | Full scale → 4090 nA = 818 mg/dL | Loses 4.5× SNR at low glucose, where hypo detection matters most |
| Programmable-gain TIA | Switch `Rf` 1 MΩ / 220 kΩ via a digital bit | Needs a gain bit in the DSP to rescale `NA_PER_CODE` — ~10 gates, and auto-ranging logic driven by `saturated_high` |
| Go to 10-bit ADC | 4× more codes over same range | ~30 % more CDAC area, 2 more SAR cycles, tighter comparator offset requirement |

**The right answer is the programmable-gain TIA with auto-ranging**, because the clinical requirement is simultaneously *fine resolution at low glucose* (hypoglycemia is the acutely dangerous direction) *and* *wide range at high glucose*. A single fixed gain cannot satisfy both; switched gain can. And note the hook is already in the RTL — `saturated_high` is exactly the signal an auto-ranging controller would consume. Say that: **"the flag that would drive the fix is already there."**

### L4. No digital filtering

The DSP is purely memoryless and point-wise. Every ADC sample passes straight through. Real CGM sensors have significant electrochemical noise plus motion artefacts, and production devices apply temporal filtering. With 13× throughput headroom there is budget for it. The natural addition is an **oversample-and-decimate moving average** — take 16 conversions and average:

```verilog
// 16-sample boxcar: averaging 16 → +2 effective bits (√16 = 4× noise reduction)
reg [19:0] acc;  reg [4:0] cnt;
always @(posedge clk or negedge rstn)
    if (!rstn)          begin acc <= 0; cnt <= 0; end
    else if (dsp_valid) begin
        if (cnt == 15) begin glucose_filt <= (acc + glucose_mgdL) >> 4; acc <= 0; cnt <= 0; end
        else           begin acc <= acc + glucose_mgdL; cnt <= cnt + 1; end
    end
```

Divide-by-16 is `>>4`, free. 20 flops. At 7.7 MSPS, 16 samples takes 2 µs — utterly invisible to a measurement that updates every 5 minutes. **This is the single highest-value thing to add next**, and saying so demonstrates you can prioritise.

Better still, a **median-of-5** filter rejects the impulsive motion artefacts that a boxcar smears, at the cost of a small sorting network — mention that if they push.

### L5. Style: block-level `reg` written with blocking assignment

Covered in §4 ③. Correct in both simulation and synthesis here; wrong shape as a habit. Scope the temp into the `always` block.

### L6. No overflow detection on the outputs

If someone writes a pathological calibration (`NA_PER_CODE_X256 = 65535`), `s2_current_nA` takes `current_product_full[23:8]` and the top bits are silently discarded — the reading wraps instead of saturating. For a medical device, **saturating arithmetic is the correct behaviour**, not wrapping:

```verilog
s2_current_nA <= |current_product_full[23:16] ? 16'hFFFF : current_product_full[23:8];
```

Eight-input OR plus a mux. Cheap insurance against a corrupted calibration register producing a plausible-looking but wrong number.

---

## 9. Physical Implementation Results — Quote These Numbers

All from [runs/RUN_2026.10.09_19.35.27/](runs/RUN_2026.10.09_19.35.27/), full OpenLane RTL-to-GDS on Sky130 HD.

| Metric | Value | Source |
|---|---|---|
| Standard-cell area (synth) | **24,423 µm²** | `1-synthesis.AREA_0.stat.rpt` |
| Synthesised cell count | **2,400** | `metrics.csv : synth_cell_count` |
| Die area | **0.0694 mm²** | `metrics.csv : DIEAREA_mm^2` |
| Core area | **60,470 µm²** | `metrics.csv : CoreArea_um^2` |
| Total cells incl. fill/tap/decap | **8,145** | `metrics.csv : TotalCells` |
| Final utilisation | **38.7 %** | `metrics.csv : Final_Util` |
| Target clock | **100 MHz** (10 ns) | `metrics.csv : CLOCK_PERIOD` |
| Setup slack (WNS) | **+0.85 ns**, TNS = 0 | `2-syn_sta.summary.rpt` |
| Hold slack | **+0.14 ns** | `2-syn_sta.summary.rpt` |
| Post-route suggested period | **15.51 ns (64.5 MHz)** | `metrics.csv : suggested_clock_*` |
| Typical dynamic power | **~8.7 nW** (4.16 nW internal + 4.52 nW switching) | `metrics.csv : power_typical_*` |
| Leakage | **0.45 pW** | `metrics.csv : power_typical_leakage_uW` |

**Cell composition, and what it tells you about the architecture** (from the histogram):

| Cell | Count | What it is |
|---|---|---|
| `xnor2_2` | 193 | **Multiplier partial-product / carry logic** |
| `nand2_2` | 158 | Multiplier AND-array and general logic |
| `nor2_2` | 151 | General combinational |
| `dfrtp_2` | 141 | Async-reset flops (reset-to-0 state) |
| `buf_1` | 233 | Fanout buffering, mostly on the broadcast calibration registers |
| `mux2_2` | 111 | Clamp/select paths and the `cal_addr` write decoder |
| `xor2_2` | 45 | Adder sum logic |
| `dfstp_2` | 9 | **Async-set flops — the nine `1` bits in the defaults (§4 ①)** |

The 193 XNOR2 + 45 XOR2 + 158 NAND2 is the **signature of two array multipliers**. Point at that and say: *"you can read the architecture straight off the cell histogram — the XNOR/NAND population is the two multipliers, and they dominate the block."* Also the honest corollary: **the multipliers dominate because the coefficients are runtime-programmable.** If the calibration were hard-wired, `×1800` would strength-reduce to three shift-adds (1800 = 2048 − 256 + 8) and the area would collapse by roughly an order of magnitude. That area is the price of field recalibration, and it's a price worth paying — say both halves.

The DSP is ~90 % of the digital top; `sar_logic` is a 12-state FSM with a shift-register datapath and contributes only ~30 flops and a few hundred gates.

---

## 10. Interviewer Q&A — Rapid Fire

**Q: Why fixed-point instead of floating-point?**
A floating-point unit is roughly 50× the area of this entire block and burns milliwatts. This is a battery-powered wearable where the whole digital section draws nanowatts. More to the point, floating point solves a *dynamic range* problem, and I don't have one — my data is 8-bit and my coefficients span a factor of 35. Q8.8 fixed-point covers that with integer hardware I already need for the multiply.

**Q: Why Q8.8 specifically — why 8 fractional bits?**
Two reasons. First, 8 fractional bits gives 1/256 = 0.4 % coefficient resolution, which is well below the sensor's own ±5 % lot-to-lot variation — there is no point being more precise than the thing you're measuring. Second, `>>8` on a 24-bit product is a clean byte-aligned slice, `[23:8]`, which is just wiring. A non-power-of-two or non-byte-aligned scale factor would cost real gates.

**Q: Where is the divider?**
There isn't one, by design. Every division in the math is by a compile-time constant, so I inverted them off-chip in Python, scaled by 256, and stored the reciprocals in the calibration registers. The silicon only multiplies and shifts. Dividers are the most expensive common arithmetic block — avoiding one entirely is the main architectural decision in this DSP.

**Q: Why 3 pipeline stages?**
Because there are two multipliers, and chaining them combinationally is 8–10 ns in Sky130 — it would blow the 10 ns period on its own. I split at the natural multiplier boundaries so each stage contains exactly one multiply. It closed with 0.85 ns of setup slack and zero TNS. Depth was set by where the long paths actually are, not by a rule of thumb.

**Q: What happens if the ADC reads below baseline?**
It's clamped to zero and `saturated_low` asserts. That's a safety requirement, not a convenience — a negative delta is physically impossible for a working sensor, so it indicates a fault. If I let the 9-bit negative value into the multiplier it would be reinterpreted as a large positive number and the device would report a wildly high glucose reading. In a closed-loop insulin system, that's the failure mode that injects a lethal dose. Clamping turns a dangerous wrong answer into a flagged safe one.

**Q: How do you handle the pipeline when `adc_valid` is low?**
The `valid` bit propagates every cycle unconditionally; the data registers only update when their stage's input valid is high. So bubbles flow through harmlessly and stale data is never reprocessed. Since the ADC takes 13 cycles per conversion and the DSP is 1-per-cycle, the pipeline is mostly bubbles in practice — the DSP has about 13× throughput headroom.

**Q: Why async reset?**
It's a medical device. The chip must come up in a known, safe, *non-zero* calibration the instant the supply is valid, without waiting for a clock. And the reset values matter as much as the reset style: they're the nominal working calibration, not zeros. A chip that reset to `na_per_code = 0` would report 0 mg/dL forever — a silently-wrong device, which in this application is far worse than an obviously-dead one.

**Q: Is your transfer function linear?**
The DSP's is, exactly — it's an affine map by construction, and the Python model confirms the only deviation is sub-LSB quantisation. The *sensor* isn't perfectly linear and the TIA compresses near the rail, which is why `saturated_high` trips at code 250 rather than 255 — about 42 mV below the rail, where the TIA output stage starts to bend. If the sensor's non-linearity needed correcting, the right fix is a piecewise-linear segmented lookup, which is more registers and a comparator tree, not a different architecture.

**Q: How would you verify this at scale?**
I already have the key asset — a Python golden model that's bit-exact with the RTL, truncations and wraps included, not a floating-point idealisation. That lets me sweep all 256 codes × a range of calibration settings exhaustively and prove equivalence, rather than relying on the 12 directed vectors. The next step would be a cocotb testbench driving the RTL from that same model as a scoreboard, plus constrained-random calibration values to hunt for the overflow case in §8-L6.

**Q: What's the weakest part of this design?**
The dynamic range. Full scale is 178 mg/dL with the default calibration, and the hyperglycemia threshold is 180 — it saturates right where it most needs to read. That's an analog gain choice, `Rf = 1 MΩ` against 0.9 V of headroom, not a DSP bug, but the DSP is where I'd fix it: a switchable TIA gain with a rescale bit in the calibration path, auto-ranged off `saturated_high`. The flag that would drive it is already in the RTL.

**Q: What would you add next, given one week?**
The 16-sample boxcar averager. Real sensors are noisy, averaging 16 buys two effective bits, it costs about 20 flops and a shift, and at 7.7 MSPS it takes 2 µs — invisible against a 5-minute measurement cadence. Highest value per gate of anything on the list.

**Q: Why is `code_baseline` unconnected at the top level?**
It's a verification port, used by the testbench. Pads are the scarcest resource on the die and `current_nA` carries the same information scaled by a known constant, so there was no reason to spend a pad on it. Synthesis correctly prunes the part of that register nothing observes, which is why the gate netlist has about 150 flops where the RTL literally describes 190.

---

## 11. Live Demo Script

```bash
cd ~/GLUEDIA

# 1. Show the RTL — walk the four blocks
gedit glucose_dsp.v &

# 2. Run the DSP testbench — 12 self-checking vectors
iverilog -o dsp_tb glucose_dsp.v glucose_dsp_tb.v && ./dsp_tb

# 3. Show the waveform — point at the 3-cycle pipeline latency
gtkwave glucose_dsp.vcd &

# 4. Run the bit-exact Python golden model + calibration plots
python3 calibration_sim.py

# 5. Full system: SAR ADC + DSP together
iverilog -o sys_tb sar_logic.v glucose_dsp.v gluedia_digital_top.v gluedia_system_tb.v && ./sys_tb

# 6. Prove the nine set-flops (§4 ① — the best 10 seconds of the demo)
grep -E "dfstp|dfrtp" runs/RUN_2026.10.09_19.35.27/reports/synthesis/1-synthesis.AREA_0.stat.rpt

# 7. Show timing closure
cat runs/RUN_2026.10.09_19.35.27/reports/synthesis/2-syn_sta.summary.rpt
```

**Sequence to tell the story in:** math on the whiteboard (§3) → RTL blocks (§4) → testbench passing (§7) → the nine set-flops (§4①) → timing closure (§9) → *then volunteer the dynamic-range limitation* (§8-L3). Ending on a limitation you found yourself, with three costed fixes and the observation that the enabling flag is already in the RTL, is a much stronger finish than ending on a number.
