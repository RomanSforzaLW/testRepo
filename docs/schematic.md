# LIPUS + PEMF schematic (text-level)

This document captures a **schematic-level** wiring plan for the current combined **LIPUS (0.5/1.0/1.5 MHz + sweeps)** and **PEMF** concept.

> Notes
> - Pin numbers are intentionally omitted unless we have the exact package/datasheet open; use **signal names** and confirm pin mapping during schematic capture.
> - LIPUS is **non-focused / collimated or slightly divergent** per customer request; that requirement is primarily **transducer + acoustic stack**, not the MOSFET driver topology.

---

## Power rails (logical)

- `VBAT`: Li-ion battery (capacity target: **2500 mAh**)
- `VLOGIC_3V3`: 3.3 V logic rail (MCU, low-voltage logic)
- `VISO_3V3` or `VISO_5V`: isolated logic rail for isolator side-2 / driver inputs (choose during detailed design)
- `VDRV_POS`: positive power rail for the LIPUS half-bridge (commonly **+20 V to +24 V**)
- `VDRV_NEG`: negative power rail for the LIPUS half-bridge (**-20 V**)
- `VGD`: gate-driver supply rail for MD1210 logic/drive (commonly **~10–12 V** referenced to driver domain; exact requirement depends on the chosen MD1210 variant and MOSFET gate target)

Recommended decoupling intent (place physically at each consumer):
- **Each IC**: `0.1 µF` + `1 µF` close to supply pins.
- **LIPUS bridge rails near MOSFETs**: `0.1 µF` + `1 µF` + `10 µF` MLCC per rail pair (low ESL), plus local bulk as needed after measurement.
- **PEMF pulse rail**: a local **bulk capacitor bank** near the coil switch to source pulse current.

---

## Power supply design (architecture-level)

### Do we have enough information?

- **Enough to define the topology**: yes (what rails exist, isolation partitioning, sequencing, filtering strategy).
- **Not enough to finalize component values**: not yet. To size converters/inductors/bulk caps confidently we still need:
  - **Piezo**: active area \(A\), impedance vs frequency (or equivalent model) across **0.5–1.5 MHz**
  - **LIPUS protocol**: envelope PRF, duty cycle, max session “ON” fraction, and how “sweep” is scheduled (continuous chirp vs step sweep)
  - **PEMF**: peak coil current, pulse width, repetition rate, allowed droop, and whether PEMF and LIPUS can run simultaneously
  - **Battery configuration**: 1S vs 2S, minimum VBAT before cutoff, charge-path constraints (operate while charging or not)

Even without those, we can implement a correct **rail tree** and leave “rating placeholders” to be filled after measurement.

### Recommended rail tree (block diagram)

```text
VBAT (Li-ion) 
  |
  +--> Buck/LDO -> VLOGIC_3V3 (nRF5340, low-voltage logic)
  |
  +--> Boost #1 -> VDRV_POS (+20..+24V)  -----> LIPUS bridge + PEMF pulse rail (optional share)
  |
  +--> Inverter/IBB -> VDRV_NEG (-20V)  ------> LIPUS bridge negative rail
  |
  +--> (Option) Boost or Buck -> VGD (~10..12V) -> MD1210 gate-drive supply (if required by chosen MD1210 configuration)
  |
  +--> (Option) Isolated DC/DC -> VISO_3V3/5V -> ISO7720DR side-2 supply (if true isolation is required beyond signal isolation)
```

### Sharing vs splitting the +HV rail

- **Option 1 (shared +HV rail)**: `VDRV_POS` also serves as `VPEMF`.
  - Pros: fewer converters, simpler BOM.
  - Cons: PEMF pulses inject droop/EMI onto LIPUS rail; needs stronger filtering and bulk capacitance.

- **Option 2 (split rails)**: separate `VPEMF` converter/cap bank from `VDRV_POS`.
  - Pros: easier to keep LIPUS clean at 0.5–1.5 MHz sweeps.
  - Cons: more parts.

### Bulk capacitor sizing method (PEMF pulse rail)

To limit droop during a PEMF pulse, use:

\[
\Delta V \approx \frac{I_{pulse}\cdot t_{pulse}}{C_{bulk}}
\]

Example placeholder (replace with real PEMF numbers):
- If \(I_{pulse}=8A\), \(t_{pulse}=0.5ms\), and you allow \(\Delta V=1V\),
  - \(C_{bulk} \approx 8A \cdot 0.5ms / 1V = 4000\ \mu F\)

That number is why PEMF rails usually use a **local electrolytic/polymer bank** close to the coil switch, plus ceramics for edge current.

### LIPUS rail stability and decoupling

The LIPUS half-bridge at 0.5–1.5 MHz needs:
- **very small high-frequency loops** (MLCCs tight to MOSFETs/driver)
- enough **local energy** that the rail doesn’t “bounce” during burst edges

Practical schematic pattern near the half-bridge:
- across `VDRV_POS` to `VDRV_NEG`: `C_HF1 100nF` + `C_HF2 1uF` + `C_HF3 10uF` (high-voltage MLCCs, derated)
- plus optional local bulk on `VDRV_POS` (and/or `VDRV_NEG`) depending on measured rail impedance

### Sequencing / enables (recommended)

Define explicit enable nets so faults can shut down power, not just gate-drive:
- `EN_HV_POS`: enable for `VDRV_POS` converter
- `EN_HV_NEG`: enable for `VDRV_NEG` converter
- `EN_VGD`: enable for gate-driver rail (if separate)
- `LIPUS_OE`: MD1210 output enable (fast)

Suggested behavior:
- Bring up `VLOGIC_3V3` first
- Then enable `VDRV_POS` and `VDRV_NEG`, wait for rails “power-good”
- Then release `LIPUS_OE`
- On fault: assert `LIPUS_OE` immediately, then disable `EN_HV_*` if the fault persists (prevents DC bias / heating)

### Switching frequency / sync strategy (conceptual)

If you synchronize multiple switchers, avoid generating intermodulation near the LIPUS carrier sweep band.
- Keep DC/DC switching frequency well away from 0.5–1.5 MHz **or** synchronize everything to a known frequency and manage filtering.
- Exact feasibility depends on the chosen converter(s) and their sync range.

### Power budget linkage to SATA 500 mW/cm²

To translate customer “SATA 500 mW/cm²” into electrical rail current, you need:
- transducer area \(A\)
- efficiency \(\eta\)

\[
P_{acoustic} = 0.5\ \mathrm{W/cm^2}\cdot A
\qquad
P_{electrical} \approx \frac{P_{acoustic}}{\eta}
\]

From \(P_{electrical}\) you can derive expected average `VDRV_POS`/`VDRV_NEG` rail currents over the burst schedule and size the converters.

---

## USB-C (PD) + Li-ion battery charging / power-path (architecture-level)

You can design this with **1-cell (1S)** or **2-cell (2S)** Li-ion. USB-C **Power Delivery (PD)** is a good fit because you can request a higher input voltage (e.g., 9 V / 12 V / 15 V) to reduce cable loss and make conversion more efficient.

### Key decision: 1S vs 2S

- **1S (single cell, 4.2 V max)**:
  - Pros: simplest charging, easiest protection, widest IC choice, simpler balancing (none).
  - Cons: your +HV rails (+20…+24 V and -20 V) require higher conversion ratio from VBAT.
- **2S (two cells in series, 8.4 V max)**:
  - Pros: easier/more efficient to generate +20…+24 V, more headroom for pulse loads.
  - Cons: charging is more complex; you must address **cell balancing** (either use a protected/balanced 2S pack or add monitoring/balancing circuitry).

### Recommended functional blocks (common to both)

1. **USB-C receptacle** (USB2 is enough unless you need data)
2. **ESD protection** on CC pins, D+/D-, and VBUS
3. **PD sink controller** to negotiate voltage/current (or fixed 5 V if no PD)
4. **Input protection / power-path front end**
   - eFuse / OVP / inrush limiting on `VBUS_PD`
   - optional ideal diode OR-ing if you support multiple inputs
5. **Battery charger with power-path** (“NVDC” style) so the system can run from USB while charging
6. **Battery protection** (pack PCB or on-board protector) + optional fuel gauge

### Block diagram: 1S with USB-PD

```text
USB-C Receptacle
  |
  +-- ESD (CC1/CC2, D+/D-, VBUS)
  |
PD Sink Controller  ---> negotiates PDO (ex: 9V/12V)
  |
VBUS_PD (5..20V depending on contract)
  |
eFuse / OVP / inrush limiting
  |
1S Buck Charger + Power-Path (SYS node)
  |            \
  |             +--> SYS (system input) -> downstream rails (3V3, +HV, -HV, etc.)
  |
BAT (1S cell, 4.2V max) + protector (pack or on-board)
```

**Typical 1S IC categories** (choose based on your input range and desired features):
- **PD sink controller** (standalone contract): e.g., STUSB4500-class, TI TPS25750-class, or a PD PHY (FUSB302-class) with MCU PD stack.
- **Input protection**: eFuse/OVP with programmable current limit and fast short protection.
- **1S charger + power-path**: “switch-mode buck charger with power-path” (supports input up to at least your negotiated PD voltage).

### Block diagram: 2S with USB-PD

```text
USB-C Receptacle
  |
  +-- ESD
  |
PD Sink Controller  ---> request 9V/12V/15V
  |
VBUS_PD
  |
eFuse / OVP / inrush limiting
  |
2S Charger + Power-Path (buck or buck-boost, depending on PD range)
  |            \
  |             +--> SYS -> downstream rails
  |
BAT (2S series pack, 8.4V max)
  |
Cell protection + (balancing strategy required)
```

### Cell balancing (2S requirement)

If you go **2S**, pick one:
- **Use a 2S pack that already includes** protection + balancing PCB (simplest integration).
- **Add a battery monitor/balancer** (more design work; better visibility into cell health).

### “Operate while charging” (power-path behavior)

Strongly recommended for a handheld device:
- Use a charger that provides a **system node (`SYS`)** regulated from the adapter while also charging the battery.
- Define system policy for high loads:
  - If PEMF + LIPUS peak load is high, you may need to **limit charging current** during therapy sessions (thermal + adapter limits).
  - Consider firmware control of `I_CHG`/`I_IN` limits (many chargers support I²C control).

### PD contract recommendations (practical)

- If you can, request **9 V or 12 V** rather than 5 V:
  - lower input current for the same power
  - easier EMI/thermal management in the cable and connector
- Keep a safe fallback to **5 V default** (device must not overdraw before PD contract).

### Charging current guidance (2500 mAh pack)

Without your thermal model we can’t “lock” the charge rate, but typical design points:
- **0.5C**: ~1.25 A charge current (gentler thermally)
- **1.0C**: ~2.5 A (often thermally limited in compact enclosures)

You’ll likely set:
- `I_IN_MAX` based on negotiated PD current and connector thermal
- `I_CHG_MAX` based on battery spec + enclosure temperature rise

### Nets to add to the schematic (recommended)

- USB/PD:
  - `USB_VBUS`, `USB_CC1`, `USB_CC2`, `USB_D+`, `USB_D-`, `VBUS_PD`
  - `PD_INT` / `PD_I2C` (if used), `VBUS_DISCH` (if implemented)
- Charger/power-path:
  - `SYS_IN` (system power from adapter/charger)
  - `BAT+` (and `BAT-`), `TS` (battery NTC), `CHG_STAT`, `PGOOD`
  - `EN_CHG`, `ILIM_SET` (or I²C-controlled limits)
- Safety:
  - `SHIP_MODE` (if supported), `PACK_PRES` (if using removable pack)

---

## LIPUS driver schematic (net-level)

### Functional blocks
- MCU: `nRF5340`
- Burst gating: dual AND gate (e.g., `SN74LVC2G08` or equivalent)
- Isolation: dual-channel digital isolator `ISO7720DR` (2 forward channels)
- Gate driver: `MD1210` (dual independent channels + `OE`)
- Power FETs: two N-MOSFETs (e.g., `BSZ097N04LS(G)` or equivalent), half-bridge
- Output match: series `L_MATCH` (and optional damping components)
- Load: piezo transducer

### Signal naming
- `LIPUS_PWM_H`: high-side PWM (carrier, complementary)
- `LIPUS_PWM_L`: low-side PWM (carrier, complementary)
- `LIPUS_ENV`: envelope (burst enable, e.g., ~2 kHz PRF)
- `LIPUS_H_GATED`: gated high-side PWM
- `LIPUS_L_GATED`: gated low-side PWM
- `LIPUS_OE`: output enable for MD1210 (hardware-kill input)

### Text schematic

```text
                     LOGIC DOMAIN (3V3)                           POWER / DRIVER DOMAIN

  +-------------------+      +-------------------+      +-------------------+
  | nRF5340           |      | AND GATE (x2)     |      | ISO7720DR (2ch)   |
  |                   |      | (gate both PWMs   |      | (forward channels)|
  |  LIPUS_PWM_H ----+------>| A1            Y1 |----->| IN1           OUT1|-----> LIPUS_H_GATED_ISO
  |  LIPUS_PWM_L ----+---+-->| A2            Y2 |--+-->| IN2           OUT2|-----> LIPUS_L_GATED_ISO
  |  LIPUS_ENV  ---------+---| B1,B2 (shared)   |  |   |                   |
  |  FAULT_KILL ---------+-------------------------+---| (optional: isolate)|-----> LIPUS_OE_ISO (optional)
  +-------------------+                         |      +-------------------+
                                                |
                                                | (Envelope is fed to BOTH AND gates)
                                                v

                                      +-------------------+
                                      | MD1210            |
                                      |                   |
  LIPUS_H_GATED_ISO ----------------->| INA        OUTA   |----RgA----> Gate(QH)
  LIPUS_L_GATED_ISO ----------------->| INB        OUTB   |----RgB----> Gate(QL)
  LIPUS_OE (from protection logic) -->| OE                |
                                      | Supplies: VGD / VDRV_NEG (per datasheet) |
                                      +-------------------+

       VDRV_POS (+20..+24V)
            |
           Drain
          +---+
          |QH |  N-MOSFET (high-side)
          +---+
            |
            +------ SW_NODE ---- L_MATCH ---- PIEZO ---- return (see below)
            |
          +---+
          |QL |  N-MOSFET (low-side)
          +---+
           Source
            |
       VDRV_NEG (-20V)
```

### LIPUS piezo connection (bipolar drive)

Two common options (pick one during capture, based on transducer construction and EMI strategy):

**Option A (single-ended piezo return to mid-reference):**
- `SW_NODE -> L_MATCH -> PIEZO(+)`
- `PIEZO(-) -> driver reference / shield node` (define explicitly; often tied to a quiet reference plane for shielding)

**Option B (differential across piezo via bridge node + defined return):**
- Still uses `SW_NODE` as the excitation node, but you define piezo return to a controlled reference and provide damping if needed.

**Key point**: because the carrier **sweeps 0.5–1.5 MHz**, `L_MATCH` cannot perfectly resonate at all points; plan for:
- a **wider-band match** (intentional damping / Q control), or
- **switchable matching** (select L/C sets per band), if efficiency must be more uniform.

### LIPUS protection hooks (schematic hooks)
- **Hardware kill**: comparator output drives `LIPUS_OE` (and/or disables the LIPUS rail converter enable pins).
- **Current sense** (recommended for shorts / cracked piezo / inductor fault):
  - place shunt + sense amp in the LIPUS rail path (final placement depends on where you want to observe current).
  - feed both MCU ADC (telemetry) and comparator (fast trip).

---

## PEMF driver schematic (net-level)

### Functional blocks
- MCU: `nRF5340`
- Gate driver: low-side MOSFET gate driver (SMD)
- Switch MOSFET: N-MOSFET (SMD power package)
- Coil: custom pancake coil (OD/ID/thickness constraints)
- Flyback clamp: diode (or diode + TVS / RC snubber as needed)
- Optional current sense: shunt + amplifier/ADC or comparator

### Signal naming
- `PEMF_EN`: on/off or PWM for pulse timing (10–100 Hz)
- `PEMF_GATE`: driven gate signal from gate driver

### Text schematic

```text
                 LOGIC DOMAIN (3V3)                       POWER DOMAIN (HV PULSE RAIL)

  nRF5340 GPIO: PEMF_EN -----> Gate Driver IN

  Gate Driver:
    VDD  -> driver supply (e.g., 10–12V or as chosen)
    GND  -> power ground (star-point strategy)
    OUT  -> PEMF_GATE

                             +VPEMF (typically same as VDRV_POS or separate)
                               |
                               +----+---- PEMF_COIL ----+----+
                                    |                   |    |
                                    |                  D_FLY  |
                                    |               (flyback) |
                                    |                   |    |
                                    +---- Drain(QP) ----+----+
                                           |
                                          Source(QP)
                                           |
                                         R_SHUNT (optional)
                                           |
                                         PGND (power ground)
```

Flyback diode orientation:
- For a **low-side switch**, diode is placed across the coil so that when QP turns **off**, coil current recirculates safely.

---

## Coil manufacturing spec (reference)

This is carried here so the schematic and mechanical constraints stay aligned:

- **Type**: flat multi-layer pancake (spiral)
- **Dimensions**: OD **44.0 mm max**, ID **22.0 mm**, thickness **3.5–4.5 mm**
- **Winding**: **45 turns**, **5–6 layers**
- **Wire**: **22 AWG** enameled copper magnet wire, **Class H (180°C) or higher**
- **Targets**: L @10 kHz **110–140 µH**, DCR **< 0.3 Ω**
- **Operation**: **7–10 A peak**, square-wave drive, rise time **< 500 µs** @ ~24 V drive
- **Leads**: 100 mm, 20 AWG stranded, pre-tinned

---

## What’s needed to finalize component values (not topology)

To convert this from “net-level schematic” to “fully valued schematic”, we need the piezo parameters and performance target mapping:
- Piezo **active area** \(A\) (cm²)
- Piezo impedance model vs frequency (or at least \(C_0\), \(R_m\), \(L_m\), \(C_m\) near 0.5/1.0/1.5 MHz)
- Required **SATA 500 mW/cm²** mapping to electrical drive (efficiency estimate or measured transfer)
- Allowed supply droop and thermal constraints (to size bulk caps and rail currents)

