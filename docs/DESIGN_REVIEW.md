# ESP32-P4 Camera Board - schematic and netlist review

Review date: 2026-08-30

## Review scope

Reviewed evidence:

- EasyEDA schematic PDF: [`design/ESP32P4_Camera_Board_Schematic_2026-08-30.pdf`](../design/ESP32P4_Camera_Board_Schematic_2026-08-30.pdf)
- EasyEDA TEL netlist: [`design/Netlist_PCB1_2026-08-30.tel`](../design/Netlist_PCB1_2026-08-30.tel)
- [Waveshare ESP32-P4-Module schematic/datasheet](https://files.waveshare.com/wiki/ESP32-P4-Module/ESP32-P4-Module-datasheet.pdf)
- [Espressif ESP32-P4 schematic checklist](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32p4/schematic-checklist-esp32p4.html)
- [Espressif ESP32-P4 PCB layout guidance](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32p4/pcb-layout-design-esp32p4.html)
- Primary component data sheets linked in the relevant findings below.

This is a schematic/connectivity review. No PCB source, Gerbers, stackup, placement, or routing files were supplied, so signal integrity, antenna geometry, thermal copper, footprint orientation, and physical clearances remain unverified.

## Executive result

The architecture is coherent: the P4 module, two-lane CSI and DSI, switchable camera rails, I2C translation, USB-C device connection, IMU, digital microphone, battery charger, and power-source ORing are all recognizable and mostly connected consistently. The MIPI lane polarities and the module's main 3.3 V/VBAT connections agree with the module pinout.

I would not release this revision for fabrication yet. The highest-risk items are the open RF antenna port, an under-margined and thermally stressed 3.3 V supply, USB/charging input-current assumptions, battery-safety limitations, and several pin-domain/development-access issues. These are fixable without changing the overall product concept.

## Stop-before-fabrication findings

### 1. The ESP32-C6 antenna output is electrically unconnected

**Evidence:** Module pin 2 (`LNA_OUT`) is absent from every net in the supplied netlist. The Waveshare module documentation says that this pin is the ESP32-C6 antenna connection and is routed to the module pad by default; a module resistor option can instead select the IPEX connector.

**Impact:** Unless the actual module is reworked for IPEX and an antenna is fitted, Wi-Fi and Bluetooth will have no usable antenna. Range may be extremely poor and the transmitter can see an uncontrolled RF load.

**Required action:** Choose and document one of these paths:

1. Route module pin 2 through a 2.4 GHz matching network and a controlled 50 ohm trace to a qualified PCB/chip antenna, with its required copper keepout.
2. Specify the Waveshare resistor configuration that selects IPEX and fit an approved 2.4 GHz antenna.

Do not leave the choice to assembly interpretation. Add the antenna option and matching components to the BOM and assembly drawing.

### 2. The main 3.3 V rail does not have enough assured power or thermal margin

**Evidence:** U18 is an AP7361C-33E-13, a 1 A SOT-223 LDO. The P4 alone requires at least about 380 mA before adding the C6 radio, camera rails, display, microphone, IMU, status LEDs, and WS2812 LEDs. The [AP7361C data sheet](https://www.diodes.com/datasheet/download/AP7361C.pdf) gives a typical 340 mV dropout at 1 A, approximately 110 degrees C/W junction-to-ambient for SOT-223 under its test condition, and about 1.1 W package dissipation at 25 degrees C.

From USB, U18 sees roughly 4.7 V after D4. Its dissipation is approximately:

- 0.70 W at 0.50 A
- 1.12 W at 0.80 A
- 1.40 W at 1.00 A

The latter two cases are at or beyond the data-sheet package limit at room temperature, before enclosure and ambient-temperature derating. From a LiPo, regulation will be lost near the lower part of the discharge curve; at 1 A and typical dropout, 3.3 V regulation requires roughly 3.64 V at U18's input.

**Impact:** Thermal cycling/shutdown, brownouts during Wi-Fi or camera bursts, reduced battery capacity, and unstable behavior that may look like firmware faults.

**Required action:** Build a worst-case rail budget with measured or specified peak currents for the module, exact IMX708 assembly, exact DSI display, LEDs, and every external load. Replace U18 with a synchronous buck-boost regulator sized with margin (normally at least 2 A output for this feature set), or redesign the power tree so high-current peripheral loads do not pass through this 1 A LDO. A buck-boost is the cleanest way to maintain 3.3 V from both 5 V USB and the full LiPo discharge range.

Also increase C16 at U18 IN. It is only 470 nF, while the AP7361C specifies at least 1 uF close to IN and uses 4.7 uF in its typical circuit. Use a voltage-rated 4.7 uF or larger ceramic directly between U18 IN and ground.

### 3. USB input current is not controlled against simultaneous board load and charging

**Evidence:** R31 = 2 kohm programs the MCP73831 for approximately 500 mA charge current. The board can simultaneously draw several hundred milliamps or more from VBUS through D4/U18. The Type-C CC pins have valid 5.1 kohm Rd resistors, but the circuit has no CC current-advertisement sensing and no input-current limiter or negotiated power controller.

**Impact:** A default-current USB source may be asked for the charger current plus the active system current. The result can be source shutdown, cable/connector droop, repeated resets, or charging that only works on some adapters.

**Required action:** Do not budget 500 mA exclusively for charging unless the available Type-C current has been detected or negotiated and the remaining system current fits. Prefer a charger/power-path IC with input-current regulation and dynamic power-path management. If retaining MCP73831, reduce charge current to a conservative value based on the minimum supported source and battery capacity, and document that the system load has priority. The [MCP73831 data sheet](https://ww1.microchip.com/downloads/en/DeviceDoc/MCP73831-Family-Data-Sheet-DS20001984H.pdf) also shows thermal regulation at the 500 mA/2 kohm setting; expect the SOT-23 charger to reduce current if copper area is inadequate.

### 4. The LiPo subsystem needs an explicit safety contract

**Evidence:** The design has a bare two-pin battery connector, MCP73831T-2DCI/OT, and no cell-protection IC, pack thermistor input, safety timer, reverse-polarity protection, or board-level undervoltage disconnect. Microchip identifies the `-2DC` option as having no deeply-depleted-cell preconditioning in its evaluation documentation.

**Impact:** The circuit is only safe if the connected pack supplies the missing protection functions and its capacity supports the chosen 500 mA rate. An unprotected or reverse-polarity pack is not safely handled. A deeply depleted cell is not given a controlled precharge by this charger option.

**Required action:** State on the schematic and product specification that only a protected, correctly polarized 4.2 V Li-ion/LiPo pack of at least the validated capacity may be used. Better: add reverse-polarity protection, pack-temperature monitoring, undervoltage cutoff, and a charger/power-path device with preconditioning and safety timing. Verify the exact JST cable polarity; two-pin battery harness polarity is not universally standardized.

### 5. Correct and validate the 1.1 V buck-converter component set

**Confirmed issues:**

- R4 = 200 ohm and R5 = 240 ohm produce the intended 1.10 V ratio, but continuously waste about 2.5 mA. These values look like missing `k` suffixes. Change them to an appropriate high-value pair such as 200 kohm/240 kohm after checking feedback leakage and layout.
- The schematic says U9 is 2.2 uH, but the netlist footprint/MPN text is `XAL4030-332MEC`, which is a 3.3 uH part according to [Coilcraft](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xal/xal40xx/xal4030-332/). Resolve the BOM/value conflict.
- L2 is BLM18AG601SN1D, rated 500 mA, while the schematic claims the rail supports 600 mA. The bead is underspecified relative to the stated load and can have substantial DC drop near its limit.

**Validation issue:** The [AP3429 data sheet](https://www.diodes.com/datasheet/download/AP3429.pdf) characterizes its reference circuit with 2.2 uH, 22 uF input, 2 x 22 uF output, and a 22 pF feed-forward capacitor. Your 47 uF output is close in nominal capacitance, but the 3.3 uH BOM part and missing feed-forward-capacitor footprint are deviations that should be validated. The safest revision is to use the characterized 2.2 uH value and reserve the 22 pF footprint across the upper feedback resistor.

### 6. Expansion headers mix voltage domains and expose internally reserved signals

**Evidence:** U13 carries GPIO39-GPIO46. U14 carries GPIO47-GPIO54. On this module, GPIO39-GPIO48 are powered by the `ESP_LDO_VO4` domain, normally treated as a 1.8 V/configurable bank, while GPIO49-GPIO54 are in a different bank. Therefore U14 mixes GPIO47/48 with GPIO49-54 on one connector. The headers expose no matching logic-supply pin, only ground.

GPIO54 is also the ESP32-C6 reset output used by ESP-Hosted; Espressif's [ESP32-P4/C6 SDIO documentation](https://github.com/espressif/esp-hosted-mcu/blob/main/docs/sdio.md) assigns P4 GPIO54 as `Reset Out`. The Waveshare module schematic also appears to share P4 GPIO6 with a C6-side control/handshake signal; treat GPIO6 as reserved until verified against the exact module revision and software configuration.

**Impact:** A 3.3 V peripheral attached to U13 or U14 pins 1-2 can overdrive a 1.8 V/unpowered bank. Using GPIO54 as ordinary expansion I/O can reset the C6 and break Wi-Fi/Bluetooth. External use of a shared GPIO6 can disturb the co-processor link.

**Required action:**

- Label every header pin with its real voltage domain and function.
- Do not mix 1.8 V and 3.3 V GPIO on an unlabeled connector.
- Add the relevant I/O-rail supply/reference pins to the header or add level translators.
- Mark GPIO54 `C6_RESET` and reserve it.
- Verify and reserve GPIO6 if it is connected internally to C6 GPIO2 on the purchased module revision.

## High-priority reliability and development findings

### 7. Add deterministic P4 reset timing and strengthen the boot strap

The module already includes a 10 kohm ESP_EN pull-up, and R16 adds 5.1 kohm externally, but there is no EN-to-ground timing capacitor. Current Espressif guidance recommends a typical 10 kohm/1 uF RC at CHIP_PU and specifically calls out slow battery ramps as a case where a supervisor may be needed. Add approximately 1 uF at module pin 87 close to the pin, then verify reset-button behavior and rise/fall timing. For maximum robustness on battery power, use a 3.0 V-class voltage supervisor.

GPIO35 relies on its internal weak pull-up. Espressif recommends reserving an external pull-up at GPIO35. Add a 10 kohm pull-up while retaining the boot button to ground. Keep capacitance off GPIO35.

### 8. Provide a practical console/JTAG/recovery interface

The USB-C connector uses the dedicated high-speed `USB_DM/USB_DP` pins. This can flash in Joint Download mode, but only in Full-Speed mode and with manual BOOT/RESET entry; it is not the normal USB Serial/JTAG connection. The conventional USB Serial/JTAG pins GPIO24/25 are unavailable as a pair because GPIO24 drives `2V8_EN`. UART0 defaults to GPIO37 TX and GPIO38 RX, but GPIO37 reaches no connector and GPIO38 is loaded by the IO38 LED.

**Recommendation:** At minimum, add labeled test pads for P4 UART0 TX, RX, GND, ESP_EN, GPIO35, and 3.3 V, and move the GPIO38 LED elsewhere. Preferably move `2V8_EN` to another GPIO and provide GPIO24/25 USB Serial/JTAG on a second connector or test pads. Keep the existing high-speed USB port for the application.

### 9. GPIO34 is a strapping pin but is driven by IMU_INT1

The ESP32-P4 data sheet identifies GPIO34 as the early-boot JTAG-source strap and says it has no internal pull resistor. In the default eFuse state its value is ignored, so the current board will normally boot; however, future JTAG/security eFuse choices can make the IMU's power-up output state determine the strap unpredictably.

Move IMU_INT1 to a non-strapping GPIO if possible. Otherwise add the externally defined strap state required by the final eFuse policy and verify that the QMI8658C interrupt output is high impedance throughout the 3 ms strap-sampling interval.

### 10. Camera I2C remains connected while the 1.8 V camera domain is off

The PCA9306 is always enabled from 3.3 V through R15, but VREF1 and the camera-side pull-ups disappear when `1V8_EN` is low. The same 3.3 V I2C bus also serves the QMI8658C. The [PCA9306 data sheet](https://www.nxp.com/docs/en/data-sheet/PCA9306.pdf) provides a controlled-EN application specifically to isolate the two sides.

**Impact:** An unpowered camera side can clamp, load, or back-power the shared bus, potentially preventing IMU communication while the camera is off.

**Recommendation:** Make PCA9306 EN firmware-controlled and keep it low until the 1.8 V rail is valid and the camera reset sequence is ready. A small transistor/open-drain stage may be needed because EN must be biased on the high-voltage side. Alternatively, give the IMU and camera separate P4 I2C controllers. Verify rise time; 10 kohm pull-ups may be too weak for 400 kHz once connector and translator capacitance are included.

### 11. Add local bypass capacitors to the microphone and each addressable LED

The netlist has no dedicated VDD bypass capacitor for the INMP441 and none for LED1-LED3. Add at least 100 nF directly at the microphone supply pins and one 100 nF capacitor directly at each WS2812B-2020 VDD/GND pair. Consider local bulk capacitance near the LED group. Confirm that the exact WS2812B-2020 suffix is rated for 3.3 V operation; several variants are not guaranteed across process and temperature at 3.3 V.

### 12. Resolve MLCC package and effective-capacitance risk

The netlist places 22 uF in 0402, 10 uF in 0402, 4.7 uF at 5 V in 0402, and 47 uF in 0603. These values can lose most of their nominal capacitance under DC bias, and the voltage/dielectric/manufacturer specifications are absent.

For every power capacitor, specify MPN, dielectric, voltage rating, tolerance, and minimum effective capacitance at its operating voltage. Increase package size where necessary. In particular verify C5, C6, C15, C18, C19, C37, and C38 against the regulator/charger stability and transient requirements.

## Interface-specific notes

### USB-C

Good:

- Both CC pins have independent 5.1 kohm Rd resistors.
- Both connector D+/D- contacts are tied by polarity as expected.
- D2/D3 provide line-to-ground ESD protection.
- D4 prevents battery backfeed into VBUS.

Needs work:

- Treat this connector as USB device/download only. There is no 5 V boost, protected VBUS switch, discharge path, or overcurrent sensing for USB host/OTG operation.
- Confirm D2/D3 capacitance and place them immediately at the connector with very short ground return.
- Add test points for VBUS, D+, and D-.
- Verify 90 ohm differential impedance, continuous ground reference, minimal vias, short stubs, and ground return vias at layer transitions.

### MIPI CSI camera

Good:

- Lane polarity and connector mapping are internally consistent in the netlist.
- 24 MHz MCLK is sourced at 1.8 V through 22 ohm series damping.
- XCLR has a 10 kohm/1 uF delayed release.
- Camera rails default off through enable pull-downs.

Needs work:

- Espressif explicitly recommends reserving 0 ohm series resistors on CSI lanes, especially when an RF module is present. DSI has these resistors; CSI does not. Add six 0 ohm footprints close to the camera/device end.
- Verify the exact IMX708 flex/module electrical data and its required rail order, delays, decoupling, inrush, MCLK timing, and FPC pin-one orientation. `IMX708` alone is not enough to qualify the connector.
- Consider an ultra-low-capacitance MIPI ESD array if the FPC is user-accessible.

### MIPI DSI display

Good:

- All six high-speed lines have 0 ohm series footprints.
- The 15-pin connector has multiple grounds between high-speed groups.
- DSI3/DSI4 have 2.2 kohm pull-ups, consistent with low-speed open-drain controls.

Needs work:

- Identify the exact display and confirm the connector pinout and FPC contact orientation.
- The connector has only 3.3 V and no dedicated local bulk capacitor. Validate display inrush/current and add local bulk capacitance if this pin powers more than low-current logic.
- Confirm that the display power demand is included in the redesigned 3.3 V budget.

### IMU

The QMI8658C power and I2C mode straps are consistent with the [QST data sheet](https://www.qstcorp.com/upload/pdf/202210/13-52-27%20QMI8658C%20Datasheet%20Rev%20A%20%281%29.pdf): VDD/VDDIO are 3.3 V, CS is high for I2C, SA0 is low, and SDx/SCx are tied to VDDIO as allowed when the auxiliary interface is unused. Preserve the manufacturer's land pattern and no-copper/mechanical recommendations beneath the MEMS package, and keep it away from board edges, mounting holes, hot regulators, inductors, speakers, and flexing zones.

### Digital microphone

The basic INMP441 connections are plausible: SCK, WS, SD, L/R low, and CHIPEN pulled up. Add the missing local bypass capacitor. Keep the clock away from MIPI and antenna routing, and place the microphone port according to its acoustic gasket/keepout requirements. Verify that the land pattern includes the correct acoustic-port opening and paste-mask treatment.

## PCB requirements that still must be reviewed

Do not infer PCB readiness from a correct netlist. The following need native PCB/Gerber evidence:

- At least a four-layer stackup with an uninterrupted ground reference beneath USB and MIPI.
- MIPI: 100 ohm differential, pair mismatch under 10 mil, inter-pair mismatch under 30 mil, continuous reference, and spacing from switching nodes/RF/high-speed clocks.
- USB: 90 ohm differential with short, symmetric routing and no plane splits.
- RF: controlled 50 ohm feed, antenna keepout on every copper layer, matching network placement, ground-via fence, and enclosure/battery clearance.
- AP3429: compact VIN-cap-switch-inductor-output-cap loop and quiet feedback routing.
- U18 and MCP73831: adequate thermal copper and temperature rise calculation, though the recommended solution is to change the power architecture rather than rely on copper alone.
- Camera regulators and oscillator placed close to the connector with quiet ground returns.
- Connector pin-one, contact-side, insertion direction, courtyard, stiffener, and mating-height checks.
- Module land pattern, paste openings, antenna-side keepout, and Waveshare's one-secondary-reflow limitation.
- Creepage/clearance, via-in-pad decisions, solder-mask slivers, assembly access, fiducials, and test-point coverage.

## Schematic/BOM cleanup

Normalize reference designators before production. The current exported netlist uses names such as `ESP32-P4 + ESP32-C6`, `IMX 708`, `MIC`, `QMI8658C`, `VBUS`, and `CHARGE` as references and assigns the power inductor as U9. Many value fields are `{Value}`. This makes automated BOM/CPL validation and assembly troubleshooting fragile.

Use conventional unique references (`U`, `J`, `L`, `D`, and so on), and give every fitted part an exact MPN or a controlled approved-alternates list. Resolve the XAL4030 value conflict and specify capacitor voltage/dielectric/package explicitly.

Add test points for:

- VBUS, VBAT, switched pre-regulator input, 3V3, 2V8, 1V8, and 1V1
- P4 EN, BOOT, UART0 TX/RX, USB D+/D-
- C6 EN/reset, BOOT, UART TX/RX
- Camera XCLR, MCLK, SCL, and SDA

## Suggested revision order

1. Decide Wi-Fi antenna implementation.
2. Replace/rebudget the main 3.3 V power tree and choose the charger/power-path strategy.
3. Define the supported LiPo pack and safety requirements.
4. Correct the 1.1 V buck BOM/feedback/bead issues.
5. Fix expansion-header voltage-domain and C6-reserved-pin labeling/routing.
6. Add P4 reset RC, GPIO35 pull-up, and a real debug/recovery interface.
7. Control the PCA9306 enable and add missing local decoupling.
8. Add CSI tuning footprints and finish camera/display part qualification.
9. Normalize references and complete the production BOM.
10. Run ERC, then review the native PCB and manufacturer DRC before ordering.

## Bring-up gates for the revised board

1. Power the board from a current-limited bench supply with no battery/camera/display; verify rail ramp and P4 reset timing.
2. Measure peak 3.3 V current and minimum rail voltage during P4 CPU load and C6 Wi-Fi transmit bursts.
3. Validate manual recovery flashing through HS USB and normal logging/debug through the newly added interface.
4. Verify C6 reset/ESP-Hosted operation and confirm external headers cannot disturb GPIO54 or any shared control pin.
5. Enable and measure camera rails in the required sequence; check ripple and decay before connecting the sensor.
6. Scope I2C rise/fall times with camera on and off; verify IMU operation in both states.
7. Test CSI/DSI with PRBS or worst-case video modes where supported; inspect lane eye quality if equipment is available.
8. Measure U18/replacement regulator, charger, diode, inductor, and bead temperatures at worst-case USB and battery conditions.
9. Validate charging across minimum/maximum battery voltage, USB source capability, ambient temperature, and simultaneous system load.
10. Perform RF conducted/OTA checks only after antenna matching and enclosure/battery placement are final.
