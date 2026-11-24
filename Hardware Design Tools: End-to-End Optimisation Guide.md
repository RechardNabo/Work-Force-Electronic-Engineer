# Hardware Design Tools: End-to-End Optimisation Guide

A reference of tools for catching design errors early, from schematic to bring-up. Pricing and features change, so verify on each vendor's site before committing.

---

## 1. Schematic Checks

### KiCad ERC (built in)
- Flags unconnected pins, conflicting pin types (two outputs on one net), and power pins not driven.
- Quality depends on **correct pin electrical types** in your symbols. Review custom symbols.
- Tip: treat every ERC warning as an error before review; add explicit `no_connect` flags and PWR_FLAG where intended.

### KiBot
- One YAML config runs ERC, DRC, BOM, gerbers, PDFs, and 3D renders.
- Works in CI (GitHub Actions), so every commit is checked automatically.
- Good for enforcing "no release unless ERC is clean".

### kicad-cli
- Official command-line interface (KiCad 8+). Scripts ERC/DRC, netlist, BOM, and plot export.
- Use when you want lightweight scripting without KiBot.

### Netlist review script (custom)
A small Python script on the exported netlist can flag:
- Nets that differ only by case (`Input1` vs `INput1`).
- Nets with exactly one pin (likely dangling).
- Global labels with the same name in different hierarchical sheets.
- Parts with missing reference designators (`R?`, `D?`).

---

## 2. Part Lifecycle and Sourcing (EOL / NRND)

| Tool | Use | Cost |
|---|---|---|
| **KiCost** | Scrapes distributor pricing and stock from a KiCad BOM | Free, open source |
| **Octopart** | Lifecycle status, stock, alternates across distributors | Free search |
| **Findchips** | Similar to Octopart, plus price history | Free search |
| **Digi-Key / Mouser BOM Manager** | Upload BOM, see lifecycle flags and stock | Free |
| **Z2Data / SiliconExpert** | Deep obsolescence and compliance data | Paid, enterprise |

**Good practice**
- Add `Lifecycle` and `Alternate MPN` columns to the BOM.
- Example: MCP2551 can have an alternate such as MCP2562 or TJA1051 (check pin and voltage compatibility before swapping).
- Re-run the lifecycle check at every release, not just at the start.

---

## 3. Analog and Power Simulation

### LTspice
- Free, fast, large library of ADI/Linear parts and models.
- Use for: sensor-to-amplifier chains, filters, op-amp stability, buck converter transients.

### ngspice (built into KiCad)
- Simulate directly from the schematic editor.
- Use when you want to stay inside KiCad.

### TI WEBENCH Power Designer
- Generates regulator designs and loss estimates from your input/output spec.

### PSpice for TI
- Free, uses TI models. Good for TI regulators and amplifiers.

### Analog Devices LTpowerCAD
- Power design and loss analysis for ADI/LT regulators.

### Worked example: current sense chain
1. Model the sensor as a voltage source: sensitivity × current, centred on VREF.
2. Add the amplifier gain stage and filter.
3. Sweep current from 0 to full scale.
4. Check output against the ADC input range.

---

## 4. Power Budgeting

Spreadsheet or script approach (most transparent):

| Column | Meaning |
|---|---|
| Rail | e.g. +5V, +3.3V, VDDA_ISO |
| Component | Reference and part number |
| Typical current | From datasheet typical table |
| Peak current | From datasheet maximum |
| Source | Datasheet page or table reference |
| Duty / condition | Always on, LED on, TX active |

Then sum per rail, compare against the regulator rating and fuse, and compute regulator dissipation `(Vin − Vout) × I` for linear stages.

**Why it helps with disagreements:** each number carries a datasheet reference, so a reviewer can argue with a specific line instead of a total.

**Pitfalls to check**
- LED currents (resistor drop, supply voltage).
- Isolated DC-DC converters: minimum load requirements.
- Inrush and startup versus steady state.
- Typical versus maximum values mixed in the same column.

---

## 5. Layout, DFM and Signal Integrity

| Tool | Use |
|---|---|
| **Saturn PCB Toolkit** (free) | Trace current, impedance, via, clearance calculations |
| **KiCad DRC + custom rules** | Creepage and clearance rules per net class |
| **InteractiveHtmlBom** | Visual BOM for assembly and review |
| **JLCPCB / PCBWay DFM checks** | Manufacturer rule checks before ordering |
| **openEMS** | Open-source EM field simulation (advanced) |
| **KiCad 3D viewer / StepUp** | Mechanical fit and enclosure checks |

**Mains and isolation boards:** define creepage and clearance by net class (mains, isolated analog, logic) and verify against the relevant standard for your voltage and pollution degree.

---

## 6. Firmware and Bring-up

| Tool | Use |
|---|---|
| **STM32CubeMX** | Pin, clock and peripheral conflict checks (STM32 projects) |
| **MPLAB Code Configurator** | Pin and peripheral setup for PIC32 |
| **PlatformIO** | Multi-target builds, unit testing, CI |
| **Ceedling / Unity** | C unit testing framework |
| **Renode** | Emulate MCUs and peripherals before hardware exists |
| **sigrok / PulseView** | Logic analyser software for SPI, I2C, UART and more |
| **Wireshark** | Ethernet, Modbus TCP, and CAN (with adapters) analysis |

**Bring-up checklist**
1. Power rails unloaded, then loaded, one at a time.
2. Reset line level and release.
3. Clock present.
4. Programming and debug connection.
5. One peripheral at a time (SPI, then Ethernet, then CAN).

---

## 7. AI-Assisted Tools

| Tool | Notes |
|---|---|
| **Flux.ai** | Browser-based EDA with AI assistance for part selection and checks |
| **CELUS** | Component selection and block-level design support |
| **Quilter** | AI-driven PCB layout generation |
| General LLMs | Useful for datasheet summaries, checklist drafting, script writing |

**Caution:** AI tools can miss pin-level errors and hallucinate pin functions or values. Always verify against the datasheet. They are best at generating checklists and scripts that *you* then verify.

---

## 8. Per-IC Datasheet Checklist Template

Create one of these for every IC on the board and keep it in the repo.

```markdown
## <Part number> (<Reference>)

- [ ] Supply pins: voltage range, decoupling per datasheet, sequencing
- [ ] Reference/output pins: are any outputs (VREF, LDO outs) tied to anything they should not be?
- [ ] Reset pin: required pull-up/pull-down, reset behaviour at power-up
- [ ] Strap/config pins: sampled at reset, required levels, internal pulls
- [ ] Logic levels: VIH/VIL versus the driving device's VOH/VOL
- [ ] 5 V tolerance: confirm per pin, not per device
- [ ] Unused pins: tie-off recommendation
- [ ] Bus connections: direction (MISO/MOSI), chip selects, shared-bus contention
- [ ] Power-up state of outputs
- [ ] Lifecycle status and alternate part
- [ ] Datasheet revision used: ____
```

---

## 9. Suggested End-to-End Workflow

1. **Capture:** use verified library symbols; set pin types correctly.
2. **Self-review:** strict ERC, netlist script, per-IC checklist.
3. **Simulate:** analog front ends, regulators, filters.
4. **Budget:** script-based power budget with datasheet references.
5. **Source:** lifecycle check and alternates in the BOM.
6. **Automate:** KiBot in CI to re-run steps on every commit.
7. **Layout:** net-class rules, DRC, DFM check.
8. **Peer review:** share the checklist and budget alongside the schematic.
9. **Bring-up:** staged power-on, logic analyser, unit-tested firmware.

---

## 10. Quick Starter Stack

If you only adopt a few tools, start here:

1. KiCad ERC (strict) + netlist script
2. KiBot in GitHub Actions
3. LTspice for analog chains
4. KiCost or Octopart for lifecycle
5. Spreadsheet or Python power budget
6. Per-IC checklist in the repo
7. sigrok/PulseView for bring-up
