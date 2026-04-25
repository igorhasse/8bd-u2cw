# Analog Triggers (LT/RT) — Design Spec

**Date:** 2026-04-25
**Driver:** `8bd-u2cw` (8BitDo Ultimate 2C Linux driver)
**Target version:** 0.4.0
**Author of spec:** Igor Hasse Santiago

## Problem

In the current driver (v0.3.0), the LT and RT shoulder triggers of the 8BitDo
Ultimate 2C are exposed to the Linux input subsystem as **digital buttons**
(`BTN_TL2`, `BTN_TR2`) using a hardcoded threshold of byte value > 32. This
makes the triggers behave as on/off in games — unusable for racing games,
flight sims, or any title (notably the **shadps4** PS4 emulator with its
pressure-sensitive R2/L2) that relies on analog trigger pressure.

The original author left commented-out code intended to expose them as
`ABS_Z`/`ABS_RZ` axes, with the note *"does not work as expected"*, and
shipped the digital-button workaround instead.

## Investigation Summary

The USB report layout used by the existing `gamepad_in_cb` callback is
correct: `data[4]` carries LT pressure and `data[5]` carries RT pressure.

By enabling `print_hex_dump` and capturing 8,286 packets in `/tmp/triggers-new.log`
while pressing the triggers gradually from rest to full pressure, we
confirmed both bytes traverse the **full 0x00–0xff range continuously**, with
values present at virtually every intermediate level (0, 1, 2, ... 0x7f, 0x80,
... 0xfe, 0xff). The hardware sends genuine analog pressure data on Hall-effect
sensors. There is no flicker or quantization — the data is clean.

**The problem is exposure-only.** The driver discards real analog data and
emits digital events.

## Likely root cause of original author's failure

The author probably attempted to expose `ABS_Z`/`ABS_RZ` while *also* keeping
`BTN_TL2`/`BTN_TR2` active and did not update `sdl_profile_export.sh` to use
axis mapping (`a4`/`a5`) instead of button mapping (`b6`/`b7`). That would
leave SDL still seeing buttons via the controller config, plus duplicate
events, plus heuristics confusion — masking the fact that the analog axes
were working at the kernel level.

## Decision

Adopt **approach A — Xbox-style analog axes only.** Remove the digital
fallback entirely. Match what the in-tree `xpad` driver does for Xbox
controllers, which is the convention every modern SDL/evdev consumer expects.

### Approaches considered and rejected

- **B (axes + buttons, dual exposure):** Reproduces the exact pattern that
  failed for the original author. Risks SDL heuristic confusion and
  double-events in games that listen broadly. Maintenance burden for marginal
  compatibility gain.
- **C (module parameter, configurable):** Overengineering. The choice is
  binary, the right answer is known, and a `modprobe` flag adds runtime
  complexity (and test surface) for no real-world benefit.

## Design

### 1. Driver changes — `8bd-u2cw.c`

**State struct (`gamepad_state`, lines 86–90):**
- Remove `bool trigger_lt_button` and `bool trigger_rt_button`.
- Keep `uint8_t trigger_lt` and `uint8_t trigger_rt`.

**Capabilities (`gamepad_input_connect`, around lines 244–250):**
- Remove `input_set_capability(device, EV_KEY, BTN_TL2)` and `BTN_TR2`.
- Activate (uncomment) the analog axis registration:
  ```c
  input_set_abs_params(device, ABS_Z,  0, 255, 0, 0);
  input_set_abs_params(device, ABS_RZ, 0, 255, 0, 0);
  ```
  Range 0–255 matches the raw byte value from the USB report and matches
  evdev convention for unsigned trigger axes. `fuzz=0` and `flat=0` are
  appropriate because the captured data is stable at rest.

**Event reporting (`gamepad_input_process`, lines 342–348):**
- Remove the two `input_report_key` calls for `BTN_TL2`/`BTN_TR2`.
- Activate (uncomment):
  ```c
  input_report_abs(device, ABS_Z,  state->trigger_lt);
  input_report_abs(device, ABS_RZ, state->trigger_rt);
  ```

**USB callback (`gamepad_in_cb`, lines 392–403):**
- Keep `state->trigger_lt = data[4]` and `state->trigger_rt = data[5]` —
  already correct.
- Delete the "Virtual buttons from triggers" threshold block (lines 396–403)
  — dead code once the virtual buttons are gone.

**Cleanup:**
- Restore `print_hex_dump` on line 367 to a commented-out state (debug aid
  toggled during investigation).
- Bump `DRIVER_VERSION` from `"0.3.0"` to `"0.4.0"`.
- Update header `Known issues` block: remove the line stating triggers
  behave as buttons.
- Add a new `Version history` entry: `v0.4 - 2026-04-25 - Analog triggers
  (LT/RT) exposed as ABS_Z/ABS_RZ`.

### 2. SDL profile — `sdl_profile_export.sh`

Change the mapping fragment on line 3 from:

```
lefttrigger:b6,righttrigger:b7
```

to:

```
lefttrigger:a4,righttrigger:a5
```

Rationale: SDL maps evdev axes in canonical order — `ABS_X=a0, ABS_Y=a1,
ABS_RX=a2, ABS_RY=a3, ABS_Z=a4, ABS_RZ=a5`. Same convention used by xpad and
expected by the SDL game-controller database for Xbox-style devices.

The device GUID (`0300daf3c82d00000a31000014010000`) does **not** change —
USB vendor/product IDs stay identical (0x2dc8 / 0x310a).

### 3. README updates — `README.md`

- Remove the "Known issues" line about triggers being detected as buttons
  (line 13).
- Add a migration note in Installation Step 3 for users upgrading from
  v0.3.x: their existing `~/.profile` will contain the old
  `SDL_GAMECONTROLLERCONFIG` line. Best practice is to open the file and
  replace the old definition with the new one, rather than appending a
  second line. Both work (later definitions override earlier ones via the
  shell concatenation logic in the export script), but appending pollutes
  the profile.
- Add a brief changelog/highlight: "v0.4 — analog triggers fixed (LT/RT now
  pressure-sensitive on Xbox layout)".

## Test plan

### Low-level (driver exposure)

1. **Rebuild and reload:**
   ```bash
   sudo rmmod 8bd_u2cw && make clean && make && sudo insmod 8bd-u2cw.ko
   ```

2. **Capabilities via `evtest`:** Select the gamepad device. Confirm:
   - `ABS` includes `ABS_X`, `ABS_Y`, `ABS_Z`, `ABS_RX`, `ABS_RY`, `ABS_RZ`,
     `ABS_HAT0X`, `ABS_HAT0Y`.
   - `KEY` does **not** include `BTN_TL2` or `BTN_TR2`.

3. **Analog behavior:** Press LT slowly — `EV_ABS code 2 (ABS_Z)` events
   should pass through intermediate values (e.g., 0, 30, 80, 150, 220, 255).
   Same for RT on `code 5 (ABS_RZ)`. At rest, no spurious events.

### `jstest` (legacy joystick API)

```bash
jstest /dev/input/js0
```

Expect 6 axes active (not 4), with LT/RT showing the full -32767..32767
range as you press (jstest auto-rescales unsigned 0–255 to signed full
range). Button count drops to 11.

### SDL (target consumer)

- `sdl-jstest` / `sdl2-jstest`: device should report `lefttrigger` and
  `righttrigger` with values 0–32767.
- A racing game: progressive throttle response, not bang-bang.
- **shadps4** (explicit use case): R2/L2 should be pressure-sensitive in any
  game that uses analog triggers.

### Regression checks (must still work)

- Left/right stick analog axes (deadzone, full range, Y not inverted).
- D-pad (4 cardinals + diagonals).
- A/B/X/Y (X and Y intentionally swapped, as in current code).
- LB/RB shoulder buttons.
- Force feedback / rumble (via `fftest` or in-game).
- L4/R4 macro activation (combo unchanged).
- Heartbeat combo (Plus+Minus+LB+RB) still logs to `dmesg`.

### Rollback plan

If anything critical breaks: `git checkout` the previous tag/commit,
`make clean && make`, `sudo rmmod 8bd_u2cw && sudo insmod 8bd-u2cw.ko`.
Single-module driver — rollback is one step.

## Implementation order

Suggested commit sequence (each independently revertible):

1. **Core driver change** — all `.c` modifications (state, caps, processing,
   USB callback cleanup, version bump, header comments).
2. **SDL profile** — `sdl_profile_export.sh` axis mapping.
3. **Docs** — `README.md` updates.
4. **Debug cleanup** — re-comment `print_hex_dump` (can be folded into 1).

Splitting profile from driver lets us roll back the SDL mapping
independently if SDL-side issues surface.

## Risks

- **R1 — Stale `~/.profile`:** Users who already ran
  `cat sdl_profile_export.sh >> ~/.profile` will end up with two
  `SDL_GAMECONTROLLERCONFIG` definitions after upgrade. Shell concatenation
  honors the later one, so it works, but it's dirty. **Mitigation:** README
  spells it out; verify with
  `unset SDL_GAMECONTROLLERCONFIG && source ./sdl_profile_export.sh` before
  trusting the test result.
- **R2 — Range mismatch with SDL Xbox heuristic:** Some Xbox-style
  conventions use 0–1023 instead of 0–255. The `xpad` driver uses 0–1023 for
  Xbox 360. If SDL ends up rescaling oddly or refusing the device,
  switching `input_set_abs_params(..., 0, 1023, 0, 0)` is a one-line
  fallback. The `sdl2-jstest` step will surface this.
- **R3 — Out-of-scope packet bytes:** The captured packets show non-zero
  oscillating bytes at offsets 14–18 (`10 1c 30 64 ...`) even at rest. They
  may carry battery, latency, or sensor data. **Out of scope** for this
  change; document only.

## Out of scope

- L4/R4 reliability improvements.
- Bluetooth support.
- Battery/connection-status reporting (the unused report bytes referenced
  in R3 above).
- Adding a custom udev hwdb entry to give SDL the mapping without the
  `SDL_GAMECONTROLLERCONFIG` env var.
