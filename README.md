# Inertia Wheel Inverted Pendulum

## Table of Contents
- [Project Overview](#project-overview)
- [Hardware](#hardware)
  - [Component List, Rationale & Datasheets](#component-list-rationale--datasheets)
  - [Where We Bought Everything](#where-we-bought-everything)
  - [3D Printing](#3d-printing)
  - [Assembly](#assembly)
- [Modeling & Control Design](#modeling--control-design) (coming soon)
- [Software](#software) (coming soon)
- [component testing](#component-testing)
- [Roadmap / Future Work](#roadmap--future-work)
- [Acknowledgements](#acknowledgements)

---

## Project Overview
Inertia Wheel Inverted Pendulum: System Overview & Control Scope

The Inertia Wheel Inverted Pendulum is an underactuated, non-linear dynamic system designed to demonstrate the real-time stabilization of an inherently unstable process. Unlike traditional cart-pole mechanisms that achieve equilibrium through base displacement, this platform balances vertically by using reaction torque generated at its free end. Accelerating a high-inertia flywheel mounted atop the pendulum rod produces equal and opposite rotational forces that continuously counteract destabilizing gravitational torques. To support physical realization, the platform integrates a complete hardware and software framework covering computer-aided mechanical design, embedded electronics, and low-latency real-time execution.

Mathematically, the platform bridges non-linear physical dynamics with state-space control theory. By linearizing the system's differential equations around the unstable upright equilibrium point, an optimal Linear Quadratic Regulator (LQR) is synthesized within MATLAB. Operating on full state feedback (continuously monitoring pendulum position, angular velocity, and flywheel speed), this real-time feedback loop establishes a baseline for stabilizing complex, underactuated hardware processes against external perturbations.

Building on this baseline of classical PD and LQR control, the project also aims to explore more advanced control strategies, including Model Predictive Control (MPC), sliding mode control, and adaptive methods, time permitting. Where implemented, these controllers would be benchmarked under identical operational conditions to examine trade-offs across dynamic response, disturbance rejection, energy efficiency, and algorithmic robustness.


---

## Hardware

This project builds on the mechanical structure and 3D-printed geometry of [askuric/inverted_inertia_pendulum](https://github.com/askuric/inverted_inertia_pendulum), which we kept unchanged, but departs from it substantially elsewhere. Several years have passed since the original project was published, and a number of the originally suggested electronic components (in particular the salvaged Mabuchi RS-385PH motor with factory-fitted encoder, and the Monster Moto Shield driver) are no longer realistically available to us; nearly all electronic components were re-selected and updated as a result. The project also implements and compares several different controllers (see [Modeling & Control Design](#modeling--control-design)), going well beyond the original's scope. This section documents every hardware component we used, why we chose it, and where to find more information or buy it. Photos of the physical parts we bought are collected in [`/images/hardware/`](./images/hardware/); a condensed purchase list with links and prices is kept in [`/docs/BOM.md`](./docs/BOM.md).

### Component List, Rationale & Datasheets

#### Microcontroller: STM32F411CEU6 ("Black Pill" board, WeAct Studio)
The microcontroller is the central processing unit of the system: it reads sensor data, executes the control algorithm, and commands the motor driver.
- Core: ARM Cortex-M4 with hardware FPU, up to 100 MHz
- Memory: 512 KB Flash, 128 KB RAM
- Hardware timers with quadrature encoder decoding mode, PWM output, and I²C peripheral, everything the control loop needs
- Datasheet: [STM32F411xC/E, STMicroelectronics](https://www.st.com/resource/en/datasheet/stm32f411ce.pdf)
- Product page: [st.com](https://www.st.com/en/microcontrollers-microprocessors/stm32f411ce.html)
- Board reference: [WeAct Black Pill V2.0, STM32-base project](https://stm32-base.org/boards/STM32F411CEU6-WeAct-Black-Pill-V2.0.html)

We replaced the original project's official Nucleo-F411RE board with a much cheaper "Black Pill" development board built around the same MCU. It exposes the same core peripherals needed for this project. The one practical difference is that the Black Pill has no onboard ST-Link debug probe; firmware is instead flashed through the chip's built-in USB DFU bootloader over the onboard USB Type-C port, using STM32CubeProgrammer instead of a one-click Keil download. Live SWD debugging (breakpoints) is not available this way; an external ST-Link can be added later if needed.

> Note: the Black Pill's 25 MHz crystal cannot be divided to the precise 48 MHz required by the USB peripheral while the core runs at its rated maximum of 100 MHz; with USB (and therefore the DFU bootloader) enabled, the achievable core clock is capped at 96 MHz. This project runs at 96 MHz for that reason.

> Caution: on this board, the +5V header pins are wired directly to the USB connector's VBUS with no protection diode. Do not power the board from an external supply and USB at the same time.

#### Motor Driver: BTS7960 (43 A dual half-bridge module)

- Datasheet: [BTS7960, Infineon Technologies](https://www.infineon.com/assets/row/public/documents/10/57/infineon-bts7960-ds-en.pdf)
- Supply voltage range: 5.5 to 27.5 V, up to 43 A continuous
- Built-in protection: overcurrent, overtemperature, short-circuit, undervoltage lockout

The original project used a Monster Moto Shield (VNH2SP30-based), an Arduino-shield-form-factor product that is largely discontinued and hard to source today. BTS7960 modules are a widely available, inexpensive, well-documented alternative that comfortably exceeds this project's current requirements, giving a generous safety margin over the motor's actual draw.

#### Motor: Mabuchi RS-385, 12 V, dual-shaft, gearless

"RS-385" is a standard small DC motor frame size, produced by Mabuchi (among other manufacturers) in several variants. We specifically required:
- Dual shaft, because the design uses both ends: the front shaft drives the inertia wheel, the rear (free) shaft carries the speed-feedback sensor
- No gearbox, because gear backlash introduces nonlinear, hard-to-model behavior that is undesirable for precise control, and because the reduced output speed of a gearmotor is unsuitable for a reaction wheel that needs to spin up and down quickly

The original repository used a salvaged Mabuchi RS-385PH pulled from office equipment (printers/copiers), which came with a factory-fitted encoder; these units are essentially unobtainable today. We sourced a new (not salvaged) Mabuchi RS-385 motor without the factory encoder, and paired it with a separately purchased AS5600 sensor for speed feedback (see below).

- Voltage: 12 V
- Shaft diameter: 2.3 mm, dual-shaft, gearless
- Codes printed on the motor housing: `6007-350-0018`, `385-17150-01`, `1H7G`

No public datasheet could be located for this exact part number; the codes above appear to be an internal packager/distributor part number rather than a standard Mabuchi catalog designation, so the figures listed here are limited to what is printed on the unit itself. For general reference on this motor family's typical electrical characteristics, see the [Mabuchi RS-385PH datasheet](https://datasheet.octopart.com/RS385PH-10280-Mabuchi-datasheet-13546593.pdf). Note that this is a different, encoder-equipped variant, so its figures should not be taken as exact for our unit.

#### Motor Speed Sensor: AS5600 magnetic rotary position sensor

- Datasheet: [AS5600, ams AG](https://files.seeedstudio.com/wiki/Grove-12-bit-Magnetic-Rotary-Position-Sensor-AS5600/res/Magnetic%20Rotary%20Position%20Sensor%20AS5600%20Datasheet.pdf)
- Product page: [ams-osram.com](https://ams-osram.com/products/sensor-solutions/position-sensors/ams-as5600-position-sensor)
- 12-bit resolution (4096 counts/revolution), contactless Hall-effect sensing, I²C output

The original project measured motor speed with a hand-built sensor: two discrete digital Hall sensors plus a small multi-pole magnetic disc salvaged from a Pololu Romi encoder kit, requiring careful manual alignment. We replaced this with a single AS5600 module (magnet included), mounted opposite a small magnet glued to the motor's rear shaft. This is mechanically much simpler to install and gives substantially higher resolution than the original 12 to 20 CPR approach.

Because a generic gearless RS-385 motor does not have the small internal-spline gear that the original inertia wheel STL's hub is shaped for, we designed and 3D-printed a small custom coupler that mates the wheel's original 12-tooth internal spline (measured directly from the STL geometry) to a plain round motor shaft. See [`/CAD/STL/motor_shaft_to_wheel_adapter_12T.stl`](./CAD/STL/motor_shaft_to_wheel_adapter_12T.stl).

#### Pendulum Angle Sensor: two-phase optical rotary encoder, 600 CPR (labeled E6A2-38F600-12C-2M)
This sensor is the primary feedback signal of the whole system: it measures the pendulum arm's angle relative to vertical, the variable the controller is trying to stabilize.

- Source: [Partineh](https://partineh.com/product/%d8%b1%d9%88%d8%aa%d8%a7%d8%b1%db%8c-%d8%a7%d9%86%da%a9%d9%88%d8%af%d8%b1-%d9%86%d9%88%d8%b1%db%8c-%d8%af%d9%88-%d9%81%d8%a7%d8%b2-600-%d8%af%d8%b1%d8%ac%d9%87-%d9%85%d8%af%d9%84-hn3806) (6–24 VDC, 38 mm body diameter, 6 mm shaft, 600 pulses/rev, up to 5000 RPM)

The part is stamped `E6A2-38F600-12C-2M`, styled after Omron's E6A2 series naming, but its specifications (38 mm body, 600 PPR) do not match any genuine Omron E6A2 model (which tops out at 25 mm body diameter and 500 PPR). It appears to be an unofficial part following that naming convention rather than a genuine Omron part, and no datasheet matching its actual specifications could be located; the figures above come from the seller's listing.

We chose a digital two-phase (quadrature) optical encoder over a potentiometer for several reasons:

- **Precision for the controller.** The higher the precision of the pendulum angle measurement, the better a controller, especially LQR, can manage the unstable equilibrium. This need for precision is what ruled out a potentiometer in the first place.
- **Prior experience with potentiometers.** From earlier work with precision potentiometers, we knew that because the pendulum's rotation is not confined to a small range (it can exceed 360°) while the highest precision is only needed near the small angular range around vertical, a multi-turn (10-turn) potentiometer only exercises a small portion of its resistive track within that critical range. This makes the signal noise-sensitive, particularly with the Chinese-made unit we would have used; German-made precision potentiometers reportedly avoid this issue, but their price was significantly higher than we could justify (this was not a low-cost part in absolute terms, just cheaper than the German alternative).
- **Noise immunity.** A digital two-phase output is inherently more noise-resistant than an analog potentiometer signal.
- **Direction sensing.** The AB output (two phases, 90° apart) reports rotation direction in addition to magnitude, something a simple potentiometer does not provide by default.
- **Effective resolution above the nominal rating.** Counting both rising and falling edges on both phases yields up to 4x the nominal resolution, so even a 360 CPR version would effectively give around 1440 counts/revolution. This is why lower-CPR options were also viable candidates. We chose the 600 CPR version as the best, most precise option available (2400 effective counts/revolution versus 1440 for the 360 CPR version).
- **Physical compatibility.** The body diameter and shaft size of this encoder (38 mm body, 6 mm shaft) match the original project's `encoder_holder.STL` and `axel_adapter_10mm_encoder.STL` print files directly, requiring no redesign.

> Note: Pull-up resistors (1 to 10 kΩ) are required on both phase lines to VCC. Without them the encoder output is unusable and can potentially damage the sensor's output stage.

#### Power Supply: 18 V, 3 A switching adapter (Lexus IR-54)
Feeds the motor side of the BTS7960 driver only (logic-side components run off the MCU board's own 5 V/3.3 V rails). Input: AC 100–240 V, 50/60 Hz; output: 18 V, 3 A. Since the motor's rated voltage (12 V) is lower than the supply, the maximum PWM duty cycle is capped in firmware (around 65 to 70%) so the motor's effective voltage never exceeds its rating. 3 A gives comfortable headroom over this small motor's actual draw.

#### Mechanical Hardware

| Part | Spec |
|---|---|
| Pendulum arm rod | Brass, ⌀6 mm × 120 mm |
| Pivot bolt + nuts | M10×1.5 bolt (1×), M10×1.5 nuts (2×) |
| Fasteners | M3 socket-head screws, assorted lengths (8 to 16 mm) |
| Heat-set threaded inserts | Brass, M3 |
| Pivot bearings | 6300-series ball bearings (10 × 35 × 11 mm), 2× |
| Base plate | ~90 × 300 mm sheet, 2 mm thick |

These were kept as specified in the original project, since they are standard, widely available parts.

### Where We Bought Everything

Most components were purchased in person rather than online, because no single supplier reliably stocked more than one or two of the parts at a time; each component was typically found at a different store. The only component ordered online was the power adapter: *(insert link here)*.

Components that were genuinely hard to find: the Monster Moto Shield (discontinued, replaced with BTS7960), a factory-encoder-equipped RS-385PH motor (essentially unobtainable, replaced with a bare motor plus AS5600), and consistent stock of the pendulum's optical rotary encoder across suppliers.

### 3D Printing

All parts were printed in PLA. Printer used: *(insert printer/slicer details)*. Minimum layer height available to us was 0.2 mm; the original repository recommends 0.1 mm specifically for the inertia wheel for best surface finish, which we adjusted for accordingly (see notes below).

All files below live under [`/CAD/STL/`](./CAD/STL/). Slicer screenshots documenting these settings are in [`/images/printing/`](./images/printing/).

| # | File | Layer height | Infill |
|---|---|---|---|
| 1 | `mabuchi_wheel_12T.STL` | 0.2 mm | 50%+ |
| 2 | `motor_mout_mabuchi.STL` | 0.2 mm | 30% |
| 3 | `bearing_holder.STL` | 0.2 mm | 30% |
| 4 | `mount_table_sharft_holder.STL` | 0.2 mm | 30% |
| 5 | `table_mount.STL` | 0.2 mm | 30% |
| 6 | `nucleo_holder.STL` | 0.2 mm | 30% |
| 7 | `encoder_holder.STL` | 0.2 mm | 30% |
| 8 | `axel_adapter_10mm_encoder.STL` | 0.2 mm | 100%* |
| 9 | `motor_shaft_to_wheel_adapter_12T.stl` (custom, this repo) | 0.2 mm | 100% |

Original Solidworks/STEP source files are kept in [`/CAD/source/`](./CAD/source/).

\* Small parts with thin cross-sections are automatically printed fully solid by the slicer regardless of the configured infill percentage, since there is no room for a distinct infill pattern. This is expected behavior and does not indicate a problem; it only increases strength.

Only the `mabuchi_wheel_12T.STL` variant is used (not the 14-tooth version), since the custom motor-shaft adapter was designed specifically to match its internal spline geometry.

### Assembly

*(Step-by-step assembly instructions to be written here: pivot/base assembly, pendulum arm, motor + wheel + encoder at the free end, pendulum-angle encoder at the pivot, electronics mounting, wiring. Reference photos go in [`/images/assembly/`](./images/assembly/).)*

## component testing

1-STM32F411CEU6 Board Test

To verify that the STM32F411CEU6 development board was functioning correctly, we performed a basic GPIO test by turning the onboard LED on and off.
The test code can be found here:
[Inertia Wheel Inverted Pendulum – GitHub](https://github.com/Dorsa-amiri/Inertia-Wheel-Inverted-Pendulum)

2-driver

---

## Modeling & Control Design
*Coming soon: dynamic model, simulation, and initial controller gain computation (PD, LQR).*

## Software
*Coming soon: firmware, hardware validation/test routines, controller implementation.*

## Roadmap / Future Work

- [ ] Control the system using image processing as an alternative sensing method
- [ ] Graphical interface with buttons to select and run a given controller
- [ ] A dedicated test/diagnostic mode: automatically return the system to its home position before running, verify each subsystem responds correctly to known commands, and gracefully handle faults (e.g. out-of-range encoder readings, a motor spinning without responding to commands, a lost communication link) instead of producing undefined/garbage output
- [ ] Compare simulation results against real system results for each controller, and compare all controllers against each other

## Acknowledgements

Based on [askuric/inverted_inertia_pendulum](https://github.com/askuric/inverted_inertia_pendulum). Mechanical design and 3D-printed geometry are used largely unmodified; electronic components were updated where the originals were no longer available.
