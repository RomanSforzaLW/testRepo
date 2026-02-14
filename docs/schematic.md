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

