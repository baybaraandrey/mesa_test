# LinuxCNC — X/Y Axis Configuration and Commissioning Reference

Machine: Burny Phantom ST retrofit  
Controller: Mesa 7i97T  
Software: LinuxCNC 2.9.4  
Status: COMMISSIONING — NOT PRODUCTION READY

**Important:** These are reference values and commissioning tasks, not commands to change the working configuration. Actual values must be verified independently for X and Y.

## 1. Axis and encoder parameters

| Parameter | Description | Current / possible values |
|---|---|---|
| `ENCODER_SCALE` | Encoder quadrature counts per mm | X: -1688 provisional; Y: -1688 temporary |
| `MAX_VELOCITY` | Maximum joint speed | Current: 5 mm/s; final TBD |
| `MAX_ACCELERATION` | Maximum joint acceleration | Current: 20 mm/s²; final TBD |
| `MIN_LIMIT` | Minimum permitted joint position | Determined from physical travel |
| `MAX_LIMIT` | Maximum permitted joint position | Determined from physical travel |
| `FERROR` | Maximum allowable following error | Positive mm value; tune after testing |
| `MIN_FERROR` | Following-error tolerance near zero speed | Positive mm value; tune after testing |
| `HOME_OFFSET` | Coordinate assigned to the home-switch reference | mm; determined during homing calibration |

**Calibration:** Measure actual travel with an independent measuring instrument. Verify encoder counts, polarity, backlash, repeatability, and positive axis direction.

## 2. PID servo tuning

| Parameter | Description | Possible values |
|---|---|---|
| `P` | Proportional gain; responds to position error | Current: 1; tune experimentally |
| `I` | Integral gain; compensates persistent error | Start at 0 |
| `D` | Derivative gain; provides damping | Start at 0 |
| `FF0` | Position feedforward | Start at 0 |
| `FF1` | Velocity feedforward | Start at 0; tune as needed |
| `FF2` | Acceleration feedforward | Start at 0 |
| `BIAS` | Constant PID output offset | Normally 0 |
| `DEADBAND` | Error region ignored by PID | Normally 0 initially |
| `MAX_OUTPUT` | Limits PID output magnitude | Current: 1; depends on output scaling |

**Tuning:** Use HAL Scope to record commanded position, actual position, following error, velocity, and PID output. Tune one mechanically safe axis at a time.

## 3. Mesa analog output and drive control

| Parameter | Description | Possible values |
|---|---|---|
| `pwmgen.NN.scale` | Converts HAL PWM command into analog output | Match existing X configuration |
| `pwmgen.NN.offset-mode` | Bipolar PWM mode required for ±10 V | Enabled |
| `pwmgen.pwm_frequency` | PWM frequency used internally | Mesa recommends 75 kHz with dither for LinuxCNC 2.9.2+ |
| `pwmgen.NN.enable` | Enables PWM output | 0 / 1 |
| `ENA0 / ENA1` | Physical servo drive-enable switches | OFF / ON; validate electrically |
| Analog polarity | Sign of voltage relative to commanded direction | Positive / negative |

**Verification:** Measure AOUT0/AOUT1 relative to their adjacent GND terminals. Confirm zero voltage at zero command and verify polarity with the drives disconnected.

Do not assume `MAX_OUTPUT=1` means ±1 V. The analog voltage depends on PWM scaling.

## 4. Limits and homing

| Parameter | Description | Possible values |
|---|---|---|
| Negative limit | Stops prohibited movement toward minimum travel | X: IN0; Y: IN2 |
| Positive limit | Stops prohibited movement toward maximum travel | X: IN1; Y: IN3 |
| Home input | Switch used to establish reference position | Negative switch, as configured |
| `HOME` | Final position after homing | mm; choose from machine geometry |
| `HOME_SEARCH_VEL` | Speed used to find home switch | Signed mm/s; initially conservative |
| `HOME_LATCH_VEL` | Speed used to refine home reference | Signed mm/s; initially conservative |
| `HOME_FINAL_VEL` | Speed moving to final home position | Nonnegative mm/s |
| `HOME_SEQUENCE` | Automatic Home All ordering | Unset until Y is commissioned |
| `HOME_IGNORE_LIMITS` | Allows shared limit/home switch during homing | 0 / 1; review safety consequences |

**Verification:** Check each limit state in HAL Show. NC wiring should produce a limit condition when the switch activates or a wire opens.

Do not perform automatic Y homing until movement direction, switches, and protective functions have been tested.

## 5. Machine geometry and motion

| Parameter | Description | Possible values |
|---|---|---|
| X travel | Physical cutting-table X travel | Measure in mm |
| Y travel | Physical cutting-table Y travel | Measure in mm |
| X/Y squareness | Perpendicularity between motion axes | Measure diagonal difference |
| Backlash | Lost motion during direction reversal | Measure in mm |
| Repeatability | Variation on repeated positioning | Measure in mm |
| Maximum cutting feed | Allowable machine speed during cutting | Determine from machine and process |
| Acceleration limits | Stable acceleration without excessive following error | Test progressively |

## 6. CMC drive and electrical safety

- Confirm CMC `J1-11 INHIBIT/RESET` electrical polarity and behavior.
- Confirm CMC `J1-14 FAULT OUTPUT` electrical type and active state.
- Verify Mesa `ENA0/ENA1` switching, polarity, and safe startup/shutdown state.
- Confirm CMC drive COMMON/reference and shield termination.
- Verify actual DC bus supply voltage and protections.
- Verify independent hardware E-stop and overtravel protection.
- Check LinuxCNC realtime latency warnings and Ethernet reliability.
- Never treat software limit inputs alone as a safety-rated protective circuit.

**Mesa note:** ENA0–ENA3 are controlled as a common group on the 7i97T. Do not assume ENA0 and ENA1 provide independent per-axis hardware enable control.

## 7. Future gantry configuration

- Confirm whether an additional XX servo motor exists.
- Determine which physical direction corresponds to each motor.
- Map its dedicated encoder, analog command, and limits.
- Establish separate gantry-side home references if required.
- Configure synchronized gantry motion and safe squaring.
- Do not add XX until physical axis identification is complete.

## 8. Final commissioning checklist

- [ ] Verify and calibrate X encoder scale
- [ ] Verify and calibrate Y encoder scale
- [ ] Test positive and negative movement direction
- [ ] Test all four limit switches
- [ ] Verify drive-enable electrical behavior
- [ ] Integrate drive fault handling
- [ ] Validate independent hardware E-stop
- [ ] Tune X PID using HAL Scope
- [ ] Tune Y PID using HAL Scope
- [ ] Measure X/Y travel and configure soft limits
- [ ] Configure automatic homing
- [ ] Measure backlash and repeatability
- [ ] Determine safe maximum velocity and acceleration
- [ ] Resolve realtime latency warnings
- [ ] Verify complete dry-run motion
- [ ] Backup verified INI/HAL files

**Rule:** Keep X and Y values separate. Never assume a parameter calibrated for one axis is automatically correct for the other.