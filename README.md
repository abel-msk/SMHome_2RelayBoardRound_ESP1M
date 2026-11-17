# SMHome 2RelayBoardRound ESP-01M — revision 8

**English** · [Русский](README.ru.md)

A round, two-layer Wi-Fi controller for roller shutters with two relay outputs, three wall-button inputs and two window reed-switch inputs. The controller is an **ESP-01M module with ESP8285 and 1 MB Flash**. Firmware is not included.

![Revision 8, component side](images/v8-3d-top.png)

![Revision 8, underside](images/v8-3d-bottom.png)

These images were rendered from the revision 8 PCB in KiCad. Available footprint models are used; some packages are missing or represented by approximate models. The images are not a mechanical clearance check.

## Project files

Open `SMHome_2RelayBoardRound_ESP1M_v7.kicad_pro` in **KiCad 10**. Filenames retain the historical `_v7` suffix; `VERSION`, schematic/PCB title blocks and board revision silkscreen identify the current revision as **8**.

| File | Contents |
|---|---|
| `SMHome_2RelayBoardRound_ESP1M_v7.kicad_sch` | Root sheet: ESP module, relay drivers, indicators and programming connector |
| `PowerSupply.kicad_sch` | Mains protection, power conversion and power terminals |
| `ExtConnectors.kicad_sch` | Button/reed inputs, pull-ups, filters, protection and ADC selector |
| `SMHome_2RelayBoardRound_ESP1M_v7.kicad_pcb` | Two-layer PCB |

`MCU_STM32.kicad_sch` is a legacy file, not a sheet of the current ESP project. Existing Gerbers, plots, BOM exports and older images are historical outputs; regenerate manufacturing files from the current sources.

## How the circuit works

### Power supply

The mains supply enters through **J4**. **F1** protects the power-supply branch; **VR1 07D471K** is connected between the fused live rail and neutral. **DB1 MB8S** rectifies the input. **U2 BP2525F**, **L1 320 µH**, the switching diodes and capacitors form a non-isolated buck supply for **+5 V**. **R14 = 0.3 Ω** is the switching-current sense resistor. **C5 = 680 µF** provides output bulk capacitance.

**U4 AMS1117-3.3** supplies the ESP and input pull-ups from the +5 V rail. Its SOT-223 tab/pin 2 is connected to **+3V3**, not GND. The BP2525F DRAIN copper area is part of the high-voltage switching circuit.

**There is no galvanic isolation from the mains.** GND is connected to the rectifier negative rail. The 3.3 V and 5 V labels describe voltages relative to this GND; they do not identify a touch-safe supply. Do not connect GND to protective earth (PE). The enclosure, external switches, reed sensors, cables and connections must provide insulation suitable for the mains-related circuit. A small voltage across a contact does not make either conductor safe relative to earth.

### Relay outputs

**RL1/RL2 RJ-SS-105DM1** have 5 V coils and independent normally-open power contacts. **Q1/Q2 BC817** switch the coils through base resistors **R3/R7 = 1 kΩ**. **R1/R6 = 10 kΩ** pull the control signals down. **D1/D2** suppress coil turn-off transients; **D4/D5** indicate coil energization through **R4/R8 = 1 kΩ**.

Both contact circuits switch the live rail to separate motor-direction outputs. **The circuit has no hardware direction interlock**: both outputs can be energized together. A shutter installation needs a suitable interlock strategy. Firmware must also enforce mutually exclusive outputs, a motor-appropriate reversal delay and a movement timeout. Motor suitability requires checking the relay's inductive-load/inrush ratings; a resistive-load rating is insufficient.

### Buttons and window contacts

**SW1–SW3** are wall-button inputs; **SN1/SN2** are window reed-switch inputs. Each external dry contact closes its signal to **J2.7 / GND**. No external live voltage may be applied to J2.

All five channels have **4.7 kΩ pull-ups to +3V3** and **100 nF capacitors to GND**: open contact = HIGH, closed contact = LOW. The approximate closed-contact current is **0.7 mA**; switch contacts must be reliable at this small signal current.

| Input | GPIO / U1 pin | Pull-up | Series resistor | Capacitor | ESD diode |
|---|---|---|---|---|---|
| SW1 | GPIO10 / 12 | R17, 4.7 kΩ | R11, 100 Ω | C11, 100 nF | D10 |
| SW2 | GPIO16 / 17 | R18, 4.7 kΩ | R12, 100 Ω | C14, 100 nF | D11 |
| SW3 | GPIO4 / 10 | R16, 4.7 kΩ | None in the digital branch | C17, 100 nF | D9 |
| SN1 | GPIO14 / 15 | R20, 4.7 kΩ | R13, 100 Ω | C15, 100 nF | D12 |
| SN2 | GPIO13 / 5 | R21, 4.7 kΩ | R15, 100 Ω | C16, 100 nF | D13 |

**D9–D13 are PESD3V3L1BA** protection devices. R17/R18/R20/R21 connect on the connector side of the 100 Ω resistors; R16 connects directly to SW3, on the GPIO side of JP3. R26 belongs only to the ADC branch and does not serve as a series resistor for the SW3 button.

For approximately 2 m runs, use one twisted pair per input: **signal + circuit GND**. Cable insulation must suit the mains-related circuit; Ethernet category alone is not an insulation rating. Keep these cables away from motor/mains switching conductors. Add firmware debounce, for example 20–50 ms of stable input. Use plain input mode when external pull-ups are fitted. GPIO16 has no internal pull-up and should be polled rather than used with ESP8266 Arduino `attachInterrupt`.

For a window contact closed when the window is closed, an open window and a broken cable both produce HIGH. This is not a supervised loop and cannot distinguish those conditions.

## Connector pinouts

### J2 — buttons, reed switches and selectable ADC input

| Pin | Function |
|---:|---|
| 1 | SW3 or ADC, selected by JP3 |
| 2 | SW1 |
| 3 | SW2 |
| 4 | SN1 |
| 5 | SN2 |
| 6 | +3V3 |
| 7 | GND; common contact return, not PE |

### JP3 — digital/analog selection

- **1–2 closed:** J2.1 connects to SW3/GPIO4. R16 pulls this input up; the external button closes it to GND.
- **2–3 closed:** J2.1 feeds the ADC through R26. R16 and C17 remain on the disconnected GPIO4 branch.
- Do not bridge both positions at once.

```text
J2.1 ── JP3 2–3 ── R26 3 kΩ ──┬── ADC (U1.16)
                                │
                             R27 1 kΩ
                                │
                               GND
```

For a low-impedance voltage source, **Vadc = Vin / 4**. The divider presents approximately 4 kΩ input resistance: 3.3 V gives 0.825 V at the ADC; 3.6 V gives 0.900 V. With ±5% resistor tolerances, the worst-case ADC voltage at 3.6 V input is approximately 0.969 V. These are calculations, not measurements or simulation results. The ESP ADC range is 0–1 V. This input is not suitable for direct mains measurement or an unconditioned current-transformer output.

### J3 — programming and reset, 3.3 V logic

| Pin | Signal |
|---:|---|
| 1 | GND |
| 2 | +3V3 |
| 3 | ESP RX; connect adapter TX |
| 4 | ESP TX; connect adapter RX |
| 5 | Flash / GPIO0, U1.8 |
| 6 | NRST, U1.18; active LOW |

The former on-board reset and boot buttons are removed. To enter the bootloader, hold GPIO0 LOW while pulsing NRST LOW, then release reset and GPIO0. Use the numbered pinout above; the old nearby `3v3` silkscreen text has not been relocated for the expanded header.

**Disconnect all mains connections before programming** and supply the logic from an isolated 3.3 V source. Do not attach a conventional USB-UART adapter or an earth-referenced oscilloscope while mains power is present.

### J4 — mains and motor outputs

| Pin | Net | Function |
|---:|---|---|
| 1 | GND | Electronics common, not protective earth |
| 2 | N | Mains neutral |
| 3 | L | Mains live and relay contact supply |
| 4 | RL2 | Switched live through relay RL2 |
| 5 | RL1 | Switched live through relay RL1 |

F1 protects only the board supply branch, not motor current through the relay contacts. Provide appropriate upstream motor/wiring protection separately.

### Relay GPIO mapping

| Function | GPIO | U1 pin |
|---|---:|---:|
| RL1 | 12 | 14 |
| RL2 | 5 | 13 |

GPIO2 and GPIO15 are not used for the external contacts. Firmware must use the revision 8 mapping, including GPIO16 for SW2, and a Flash configuration compatible with using GPIO10 on this module.

## Libraries and 3D models

The project uses standard KiCad libraries plus custom libraries referenced by `sym-lib-table` and `fp-lib-table`. These tables currently contain local absolute paths and need adjustment on another machine. Configure the custom symbols/footprints locally and set **`KIKAD_USER_3DMODELS`** (historical spelling) to the custom 3D-model directory. The ESP model also uses a legacy 3D path alias. Embedded symbols and board footprints preserve the current design, but cloning the repository does not install all editing libraries or models.

The PNG renders are included so the board can be viewed on GitHub without those libraries. Older `images/3dview.png` and `images/top.png` are not revision 8 illustrations.

## Revision 8 and verification status

- Updated component population and routing; five external 4.7 kΩ input pull-ups added.
- R26 moved exclusively into the ADC branch; R16 placed on SW3 so it does not bias ADC mode.
- Separate SW1 and selectable SW3/ADC terminals; J3 expanded to six programming/reset pins.
- On-board reset/boot buttons and the separate GPIO9 indicator removed; relay GPIO mapping updated.
- Revision 8 title blocks and board marking, bilingual documentation and new 3D renders.

Checks performed with KiCad 10.0.8 on **2026-11-18**, against the saved project:

| Check | Result |
|---|---|
| Schematic component inventory | 64 components, including connectors and test points |
| DRC after zone refill | 1 error, 60 warnings; 0 unconnected items |
| Schematic/PCB parity | 0 issues |
| ERC | 14 errors, 353 warnings |

The DRC error is an incomplete GND thermal relief at **J2.7 on B.Cu** (one spoke instead of the configured minimum of two). Warnings include 32 footprint-library mismatches, 21 silkscreen/edge clearances and 7 isolated copper areas. ERC includes unconnected pins, the unmatched `AC_SENS` hierarchical sheet pin, a pin-type conflict and power-input declarations; most warnings are off-grid endpoints and symbol-library mismatches. The `JP2` text beside J2.6 is a legacy label; that terminal is +3V3.

**Revision 8 is a saved development revision, not a fabrication-ready or safety-certified release.** Remaining electrical/layout issues, motor interlocking, external insulation and final mechanical fit require resolution/validation before manufacture or mains operation. No hardware measurements, new SPICE simulations or full compliance assessment were performed for this documentation update.
