# Keymapper BLE Protocol & Schema Reference

## BLE Connection

| Property | Value |
|----------|-------|
| Service UUID | `4fafc201-1fb5-459e-8fcc-c5c9c331914b` |
| Config characteristic | `beb5483e-36e1-4688-b7f5-ea07361b26a8` — Read + Write With Response |
| Input characteristic | `beb5483e-36e1-4688-b7f5-ea07361b26a9` — Read + Notify |
| Preferred ATT MTU | 512 bytes |
| Transport | JSON request/response on the config characteristic; binary notify on the input characteristic |

The config characteristic carries the command protocol below. The input
characteristic is a separate, one-way live feed of physical input state — see
[Live Input Mirror](#live-input-mirror).

The web app writes a JSON command in **one** `writeValueWithResponse` call, then
immediately reads the response. Commands larger than the negotiated MTU (a `put`
with a full profile is ~1 KB) are carried by the ATT layer as a queued/long
write; the firmware's GATT stack reassembles the fragments and delivers the
complete command to a single write callback. The web app must **not** pre-split
a command into multiple writes — each write is an independent ATT transaction,
so the firmware would receive JSON fragments, not one command. The firmware must
have a response ready before the read.

---

## Command Protocol

All commands and responses are UTF-8 JSON.

### `ls` — List all profiles

**Request**
```json
{ "cmd": "ls" }
```

**Response**
```json
{
  "ids": ["m9k2r4", "n1p8z9"],
  "liveId": "m9k2r4"
}
```

---

### `get` — Read one profile

**Request**
```json
{ "cmd": "get", "id": "m9k2r4" }
```

**Response** — full profile object (see Profile Schema below)
```json
{
  "version": 4,
  "id": "m9k2r4",
  "name": "Racing Game",
  "group": "Gaming",
  "bindings": { ... },
  "joystick": { "mode": "analog" }
}
```

---

### `put` — Write / create / update a profile

**Request**
```json
{
  "cmd": "put",
  "profile": {
    "version": 4,
    "id": "m9k2r4",
    "name": "Racing Game",
    "group": "Gaming",
    "bindings": { ... },
    "joystick": { "mode": "analog" }
  }
}
```

**Response**
```json
{ "ok": true }
```

---

### `del` — Delete a profile file

**Request**
```json
{ "cmd": "del", "id": "m9k2r4" }
```

**Response**
```json
{ "ok": true }
```

---

### `live` — Set the active profile

**Request**
```json
{ "cmd": "live", "id": "m9k2r4" }
```

**Response**
```json
{ "ok": true }
```

---

## Live Input Mirror

> Unrelated to the `live` command above, which sets the active profile. This is
> the input characteristic `beb5483e-36e1-4688-b7f5-ea07361b26a9`.

A one-way feed of which physical inputs are held right now, so the web app can
light up a key as it is pressed on the device.

**Payload** — 4 bytes, a **little-endian `uint32` bitmap**. Bit *n* is set while
the input at index *n* is held. Not JSON: this is a hot path on the 2 ms scan
loop, and a 4-byte packed frame keeps it off the allocator.

```js
const bits = dataView.getUint32(0, /* littleEndian */ true);
const k1Held = !!(bits & (1 << 0));
```

**Bit order** — matches `INPUT_IDS[]` in the firmware's `src/config.c` and
`LIVE_INPUT_ORDER` in the web app. The wire carries positions, not names, so
reordering one side without the other silently mismaps inputs instead of failing:

| Bit | Input | Bit | Input | Bit | Input |
|-----|-------|-----|-------|-----|-------|
| 0–11 | `K1`…`K12` | 12 | `HAT_U` | 13 | `HAT_D` |
| 14 | `HAT_L` | 15 | `HAT_R` | 16 | `HAT_C` |
| 17 | `S1` | 18 | `S2` | 19 | `JD_U` |
| 20 | `JD_D` | 21 | `JD_L` | 22 | `JD_R` |

Bits 23–31 are reserved and currently always zero.

**Semantics**

- **Notified on change only.** The scan loop runs at 2 ms, but a frame is only
  sent when the bitmap differs from the last one sent — a held key is silent.
  Inputs are already debounced in `inputs_scan`, so the feed does not chatter.
- **State, not events.** Each frame is the complete current state, so a client
  can paint directly from it and never has to track press/release pairs.
- **Coalescing.** If the BLE stack falls behind, pending frames collapse into the
  newest state rather than queueing a backlog — the latest state always wins. A
  tap shorter than the radio's catch-up window can therefore be missed; this feed
  is a mirror of current state, not an input log. HID reporting is unaffected.
- **Sent on subscribe.** The device pushes current state when a client subscribes,
  so a fresh subscriber does not sit blank until the next press.
- **READ** returns the same 4-byte payload on demand, and works whether or not
  anyone is subscribed.
- **`JD_*` are digital-mode only.** `inputs_scan` derives them from the stick only
  when `joystick.mode` is `digital`; in `analog`/`mouse` those bits stay zero.
  The analog stick position is not carried on this feed.
- **Requires USB.** The scan loop only runs once the USB host has enumerated the
  device, so nothing is published while it is unplugged.

**Compatibility** — firmware predating this feature has no input characteristic.
Clients must treat a missing characteristic as "mirroring unavailable" and keep
the connection usable for editing.

---

## On-Device File Layout

```
/profiles/
    m9k2r4.json       ← one file per profile, named by profile id
    n1p8z9.json
    ...
/config.json          ← tracks which profile is currently active
```

### `/config.json`
```json
{
  "liveId": "m9k2r4"
}
```

### `/profiles/<id>.json`
Full profile object — see Profile Schema below.

---

## Profile Schema

```json
{
  "version": 4,
  "id": "m9k2r4",
  "name": "Racing Game",
  "group": "Gaming",
  "bindings": {
    "K1": { "type": "key", "key": "A" },
    "K2": { "type": "key", "key": "C", "mods": ["CTRL"] },
    "HAT_U": { "type": "mouse", "action": "scroll_up" },
    "S1": { "type": "macro", "steps": [
      { "type": "key", "keys": ["C"], "mods": ["CTRL"] },
      { "type": "delay", "ms": 250 },
      { "type": "gamepad", "button": "A" }
    ] }
  },
  "joystick": {
    "mode": "analog"
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `version` | integer | `4`. Version `3` also loads — the two differ only in macro step form (see Macro below) |
| `id` | string | Immutable, `[0-9a-z]` only, used as filename |
| `name` | string | Human-readable, freely editable |
| `group` | string | Group label (e.g. `"Gaming"`). Empty string = ungrouped |
| `bindings` | object | Map of input ID → binding. Omitted inputs have no assignment |
| `joystick` | object | `{ "mode": "analog" \| "digital" \| "mouse" }` |

---

## Input IDs

### Grid keys
`K1` `K2` `K3` `K4` `K5` `K6` `K7` `K8` `K9` `K10` `K11` `K12`

### Hat switch
`HAT_U` `HAT_D` `HAT_L` `HAT_R` `HAT_C`

### Thumb buttons
`S1` `S2`

### Digital joystick directions
`JD_U` `JD_D` `JD_L` `JD_R`

> Only present in `bindings` when joystick mode is `"digital"`.

---

## Binding Types

### Keyboard
```json
{ "type": "key", "key": "A" }
{ "type": "key", "key": "C", "mods": ["CTRL"] }
{ "type": "key", "keys": ["A", "B"], "mods": ["CTRL", "SHIFT"] }
{ "type": "key", "mods": ["SHIFT"] }
```
Valid keys: `A`–`Z`, `0`–`9`, `F1`–`F12`, `ENTER`, `SPACE`, `ESCAPE`, `TAB`, `BACKSPACE`, `DELETE`, `UP`, `DOWN`, `LEFT`, `RIGHT`, `HOME`, `END`, `PGUP`, `PGDN`

| Field | Type | Notes |
|-------|------|-------|
| `key` | string | A single key |
| `keys` | array | Up to 4 keys held together. Use instead of `key`; longer chords are truncated |
| `mods` | array | Modifiers held down with the keys. Optional |

Modifiers and keys are held for as long as the input is pressed. `mods` alone
(no key) is valid — a button that just holds Shift.

**Valid modifiers:** `CTRL` `SHIFT` `ALT` `GUI` (bare names mean the left-hand
key), `LCTRL` `RCTRL` `LSHIFT` `RSHIFT` `LALT` `RALT` `LGUI` `RGUI` for a
specific side, and `CMD` `WIN` `META` (= `LGUI`), `OPT` (= `LALT`).

An unknown key or modifier drops the whole binding rather than part of it, so a
typo can't silently turn `Ctrl+C` into a bare `C`.

### Mouse
```json
{ "type": "mouse", "action": "left_click" }
```
Valid actions: `left_click`, `right_click`, `middle_click`, `scroll_up`, `scroll_down`, `button4`, `button5`

### Gamepad
```json
{ "type": "gamepad", "button": "A" }
```
Valid buttons: `A`, `B`, `X`, `Y`, `LB`, `RB`, `LT`, `RT`, `L3`, `R3`, `Start`, `Select`, `Home`

### Media
```json
{ "type": "media", "action": "play_pause" }
```
Valid actions: `play_pause`, `stop`, `next`, `prev`, `vol_up`, `vol_down`, `mute`

### System
```json
{ "type": "system", "action": "copy" }
```
Valid actions: `copy`, `paste`, `cut`, `undo`, `redo`, `save`, `select_all`, `find`, `screenshot`, `lock_screen`

### Macro
```json
{
  "type": "macro",
  "retrigger": "restart",
  "steps": [
    { "type": "key", "keys": ["C"], "mods": ["CTRL"] },
    { "type": "delay", "ms": 250 },
    { "type": "mouse", "action": "left_click" },
    { "type": "gamepad", "button": "A" },
    { "type": "media", "action": "play_pause" },
    { "type": "system", "action": "save" },
    { "type": "key", "keys": ["ENTER"], "hold": 100 }
  ]
}
```

An ordered array of steps, played once per press (not repeated while held). Each
step is asserted, held, then released before the next one starts. A macro fires
on the press edge and always plays to the end — releasing the input early doesn't
cut it short.

Playback never blocks the rest of the device: other inputs keep scanning and
reporting while a macro runs, several macros can run at once, and a step is
merged with whatever is physically held rather than replacing it (hold Shift
during a macro and its keys arrive shifted).

Steps reuse the binding vocabulary above — `{"type":"gamepad","button":"A"}`
means the same thing in a macro as it does as a binding — plus a `delay` step:

| Step type | Fields | Effect |
|-----------|--------|--------|
| `key` | `key` / `keys`, `mods` | Press a chord, then release it. Same fields as a keyboard binding |
| `mouse` | `action` | Click a button, or emit one scroll tick |
| `gamepad` | `button` | Press + release a gamepad button |
| `media` | `action` | Press + release a consumer-control key |
| `system` | `action` | The named shortcut, played as a keyboard chord |
| `delay` | `ms` | Wait, sending nothing. `ms` is required and must be > 0 |

`type` is optional and defaults to `"key"`, so `{"keys":["C"],"mods":["CTRL"]}`
is a valid step.

`hold` (optional, milliseconds) sets how long that step is held before release;
it defaults to 12 ms. Both `hold` and `ms` are clamped to 5000 ms.

`retrigger` (optional) says what a press does while that same macro is still
playing:

| Value | Effect |
|-------|--------|
| `"restart"` | **Default.** Release the current step and play again from the top |
| `"ignore"` | Let the run finish; the press does nothing |

It's the *default* when absent, so profiles written before the field existed —
and every v3 profile — behave as `"restart"`. An unrecognised value falls back to
`"restart"` with a warning rather than dropping the macro. `retrigger` only
governs re-pressing the same input; a different macro always starts on its own.

**Shorthand:** a bare string is a plain key, so the v3 form
`{"type":"macro","steps":["H","I"]}` still loads and means what it always did.

**Limits and error handling:** 32 steps max (longer sequences are truncated) and
4 keys per chord. An unusable step is skipped with a warning and the rest of the
macro still plays; a macro with no usable steps leaves the input unbound.

---

## Joystick Modes

| Mode | Firmware behaviour |
|------|-------------------|
| `analog` | Report raw X/Y axis as HID gamepad left stick |
| `digital` | Map 4 directions to the `JD_U/D/L/R` input IDs in `bindings` |
| `mouse` | Use axis movement to drive cursor |

---

## Web App Behaviour Summary

| Event | Web app sends |
|-------|--------------|
| Device connects | `ls` → sequential `get` for each id; subscribes to the input characteristic |
| Binding/joystick edited | `put` for the one changed profile (debounced 500 ms) |
| New profile created | `put` for the new profile |
| Profile imported from file | `put` for the updated profile |
| Profile deleted | `del` for the removed id; `live` if it was the active profile |
| Set Active pressed | `live` only — no profile data write |
