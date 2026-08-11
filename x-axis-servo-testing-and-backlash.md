# X-axis servo testing, tuning symptoms, and backlash

Date: 2026-08-11

## Scope and safety

This document is for a future, explicitly authorized X-axis motion-commissioning phase.
It does not authorize connecting CMC J1-11, enabling the CMC drive, moving the motor, or
changing the current ENA0-only bench test.

The machine is a heavy industrial plasma/oxy-fuel gantry. Wrong analog-command polarity
or wrong encoder polarity can create positive feedback and immediate servo runaway.

Before any powered motion test:

1. Complete the Mesa ENA0 bench test with the CMC drive disconnected.
2. Determine the electrical behavior of CMC J1-11 INHIBIT/RESET and J1-14 FAULT.
3. Provide a hardware method of removing drive torque independently of LinuxCNC.
4. Disable all plasma equipment and keep E-stop immediately reachable.
5. Commission only one drive; physically prevent other axes from producing torque.
6. Confirm whether X and XX are the two sides of one gantry before allowing either side
   to move independently.
7. Verify production encoder scale and polarity before closing the position loop.
8. Retain the motor tachometer on CMC J1-6/J1-7.

Immediate runaway or continuously increasing following error is not a normal tuning
symptom. It usually means incorrect command polarity, feedback polarity, scale, or failed
feedback. Press E-stop and correct the cause; do not try to fix it by randomly changing
gains.

## Current X-axis configuration

The present one-joint configuration maps X to joint 0:

| Function | LinuxCNC/HAL name |
|---|---|
| X joint | `joint.0` |
| PID | `pid.x` |
| Encoder feedback | `hm2_7i97.0.encoder.00.position` |
| Analog command | `hm2_7i97.0.pwmgen.00.value` |
| Analog/ENA enable | `hm2_7i97.0.pwmgen.00.enable` |

Values observed in `burny_7i97t_test.ini` on 2026-08-11:

```ini
DEADBAND = 0
P = 10
I = 0
D = 0
FF0 = 0
FF1 = 0
FF2 = 0
BIAS = 0
MAX_OUTPUT = 5
ANALOG_SCALE_MAX = 10
ENCODER_SCALE = 1000
FERROR = 100
MIN_FERROR = 50
```

Earlier project notes mention temporary tests near `P=100` and `FF1=1`. That historical
record does not match the current INI. Neither set is a proven production tune. Inspect
the source and loaded runtime values before every test.

`P`, `I`, `D`, `FF0`, `FF1`, `FF2`, `BIAS`, `DEADBAND`, and `MAX_OUTPUT` are custom INI
entries. They affect `pid.x` because `main.hal` explicitly applies them with `setp`.
The current `MAX_ERROR` entry is not connected to `pid.x.maxerror` in `main.hal` and must
not be assumed effective.

## Runtime inspection

With LinuxCNC running, inspect the loaded gains without changing them:

```bash
halcmd getp pid.x.Pgain
halcmd getp pid.x.Igain
halcmd getp pid.x.Dgain
halcmd getp pid.x.FF0
halcmd getp pid.x.FF1
halcmd getp pid.x.FF2
halcmd getp pid.x.bias
halcmd getp pid.x.deadband
halcmd getp pid.x.maxoutput
```

Useful signals for HAL Scope or HAL Meter:

```text
pid.x.command
pid.x.feedback
pid.x.error
pid.x.output
pid.x.saturated
joint.0.f-error
joint.0.f-error-lim
hm2_7i97.0.pwmgen.00.value
hm2_7i97.0.pwmgen.00.enable
```

Useful net checks:

```bash
halcmd show net joint-0-pos-cmd
halcmd show net joint-0-pos-fb
halcmd show net joint.0.output
```

## What the current P-only settings do

With all other terms at zero, the current loop is approximately:

```text
output = limit(P × (command − feedback), ±MAX_OUTPUT)
```

For `P=10` and `MAX_OUTPUT=5`:

| Position error | Approximate PID output |
|---:|---:|
| 0.001 mm | 0.01 output unit |
| 0.100 mm | 1.00 output unit |
| 0.500 mm | limited to 5.00 output units |

With `ANALOG_SCALE_MAX=10`, the intended convention is approximately one output unit per
volt of analog command. Confirm this with a meter before enabling the CMC drive. `P=10`
is not automatically safe: the result depends on encoder scaling, CMC velocity gain,
tachometer-loop behavior, mechanics, and voltage-to-axis-speed scaling.

## Initial commissioning sequence

Before closing the loop, with the drive unable to produce torque:

1. Manually move the mechanism and confirm that positive physical X movement produces
   the encoder sign expected by LinuxCNC.
2. Measure AOUT0 relative to GND0 and verify zero, positive, and negative command polarity
   using a separately reviewed, low-output test method.
3. Do not alter the CMC drive's original inner velocity-loop adjustments without its
   documentation and a recorded reason.

For the first authorized closed-loop test:

- Use very low velocity and acceleration.
- Use a deliberately small, measured `MAX_OUTPUT` appropriate to the proven test speed.
- Start with P only: `I=0`, `D=0`, `FF0=0`, `FF1=0`, `FF2=0`, and `BIAS=0`.
- Start with low P and increase it only in small, recorded steps.
- Make short moves near the middle of travel.
- Monitor following error, PID output, and saturation.
- Stop for wrong direction, increasing error, violent movement, or abnormal noise.

`FERROR` and `MIN_FERROR` are safety-related machine-unit limits. Do not make them very
large merely to hide a tuning problem. The present 100 mm and 50 mm values are
placeholders, not suitable final protection values. A tuning limit must avoid nuisance
trips while remaining small enough to stop a bad loop before dangerous travel.

## Parameter symptoms and adjustment guidance

The exact PID calculation is documented in the
[LinuxCNC PID manual](https://linuxcnc.org/docs/stable/html/man/man9/pid.9.html).

### P — proportional position-error gain

P produces output proportional to commanded position minus encoder feedback.

- Too low: soft response, poor disturbance rejection, and following error that grows
  with speed.
- Too high: rapid oscillation, buzzing, harsh reversals, or instability.
- Method: start low and use short, slow moves. Increase gradually until response is firm
  but stable, retaining margin below the onset of oscillation.

### I — integral error gain

I accumulates persistent error.

- Too little: a repeatable load-dependent steady error may remain.
- Too much: slow oscillation, overshoot, long recovery, or windup behavior.
- Method: leave at zero during P and FF1 tuning. Add only if a measured steady error
  remains after drive balance and feedforward are correct.

### D — derivative error gain

D reacts to changing error and can add damping, but it also amplifies encoder noise.

- Too much: buzzing, noisy output, spikes, or rough motion.
- Method: leave at zero initially. Add only when scope evidence shows a need.

### FF0 — commanded-position feedforward

FF0 makes output depend on absolute commanded position. It should normally remain zero
for this outer position-loop architecture. It is not a drive-balance adjustment.

### FF1 — commanded-velocity feedforward

FF1 supplies output proportional to commanded velocity and is important when the CMC is
used as an inner velocity-loop drive.

- Too low: following error has the same sign as motion and increases with speed because P
  needs position error to create the velocity command.
- Too high: following error changes sign during steady motion, with possible reversal
  overshoot.
- Method: first establish a stable P-only loop. Then adjust FF1 to minimize
  steady-velocity following error in both directions.

If a safe test establishes that `V` volts produces `S` mm/s, an initial estimate is:

```text
FF1 = V / S   volts per (mm/s)
```

Do not assume `FF1=1` unless measured analog and machine-speed scaling support it.

### FF2 — commanded-acceleration feedforward

FF2 can compensate acceleration-related error.

- Too little: acceleration-related following error may remain after P and FF1 are tuned.
- Too much: harsh starts/stops, output spikes, and overshoot.
- Method: leave at zero initially and consider it only after P and FF1 are stable.

### BIAS — constant enabled-loop output

BIAS should normally remain zero. First correct CMC balance, Mesa analog zero, and
wiring/reference errors. Use BIAS only for a measured and repeatable requirement; it is
never a fix for incorrect polarity.

### DEADBAND — ignored position-error band

- Too small or zero: hunting, chatter, or squealing between encoder counts.
- Too large: visible looseness and reduced positioning accuracy near the target.
- Method: tune the gross loop first, then use roughly one or two verified encoder counts
  if count quantization causes hunting.

Calculate one encoder count in machine units as:

```text
one count = 1 / abs(ENCODER_SCALE)
```

At a verified `ENCODER_SCALE=1000` counts/mm, one count is 0.001 mm. Recalculate after
the production encoder and drivetrain scaling are established.

### MAX_OUTPUT — PID/analog-command limit

- Too high for initial testing: a small error can demand dangerous speed.
- Too low: `pid.x.saturated` remains true and following error grows because commanded
  speed or acceleration cannot be achieved.
- Method: start with a small measured commissioning limit and increase only after the
  voltage-to-speed relationship is known.

`MAX_OUTPUT` limits command voltage. It is not a motor-current or torque limit and does
not replace the CMC current limit or an independent hardware safety circuit.

## Backlash measurement and configuration

Backlash compensation is not servo tuning. Establish a stable servo loop before
measuring mechanical lost motion at the load.

A motor-mounted encoder can show motor movement while clearance in a coupling, gearbox,
pinion, or rack prevents gantry movement. Software compensation does not remove this
mechanical looseness or improve stiffness. Repair loose hardware wherever practical.

Do not configure X-axis backlash until the physical X/XX gantry relationship is
confirmed. Applying correction to only one side of a dual-drive gantry could rack it.

### When `BACKLASH` is appropriate

Use the simple `[JOINT_0]BACKLASH` setting only when:

- X is confirmed as joint 0.
- Position feedback is measured before the mechanical lost motion, such as at the motor.
- Reversal loss is repeatable and approximately constant over X travel.
- No `COMP_FILE` is configured for joint 0.

LinuxCNC ignores `BACKLASH` if `COMP_FILE` is specified. Do not configure both expecting
their corrections to be added. See the
[LinuxCNC INI documentation](https://linuxcnc.org/docs/html/config/ini-config.html).

### Measuring backlash

1. Inspect and tighten the drivetrain.
2. Tune stable low-speed response with backlash absent or set to zero.
3. Place a preloaded dial indicator parallel to X against the carriage or gantry—not the
   motor shaft or encoder.
4. Use very low velocity and acceleration near the middle of travel.
5. Approach a reference point in one direction and zero the indicator.
6. Reverse in small commanded increments. Record the accumulated commanded motion before
   the indicator consistently begins moving in reverse.
7. Repeat in both directions and at several X positions.
8. Separate repeatable lost motion from following error, indicator compliance,
   structural flex, and measurement noise.
9. Use a positive representative value in millimetres only when measurements are stable.

Do not calculate backlash from encoder movement alone when the encoder is on the motor
side of the mechanical clearance.

### Working LinuxCNC configuration

This setup uses millimetres and maps X to joint 0. Add `BACKLASH` inside the existing
`[JOINT_0]` section of `burny_7i97t_test.ini`:

```ini
[JOINT_0]
# existing joint settings...
BACKLASH = 0.000
```

Replace `0.000` with the measured positive lost-motion magnitude in millimetres. For
example only, if repeated measurement establishes 0.20 mm:

```ini
BACKLASH = 0.20
```

`BACKLASH` is a native LinuxCNC motion setting; it needs no `setp` line in `main.hal`.
The current configuration has no `COMP_FILE`, so this setting will be active when added.
Restart LinuxCNC after editing the INI.

Verify the loaded value and observe compensation with:

```bash
halcmd getp ini.0.backlash
halcmd show pin joint.0.backlash-corr
halcmd show pin joint.0.backlash-filt
halcmd show pin joint.0.backlash-vel
```

The `joint.0.backlash-*` pins are debugging pins and can vary by LinuxCNC version. The
authoritative configured value is `[JOINT_0]BACKLASH`, exposed as `ini.0.backlash`.

### Validating backlash compensation

1. Repeat the low-speed dial-indicator reversal test.
2. Test both directions and several X positions.
3. Increase only by a small measured amount if repeatable lost motion remains.
4. Reduce the value if the indicator overshoots or the axis jerks on reversal.
5. Check following error and PID output around reversals in HAL Scope.
6. Do not increase acceleration merely to make compensation act faster.
7. Record the method, measurement locations, temperature, and accepted value.

Keep `BACKLASH=0` until the real X mechanics are connected and measured. A guessed value
creates deliberate motor-position correction at every reversal and can make a correctly
operating axis worse.

## Official references

- [LinuxCNC PID component](https://linuxcnc.org/docs/stable/html/man/man9/pid.9.html)
- [LinuxCNC INI configuration](https://linuxcnc.org/docs/html/config/ini-config.html)
- [LinuxCNC motion component](https://linuxcnc.org/docs/html/man/man9/motion.9.html)
