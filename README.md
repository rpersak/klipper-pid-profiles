# klipper-pid-profiles

Automatic PID profile switching for [Klipper](https://www.klipper3d.org/) — vanilla Klipper, no extra modules required.

Selects the right hotend and bed PID values based on target temperature, so you always get optimal heating behaviour regardless of the material you print.

| Profile | Hotend | Bed | Typical materials |
|---|---|---|---|
| **STANDARD** | < 250 °C | ≤ 85 °C | PLA, PETG, TPU |
| **HIGH** | ≥ 250 °C | > 85 °C | ABS, ASA, PC |

Every time `M109` or `M190` is called (by your slicer or macros), the correct PID is applied automatically before heating starts. No slicer changes needed.

---

## Files

| File | For |
|---|---|
| `pid_profiles_general.cfg` | Any Klipper printer |
| `pid_profiles_sv08.cfg` | Sovol SV08 with the matching `Macro.cfg` |
| `pid_calibrate.py` | Sovol SV08 only — patched Klipper module that adds the missing `SET_HEATER_PID` command |

---

## Installation

### Generic printer — `pid_profiles_general.cfg`

#### Option A — you do NOT already override `M109`/`M190`

1. Copy `pid_profiles_general.cfg` to your Klipper config directory.

2. Add the following to `printer.cfg`:
   ```ini
   [include pid_profiles_general.cfg]
   [pid_calibrate]
   ```
   > `[pid_calibrate]` enables Klipper's `SET_HEATER_PID` and `PID_CALIBRATE` commands.
   > Without it you will get `Unknown command: "SET_HEATER_PID"`.

3. Fill in your calibrated PID values (see [Calibration](#calibration) below).

4. Restart Klipper (`FIRMWARE_RESTART`).

#### Option B — you already override `M109`/`M190` in your own macros

1. Copy `pid_profiles_general.cfg` to your Klipper config directory.

2. Add the following to `printer.cfg`:
   ```ini
   [include pid_profiles_general.cfg]
   [pid_calibrate]
   ```
   > `[pid_calibrate]` enables Klipper's `SET_HEATER_PID` and `PID_CALIBRATE` commands.
   > Without it you will get `Unknown command: "SET_HEATER_PID"`.

3. Delete (or comment out) the `M109` and `M190` sections at the bottom of `pid_profiles_general.cfg` to avoid duplicate macro errors.

4. In your **existing** `M109` macro, add these two lines just before the `M104` call:
   ```jinja
   {% if s != 0 %}
       _APPLY_PID HEATER=extruder TARGET={s}
   {% endif %}
   ```

5. In your **existing** `M190` macro, add these two lines just before the `M140` call:
   ```jinja
   {% if s != 0 %}
       _APPLY_PID HEATER=heater_bed TARGET={s}
   {% endif %}
   ```

6. Fill in your calibrated PID values (see [Calibration](#calibration) below).

7. Restart Klipper (`FIRMWARE_RESTART`).

---

### Sovol SV08 — `pid_profiles_sv08.cfg`

This version assumes you are using the SV08 `Macro.cfg` that already contains the `_APPLY_PID` hooks in `M109` and `M190`. If you are not, follow Option B above using `pid_profiles_general.cfg` instead.

#### Step 1 — Patch `pid_calibrate.py`

Sovol's Klipper fork ships a modified `pid_calibrate.py` that is **missing the `SET_HEATER_PID` command**. Without it, `_APPLY_PID` will fail every time the heater is set:

```
!! Unknown command: "SET_HEATER_PID"
```

Fix this **once** before the first `FIRMWARE_RESTART`:

1. Copy `pid_calibrate.py` from this repo to the printer at:
   ```
   /home/sovol/klipper/klippy/extras/pid_calibrate.py
   ```
   Use SCP, Mainsail's file editor, or an SFTP client (e.g. WinSCP, FileZilla).

2. This file is Sovol's original `pid_calibrate.py` with only `SET_HEATER_PID` added — no other behaviour is changed.

> **After a Klipper update via Sovol's updater** this file may be overwritten. If `SET_HEATER_PID` stops working after an update, re-copy the file and do `FIRMWARE_RESTART`.

---

#### Step 2 — Required changes to `Macro.cfg`

Open `Macro.cfg` and update the `M109` and `M190` macros as follows.

**M109 — before:**
```jinja
[gcode_macro M109]
rename_existing: M99109
gcode:
    {% set s = params.S|float %}
    M104 {% for p in params %}{'%s%s' % (p, params[p])}{% endfor %}
    {% if s != 0 %}
        TEMPERATURE_WAIT SENSOR=extruder MINIMUM={s-1} MAXIMUM={s+2}
    {% endif %}
```

**M109 — after:**
```jinja
[gcode_macro M109]
rename_existing: M99109
gcode:
    {% set s = params.S|float %}
    {% if s != 0 %}
        _APPLY_PID HEATER=extruder TARGET={s}
    {% endif %}
    M104 {% for p in params %}{'%s%s' % (p, params[p])}{% endfor %}
    {% if s != 0 %}
        TEMPERATURE_WAIT SENSOR=extruder MINIMUM={s-1} MAXIMUM={s+2}
    {% endif %}
```

**M190 — before:**
```jinja
[gcode_macro M190]
rename_existing: M99190
gcode:
    {% set s = params.S|float %}
    M140 {% for p in params %}{'%s%s' % (p, params[p])}{% endfor %}
    {% if s != 0 %}
        TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={s-1} MAXIMUM={s+2}
    {% endif %}
```

**M190 — after:**
```jinja
[gcode_macro M190]
rename_existing: M99190
gcode:
    {% set s = params.S|float %}
    {% if s != 0 %}
        _APPLY_PID HEATER=heater_bed TARGET={s}
    {% endif %}
    M140 {% for p in params %}{'%s%s' % (p, params[p])}{% endfor %}
    {% if s != 0 %}
        TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={s-1} MAXIMUM={s+2}
    {% endif %}
```

#### Step 3 — Add the include to `printer.cfg`

```ini
[include Macro.cfg]
[include pid_profiles_sv08.cfg]   ← add this line
[pid_calibrate]                   ← add this line
```

> `[pid_calibrate]` enables Klipper's `SET_HEATER_PID` and `PID_CALIBRATE` commands.
> Without it you will get `Unknown command: "SET_HEATER_PID"`.

#### Step 4 — Fill in your PID values and restart

See [Calibration](#calibration) below, then `FIRMWARE_RESTART`.

---

## Calibration

Run each command once with the printer idle. Wait for Klipper to complete the calibration cycle (it takes a few minutes per run).

```
PID_CALIBRATE HEATER=extruder    TARGET=220   # hotend standard
PID_CALIBRATE HEATER=extruder    TARGET=260   # hotend high
PID_CALIBRATE HEATER=heater_bed  TARGET=65    # bed standard
PID_CALIBRATE HEATER=heater_bed  TARGET=100   # bed high
```

After each run, Klipper prints something like:
```
// PID parameters: pid_Kp=33.838 pid_Ki=5.223 pid_Kd=47.752
```

Copy the three values into the matching variables in your `.cfg` file:

```ini
variable_extruder_standard_kp: 33.838
variable_extruder_standard_ki: 5.223
variable_extruder_standard_kd: 47.752
```

> **Do NOT use `SAVE_CONFIG`** after calibration — that would overwrite your `printer.cfg` values and bypass this system. Just copy the numbers manually into the profile file.

---

## How it works

`_APPLY_PID` is called at the start of every `M109`/`M190` command, before the heater target is set. It reads the target temperature, picks the right profile, and runs Klipper's built-in `SET_HEATER_PID` command to update the active PID parameters. The heater then runs with the correct values from the very first second.

Each switch is logged in the Klipper console:
```
PID: extruder STANDARD (220 C)
PID: bed HIGH (100 C)
```

---

## Customising temperature thresholds

The breakpoints (250 °C for hotend, 85 °C for bed) are set directly in the `_APPLY_PID` macro. Edit the `{% if target >= 250 %}` and `{% if target > 85 %}` conditions to match your needs.

You can also extend the macro to three or more profiles by adding `{% elif %}` branches.

---

## Comparison with other solutions

| | **klipper-pid-profiles** | [klipper-better-pid](https://github.com/gilbertorconde/klipper-better-pid) | [Kalico](https://github.com/KalicoCrew/kalico) |
|---|---|---|---|
| Vanilla Klipper | ✅ | ❌ (external module) | ❌ (firmware fork) |
| Installation | copy one `.cfg` | `install.sh` + systemd | replace Klipper |
| Profile switching | discrete profiles | smooth interpolation | built-in command |
| Configuration | variables in `.cfg` | separate config file | G-code commands |

Use **klipper-better-pid** if you need smooth interpolation across many calibration points. Use this project if you want a simple, dependency-free solution that lives entirely in your config.

---

## License

MIT
