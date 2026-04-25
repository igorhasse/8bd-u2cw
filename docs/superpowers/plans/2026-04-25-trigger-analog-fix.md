# Analog Triggers (LT/RT) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expose the 8BitDo Ultimate 2C LT/RT shoulder triggers as analog Xbox-style axes (`ABS_Z`/`ABS_RZ`) instead of digital buttons, restoring pressure-sensitive trigger input for racing games, flight sims, and emulators like shadps4.

**Architecture:** Kernel module change — replace `BTN_TL2`/`BTN_TR2` digital fallback with `ABS_Z`/`ABS_RZ` analog axis exposure. The USB report already carries analog pressure data on bytes 4 and 5; the driver simply needs to forward those values as `EV_ABS` events. SDL profile updated in lockstep to reference axis indices instead of button indices.

**Tech Stack:** Linux kernel input subsystem (`<linux/input.h>`), USB driver framework, SDL `SDL_GAMECONTROLLERCONFIG`. Kernel build via `make` (Kbuild). No unit-test harness — verification is manual via `evtest`, `jstest`, and SDL test apps.

**Reference spec:** `docs/superpowers/specs/2026-04-25-trigger-analog-fix-design.md`

**Pre-condition:** Working tree currently has `print_hex_dump` uncommented at `8bd-u2cw.c:367` (debug aid from investigation). The plan reverts this in Task 1 alongside the other changes.

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `8bd-u2cw.c` | Modify | Driver source — state struct, capabilities, event reporting, USB callback, version, comments |
| `sdl_profile_export.sh` | Modify | SDL game-controller mapping — change LT/RT from `b6`/`b7` to `a4`/`a5` |
| `README.md` | Modify | User-facing docs — remove "trigger as button" known-issue, add migration note |

No new files. All changes are in-place edits.

---

## Task 1: Apply core driver changes

This task bundles every `.c` modification into one coherent commit. Splitting the kernel-module edits across multiple commits would leave the tree in an unbuildable intermediate state (e.g., capability registered but state field removed). The granularity here is **one logical unit of work = one buildable commit**.

**Files:**
- Modify: `8bd-u2cw.c`

**Edits required (in order):**

- [ ] **Step 1: Update version history and known-issues block in header comment**

Find lines 14–15:
```c
 * Known issues:
 *    - Shoulder triggers LT and RT work as buttons rather than triggers
```

Replace with:
```c
 * Known issues:
 *    (none currently tracked)
```

Find line 28 (last entry of version history):
```c
 *  v0.3 - 2025-10-26 - Optimized cleanup
```

Add a new line directly after it:
```c
 *  v0.3 - 2025-10-26 - Optimized cleanup
 *  v0.4 - 2026-04-25 - Analog triggers (LT/RT) exposed as ABS_Z/ABS_RZ
```

- [ ] **Step 2: Bump driver version**

Find line 34:
```c
#define DRIVER_VERSION "0.3.0"
```

Replace with:
```c
#define DRIVER_VERSION "0.4.0"
```

- [ ] **Step 3: Remove virtual-button fields from `gamepad_state`**

Find lines 86–90:
```c
	// Shoulder trigger
	uint8_t trigger_lt;
	uint8_t trigger_rt;
	bool trigger_lt_button;
	bool trigger_rt_button;
```

Replace with:
```c
	// Shoulder trigger (analog)
	uint8_t trigger_lt;
	uint8_t trigger_rt;
```

- [ ] **Step 4: Replace `BTN_TL2`/`BTN_TR2` capability with `ABS_Z`/`ABS_RZ`**

Find lines 244–250:
```c
	input_set_capability(device, EV_KEY, BTN_TL2);
	input_set_capability(device, EV_KEY, BTN_TR2);

	/* LT and RT as trigger - does not work as expected
	input_set_abs_params(device, ABS_Z, 0, 255, 0, 0);
	input_set_abs_params(device, ABS_RZ, 0, 255, 0, 0);
	*/
```

Replace with:
```c
	// LT and RT as analog triggers (Xbox-style)
	input_set_abs_params(device, ABS_Z, 0, 255, 0, 0);
	input_set_abs_params(device, ABS_RZ, 0, 255, 0, 0);
```

- [ ] **Step 5: Replace trigger button reports with analog axis reports in `gamepad_input_process`**

Find lines 342–348:
```c
		input_report_key(device, BTN_TL2, state->trigger_lt_button);
		input_report_key(device, BTN_TR2, state->trigger_rt_button);

		/* LT and RT as trigger - does not work as expected
		input_report_abs(device, ABS_Z, state->trigger_lt);
		input_report_abs(device, ABS_RZ, state->trigger_rt);
		*/
```

Replace with:
```c
		input_report_abs(device, ABS_Z, state->trigger_lt);
		input_report_abs(device, ABS_RZ, state->trigger_rt);
```

- [ ] **Step 6: Remove threshold logic from `gamepad_in_cb`**

Find lines 391–403:
```c
		// Trigger
		state->trigger_lt         = data[4];
		state->trigger_rt         = data[5];

		// Virtual buttons from triggers
		if (state->trigger_lt < 16)
			state->trigger_lt_button = false;
		else if (state->trigger_lt > 32)
			state->trigger_lt_button = true;
		if (state->trigger_rt < 16)
			state->trigger_rt_button = false;
		else if (state->trigger_rt > 32)
			state->trigger_rt_button = true;
```

Replace with:
```c
		// Trigger (analog, full 0-255 range)
		state->trigger_lt         = data[4];
		state->trigger_rt         = data[5];
```

- [ ] **Step 7: Re-comment the debug `print_hex_dump` (cleanup of investigation aid)**

Find line 367:
```c
	print_hex_dump(KERN_INFO, DRIVER_NAME ": ", DUMP_PREFIX_OFFSET, 32, 1, gamepad->usb_in_data, PACKET_SIZE, 0);
```

Replace with:
```c
	// print_hex_dump(KERN_INFO, DRIVER_NAME ": ", DUMP_PREFIX_OFFSET, 32, 1, gamepad->usb_in_data, PACKET_SIZE, 0);
```

- [ ] **Step 8: Verify build**

Run:
```bash
cd /home/atenccion/personal-work/8bd-u2cw
make clean && make 2>&1 | tail -20
```

Expected: build succeeds, produces `8bd-u2cw.ko`. No warnings about unused variables (the removed `trigger_lt_button`/`trigger_rt_button` fields shouldn't be referenced anywhere now).

If a warning surfaces (e.g., a stray reference), grep for it and remove:
```bash
grep -n "trigger_lt_button\|trigger_rt_button\|BTN_TL2\|BTN_TR2" 8bd-u2cw.c
```
Expected output: empty.

- [ ] **Step 9: Commit**

```bash
git add 8bd-u2cw.c
git commit -m "$(cat <<'EOF'
feat(driver): expose LT/RT as analog ABS_Z/ABS_RZ axes

Replaces the digital BTN_TL2/BTN_TR2 fallback with full 0-255 analog
exposure so triggers behave as Xbox-style pressure-sensitive axes.
Removes the threshold-based virtual-button conversion that made racing
games and pressure-sensitive emulators (shadps4) unable to use trigger
gradients.

Bumps driver version to 0.4.0.
EOF
)"
```

---

## Task 2: Reload module and confirm clean load

**Files:**
- (Runtime only — no file changes)

- [ ] **Step 1: Unload current module and load freshly built one**

Run:
```bash
sudo rmmod 8bd_u2cw 2>/dev/null
sudo modprobe ff_memless
sudo insmod 8bd-u2cw.ko
```

Expected: no errors. `dmesg | tail -5` should show:
```
8bd-u2cw: Initialize gamepad 8BitDo Ultimate 2C (Driver 8bd-u2cw 0.4.0)
8bd-u2cw: Gamepad connected successfuly
```

If `rmmod` fails because the gamepad is in use (some process has the joystick device open), close that consumer first.

- [ ] **Step 2: Confirm version string is 0.4.0**

Run:
```bash
modinfo 8bd-u2cw.ko | grep version
```

Expected: a `version:` line **is not** strictly emitted (driver doesn't set MODULE_VERSION). Instead verify via `dmesg`:
```bash
sudo dmesg | grep "8bd-u2cw:" | tail -3
```
Expected: contains `Driver 8bd-u2cw 0.4.0`.

---

## Task 3: Verify capabilities via evtest

**Files:**
- (Runtime only)

- [ ] **Step 1: List input devices and find the gamepad**

Run:
```bash
sudo evtest
```

Pick the entry whose name contains `8BitDo Ultimate 2C`. Note the `/dev/input/eventN` number — it'll be referenced as `$EVDEV` below.

- [ ] **Step 2: Inspect the capability listing printed by evtest**

In the evtest output, look at the `Supported events:` block.

Expected:
- Under `Event type 1 (EV_KEY)`: BTN_A, BTN_B, BTN_X, BTN_Y, BTN_TL, BTN_TR, BTN_THUMBL, BTN_THUMBR, BTN_START, BTN_SELECT, BTN_MODE, BTN_TRIGGER_HAPPY1, BTN_TRIGGER_HAPPY2.
- Under `Event type 3 (EV_ABS)`: ABS_X, ABS_Y, ABS_Z, ABS_RX, ABS_RY, ABS_RZ, ABS_HAT0X, ABS_HAT0Y.

NOT expected (must be absent): `BTN_TL2`, `BTN_TR2`.

If `BTN_TL2`/`BTN_TR2` still appear, you reloaded the wrong `.ko`. Re-do Task 2.

- [ ] **Step 3: Inspect ABS axis params for ABS_Z and ABS_RZ**

In evtest output, find the lines for `ABS_Z (5)` and `ABS_RZ (6)`. They should show:
```
    Value     0
    Min       0
    Max     255
    Fuzz      0
    Flat      0
    Resolution      0
```

If `Value` reads non-zero at rest, the trigger is being held — release it and re-check.

- [ ] **Step 4: Stay in evtest, watch live events for a graduated trigger press**

Press LT slowly from rest to full press over ~3 seconds. Expected: a stream of lines like:
```
Event: time ..., type 3 (EV_ABS), code 2 (ABS_Z), value 17
Event: time ..., type 3 (EV_ABS), code 2 (ABS_Z), value 43
Event: time ..., type 3 (EV_ABS), code 2 (ABS_Z), value 91
...
Event: time ..., type 3 (EV_ABS), code 2 (ABS_Z), value 255
```

Do the same with RT — should produce events on `code 5 (ABS_RZ)`.

- [ ] **Step 5: Confirm absence of stray button events**

While pressing LT and RT, no events with `type 1 (EV_KEY)` and code matching `BTN_TL2 (0x137)` or `BTN_TR2 (0x138)` should ever fire. Quit evtest with Ctrl+C.

---

## Task 4: Update SDL profile

**Files:**
- Modify: `sdl_profile_export.sh`

- [ ] **Step 1: Change LT/RT from button mapping to axis mapping**

Find line 3 (the `export SDL_GAMECONTROLLERCONFIG=...` line). Locate the substring:
```
lefttrigger:b6,righttrigger:b7
```

Replace with:
```
lefttrigger:a4,righttrigger:a5
```

The rest of the line (GUID, all other mappings, `platform:Linux,`) stays unchanged.

- [ ] **Step 2: Verify the file by sourcing and printing**

Run:
```bash
unset SDL_GAMECONTROLLERCONFIG
source ./sdl_profile_export.sh
echo "$SDL_GAMECONTROLLERCONFIG" | grep -oE 'lefttrigger:[^,]+,righttrigger:[^,]+'
```

Expected output:
```
lefttrigger:a4,righttrigger:a5
```

- [ ] **Step 3: Commit**

```bash
git add sdl_profile_export.sh
git commit -m "$(cat <<'EOF'
feat(sdl): map LT/RT to analog axes (a4/a5)

Companion to driver v0.4.0 — SDL game-controller config now points
LT/RT at the new ABS_Z/ABS_RZ axes (canonical SDL indices a4/a5)
instead of the legacy b6/b7 button indices.

Users upgrading from v0.3.x: open ~/.profile and replace the old
SDL_GAMECONTROLLERCONFIG line, otherwise both definitions will end
up appended.
EOF
)"
```

---

## Task 5: Verify SDL exposes analog triggers

**Files:**
- (Runtime only)

- [ ] **Step 1: Check whether sdl2-jstest is installed**

Run:
```bash
which sdl2-jstest sdl-jstest 2>/dev/null
```

If neither is installed, install via your package manager (Arch: `sudo pacman -S sdl2-jstest`; Debian/Ubuntu: `sudo apt install sdl2-jstest` if available, otherwise skip to Step 4).

- [ ] **Step 2: Run sdl2-jstest with the updated profile sourced**

```bash
unset SDL_GAMECONTROLLERCONFIG
source ./sdl_profile_export.sh
sdl2-jstest --list
```

Expected: device shows up as a recognized GameController (not just a raw Joystick).

- [ ] **Step 3: Test the gamepad in interactive mode**

```bash
sdl2-jstest --test 0
```

Press LT and RT slowly. Expected: `Trigger Left` and `Trigger Right` rows show values progressing through intermediates (0 → ... → 32767 in SDL's signed scale, which corresponds to evdev 0 → 255).

If sdl2-jstest reports the triggers as **buttons** instead of **triggers**, the SDL config wasn't picked up — verify `echo $SDL_GAMECONTROLLERCONFIG` contains the new mapping. Common gotcha: a stale definition earlier in the env var wins; `unset` first.

- [ ] **Step 4: (Fallback if no sdl2-jstest) — open shadps4 directly**

Launch shadps4 with any game that uses analog R2/L2 (e.g., a racing game). In-game, accelerate progressively with R2. Expect progressive throttle response, not bang-bang. If the input still feels binary, `unset SDL_GAMECONTROLLERCONFIG && source ./sdl_profile_export.sh && shadps4 ...` to ensure the profile is loaded in shadps4's environment.

---

## Task 6: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Remove the trigger-as-button known-issue line**

Find lines 11–13:
```markdown
**Known issues**

* Shoulder triggers LT and RT are detected as buttons rather than triggers
```

Replace with:
```markdown
**Recent changes**

* v0.4 — Analog triggers (LT/RT) now work with full pressure sensitivity (Xbox-style)
```

- [ ] **Step 2: Add a migration note in Step 3**

Find lines 72–76:
```markdown
SDL based games may have a wrong gamepad mapping included. Add the correct mapping to your profile.

```bash
cat sdl_profile_export.sh >> ~/.profile
```
```

Replace with:
```markdown
SDL based games may have a wrong gamepad mapping included. Add the correct mapping to your profile.

```bash
cat sdl_profile_export.sh >> ~/.profile
```

> **Upgrading from v0.3.x?** Your `~/.profile` already has an older `SDL_GAMECONTROLLERCONFIG` line. Open the file and **replace** the old line with the new one rather than appending — otherwise both definitions will accumulate. Both still work (the later definition wins via shell concatenation), but appending pollutes your profile.
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
docs: update README for analog trigger fix

- Drop the "triggers as buttons" known-issue line
- Add v0.4 changelog highlight
- Add migration note for users upgrading from v0.3.x ~/.profile
EOF
)"
```

---

## Task 7: Regression sweep

**Files:**
- (Runtime only)

Validate that nothing else broke. Run each check; if any fails, stop and investigate before claiming completion.

- [ ] **Step 1: Sticks (left + right)**

Run `sudo evtest` on the gamepad. Move both sticks through full range. Expected: `ABS_X`/`ABS_Y` (left) and `ABS_RX`/`ABS_RY` (right) emit events from -32768 to +32767. Y axes should not be inverted (push stick up → negative Y, push down → positive Y, matching the kernel convention with the `-state->stick_left_y` mirror in `gamepad_input_process`).

- [ ] **Step 2: D-pad**

Press each of the 4 cardinal directions. Expected: `ABS_HAT0X` events with values -1/+1 for left/right; `ABS_HAT0Y` with -1/+1 for up/down. Press diagonal (e.g., up+right) — both axes fire simultaneously.

- [ ] **Step 3: Face buttons**

Press A, B, X, Y. Expected events:
- A button → `BTN_A (0x130)`
- B button → `BTN_B (0x131)`
- X button (top of diamond) → `BTN_Y (0x134)` (intentional swap, see `8bd-u2cw.c:318-319`)
- Y button (left of diamond) → `BTN_X (0x133)` (intentional swap)

- [ ] **Step 4: Shoulder buttons (LB/RB)**

Press LB and RB. Expected: `BTN_TL (0x136)` and `BTN_TR (0x137)`. Confirm these did **not** get accidentally renamed during the LT/RT changes.

- [ ] **Step 5: Stick buttons + middle buttons**

Press L3, R3, Plus, Minus, Menu (Xbox/Home). Expected: `BTN_THUMBL`, `BTN_THUMBR`, `BTN_START`, `BTN_SELECT`, `BTN_MODE` respectively.

- [ ] **Step 6: Force feedback / rumble**

If `fftest` is installed:
```bash
sudo fftest /dev/input/eventN  # same N as before
```
Choose the rumble effect. Expected: gamepad vibrates. If `fftest` is unavailable, run any game with rumble support and trigger an in-game rumble event.

- [ ] **Step 7: L4/R4 macro (experimental, only if previously activated)**

Skip if you've never run the activation macro. If you have, press L4 or R4 — expect `BTN_TRIGGER_HAPPY1` / `BTN_TRIGGER_HAPPY2`.

- [ ] **Step 8: Heartbeat combo**

Press Plus + Minus + LB + RB simultaneously. Run `sudo dmesg | tail -3`. Expected: a log line `8bd-u2cw: Heartbeat! (L + R + Plus + Minus)`.

---

## Task 8: Final acceptance — real game

**Files:**
- (Runtime only)

- [ ] **Step 1: Load updated SDL config in current shell**

```bash
unset SDL_GAMECONTROLLERCONFIG
source /home/atenccion/personal-work/8bd-u2cw/sdl_profile_export.sh
```

- [ ] **Step 2: Launch a real consumer**

Pick whichever you have available:
- A racing game (Steam: any racing title with controller support).
- shadps4 with any game using analog R2/L2.
- An emulator like RPCS3, PCSX2, Dolphin (most reliable for verifying analog triggers map correctly).

- [ ] **Step 3: Test progressive trigger response**

Apply progressively increasing pressure to LT and then RT. Acceptance criteria: in-game response (acceleration, brake, weapon charge, etc.) reflects pressure level continuously — not on/off. If the game has an input-test screen or controller-config screen, use it to confirm the trigger value moves through the full range.

- [ ] **Step 4: Decision point**

If progressive response works → done. If still feels binary in the game but fine in `evtest`/`sdl2-jstest`, the game-side mapping is wrong (check the game's controller config) — not a driver issue.

If it feels binary in `sdl2-jstest`, see "Risk R2" in the spec: try changing both `input_set_abs_params` calls to use `0, 1023, 0, 0` instead of `0, 255, 0, 0`. That requires rebuilding (return to Task 1, modify, rebuild, reload, retest).

---

## Task 9: Install for daily use

**Files:**
- (System-level — installs the module)

- [ ] **Step 1: Install for current kernel**

```bash
cd /home/atenccion/personal-work/8bd-u2cw
sudo make install
```

Expected: module copied into `/lib/modules/$(uname -r)/...`. `modprobe`/`depmod` invoked.

- [ ] **Step 2: Confirm xpad does not steal the device**

Check `/etc/modprobe.d/` for a blacklist file (the install via udev rule `99-8bd-u2cw.rules` should handle this — verify by inspecting the rule file if needed):
```bash
cat /home/atenccion/personal-work/8bd-u2cw/99-8bd-u2cw.rules
```

If `xpad` keeps grabbing the device on reboot, the install step's udev rule isn't applying — re-run `sudo make install` and `sudo udevadm control --reload-rules`.

- [ ] **Step 3: Reboot and verify it sticks**

Reboot. Replug the gamepad. `sudo dmesg | grep 8bd-u2cw` should show v0.4.0 loaded and the gamepad detected. Run any quick `sdl2-jstest --test 0` confirmation that triggers are still analog after a clean boot.

---

## Self-Review checklist (already done — listed for transparency)

- **Spec coverage:** Every section of the spec maps to a task. Section 1 (driver) → Task 1. Section 2 (SDL profile) → Task 4. Section 3 (README) → Task 6. Test plan low-level → Tasks 2-3. Test plan SDL → Task 5. Regression → Task 7. Final acceptance → Task 8.
- **Placeholder scan:** No TBDs, no "implement later", every step shows the actual code or command.
- **Type/name consistency:** `trigger_lt`/`trigger_rt` retained throughout. Removed fields (`trigger_lt_button`/`trigger_rt_button`) referenced consistently as removed in steps 3, 5, 6 of Task 1.
- **Risk handling:** R1 (stale `~/.profile`) → migration note in Task 6 + `unset` discipline in Tasks 5/8. R2 (range mismatch) → fallback in Task 8 step 4. R3 (mystery bytes) → out of scope, not in plan.
