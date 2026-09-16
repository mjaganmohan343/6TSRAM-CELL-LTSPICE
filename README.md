# 6T SRAM Cell — LTspice Design (TSMC 0.18 µm)

This project implements and simulates a **single 6T SRAM bit-cell** with its
supporting **precharge**, **write driver**, and **sense amplifier** circuitry,
using the TSMC 018 (0.18 µm) BSIM3v3.1 process models in LTspice.

---

## 1. Files in this project

| File          | Purpose                                                              |
|---------------|-----------------------------------------------------------------------|
| `6TRAM.asc`   | Top-level LTspice schematic (netlist description of the full circuit) |
| `tsmc018.lib` | TSMC 0.18 µm BSIM3v3.1 SPICE model file (`CMOSN` / `CMOSP` devices)    |
| `cmosn.asy`   | LTspice symbol for the NMOS device, pointing to `CMOSN` in the .lib    |
| `cmosp.asy`   | LTspice symbol for the PMOS device, pointing to `CMOSP` in the .lib    |

The schematic includes the model file directly:
```
.include tsmc018.lib
.tran 80n
```
So the simulation runs an 80 ns transient analysis.

---

## 2. Circuit overview

The design has three functional blocks:

1. **6T SRAM core cell** — the actual 1-bit storage element (cross-coupled
   inverters + 2 access transistors).
2. **Precharge + write path** — circuitry that precharges the bitlines and
   drives new data onto them during a write.
3. **Sense amplifier (read path)** — a second cross-coupled latch that senses
   the small differential voltage on the bitlines during a read and
   regenerates it to full-swing digital levels.

### 2.1 Core 6T SRAM Cell (M1–M6)

| Device | Type | W/L         | Role                                   |
|--------|------|-------------|-----------------------------------------|
| M1     | PMOS | 540 nm/180 nm | Pull-up, inverter 1 (drives node **Q**) |
| M2     | PMOS | 540 nm/180 nm | Pull-up, inverter 2 (drives node **Q_bar**) |
| M3     | NMOS | 1.08 µm/180 nm | Pull-down, inverter 1 (drives **Q**)  |
| M4     | NMOS | 1.08 µm/180 nm | Pull-down, inverter 2 (drives **Q_bar**) |
| M5     | NMOS | 720 nm/180 nm | Access transistor, gated by **Word**, connects **Q_bar** side to a bitline |
| M6     | NMOS | 720 nm/180 nm | Access transistor, gated by **Word**, connects **Q** side to **Bit** |

- M1/M3 and M2/M4 form the two cross-coupled CMOS inverters that store the
  bit at nodes **Q** and **Q_bar**.
- M5/M6 are the access (pass-gate) transistors. They are turned on together
  by the **Word** (wordline) signal and connect the internal storage nodes to
  the bitlines so the cell can be read or written.
- Standard 6T sizing ratio is respected: pull-down NMOS (1.08 µm) is wider
  than the access NMOS (720 nm), which is wider than the pull-up PMOS
  (540 nm) — this preserves both write margin (access transistor must be
  able to overpower the pull-up) and read stability (pull-down must be
  stronger than the access device).

### 2.2 Write driver (M7, M8)

| Device | Type | W/L | Role |
|--------|------|-----|------|
| M7 | NMOS | 720 nm/180 nm | Pulls the **Bit_bar**-side node low, gated by **Write** |
| M8 | NMOS | 720 nm/180 nm | Pulls the **D_bar** node, gated by **Write** |

- `VD` (PULSE source) supplies the data to be written; `VD_bar` supplies its
  complement.
- When **Write** is asserted and **Word** is high, M7/M8 (together with the
  access transistors M5/M6) force the bitlines to the new data value, which
  overpowers the cross-coupled latch and flips the stored bit.

### 2.3 Precharge network (M9, M10, VPC)

| Device | Type | W/L | Role |
|--------|------|-----|------|
| M9  | PMOS | 540 nm/180 nm | Precharges the **Bit** line to VDD |
| M10 | PMOS | 540 nm/180 nm | Precharges the **Bit_bar** line to VDD |

- `VPC` is an active-low precharge pulse. Before every read/write access the
  bitlines are pulled up to `VDD` (1.8 V) through M9/M10 so that only a
  small differential needs to develop during the actual access.
- `C1` and `C2` (100 fF each) model the parasitic capacitance of the long
  bitlines — this is what causes the finite bitline discharge time that the
  sense amplifier must resolve.

### 2.4 Sense amplifier (M11–M16)

| Device | Type | W/L | Role |
|--------|------|-----|------|
| M11, M12 | PMOS | 540 nm/180 nm | Cross-coupled pull-up pair of the sense-amp latch |
| M13, M14 | NMOS | 1.08 µm/180 nm | Cross-coupled pull-down pair of the sense-amp latch |
| M15 | PMOS | 540 nm/180 nm | Enables the latch's PMOS side, gated by **SE_bar** |
| M16 | NMOS | 720 nm/180 nm | Enables the latch's NMOS side (footer), gated by **SE** |

- This is a second cross-coupled inverter pair (same topology as the storage
  cell) used as a **latch-type sense amplifier**.
- It taps the differential voltage that has developed on **Bit/Bit_bar**
  after the bitlines are allowed to discharge slightly through the accessed
  cell, and regenerates it into a full-rail digital output (**Bit** /
  **Bit_Bar** output pins).
- **SE** (sense enable) and **SE_bar** (its complement) turn the latch on
  only during the sensing window, saving power the rest of the time.

---

## 3. Key signals / nets

| Net | Description |
|-----|-------------|
| `VDD` | Supply, 1.8 V |
| `Word` | Wordline — turns ON access transistors M5/M6 for the addressed row |
| `Write` | Write-enable — turns ON the write-driver transistors M7/M8 |
| `PC` | Precharge control for the bitline precharge PMOS (M9/M10) |
| `Bit`, `Bit_bar` | True/complement bitlines connected to the cell through the access transistors |
| `D_bar` | Complementary data input into the write path |
| `Q`, `Q_bar` | Internal storage nodes of the 6T cell (the actual stored bit and its complement) |
| `SE`, `SE_bar` | Sense-amplifier enable and its complement |
| `Bit` / `Bit_Bar` (sense-amp outputs) | Full-swing, latched read-out of the sensed bit |

## 4. Stimulus sources

| Source | Waveform | Purpose |
|--------|----------|---------|
| `VD` | PULSE(0 1.8 1n 100p 100p 10n 20n) | Data to be written |
| `VD_bar` | PULSE(1.8 0 0 2p 2p 10n 20n) | Complementary data |
| `V1` (Write) | PULSE(0 1.8 1n 100p 100p 2n 10n) | Write-enable pulse |
| `V3`/`V4` (Word) | PULSE(0 1.8 1n/6n 100p 100p 2n 5n/1n 10n) | Wordline activation for write/read access |
| `VPC` | PULSE(1.8 0 3n 100p 100p 1n 10n) | Active-low precharge pulse |
| `VSE` / `VSE_bar` | PULSE(0 1.8 7n …) / PULSE(1.8 0 7n …) | Sense-amp enable, timed after the bitline develops its differential |
| `V2` | DC 1.8 V | VDD supply |

The timing is staggered (precharge → wordline/write assert → sense-amp
enable) to emulate a realistic SRAM read/write cycle within the 80 ns
transient window.

## 5. How to simulate

1. Open `6TRAM.asc` in LTspice (place `tsmc018.lib`, `cmosn.asy`, and
   `cmosp.asy` in the same folder, or in LTspice's `lib\sym` / working
   directory so the symbol and model references resolve).
2. Run the simulation (the `.tran 80n` directive is already in the
   schematic).
3. Recommended traces to probe:
   - `V(Word)`, `V(Write)`, `V(PC)` — control timing
   - `V(Q)`, `V(Q_bar)` — stored bit and its complement (should flip during
     the write pulse and hold state otherwise)
   - `V(Bit)`, `V(Bit_bar)` — bitline voltages (watch the precharge-then-small-
     differential-discharge behavior during a read)
   - `V(SE)`, output of the sense amp — confirms correct regeneration of the
     bitline differential into a full logic level

## 6. Design notes / things worth checking

- **Read stability**: verify `Q`/`Q_bar` do not flip during a read access
  (the pull-down NMOS should stay stronger than the access NMOS at all
  corners).
- **Write margin**: verify the write-driver + access transistor path can
  successfully flip the cell against the pull-up PMOS.
- **Sense margin**: check the bitline differential at the moment `SE` is
  asserted is large enough for the sense amp to resolve correctly given the
  100 fF bitline capacitance.
- Model file corresponds to **TSMC 0.18 µm** process, lot `T58F`, wafer
  `9005` (BSIM3v3.1, SPICE3f5 Level 8 / HSPICE Level 49).
