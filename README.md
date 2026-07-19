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
immediately reads the response. The firmware must have a response ready before
the read.

### The 512-byte ceiling

**Every command and every response must fit in 512 bytes.** This is
`BLE_ATT_ATTR_MAX_LEN` — the maximum length of an ATT attribute value in the
Bluetooth spec. It is not a tunable, and it is **not** lifted by negotiating a
larger MTU or by using long reads / queued writes:

- A read of a longer value stops at 512 bytes. The client gets a truncated
  object and can only fail to parse it — there is no error, just corruption.
- A queued (long) write whose reassembled value exceeds 512 is rejected by the
  stack with `BLE_ATT_ERR_INVALID_ATTR_VALUE_LEN`.

A full profile is 1.5–2 KB, several times over. So `get` and `put` move a
profile as a series of windows — see [Chunked profile transfer](#chunked-profile-transfer).
`ls`, `del`, `live`, and `name` are small enough to fit in one operation.

Within a single command the web app must **not** pre-split the JSON into
multiple writes: each write is an independent ATT transaction, so the firmware
would receive JSON fragments rather than one command. Chunking happens at the
protocol level, above ATT — each chunk is its own complete, well-formed command.

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
  "liveId": "m9k2r4",
  "name": "Glide Controller"
}
```

`name` is the advertised BLE device name. It is read from the GAP service rather
than from `/config.json`, so it reflects what is actually being advertised —
including the built-in default when nothing has been stored.

---

### Chunked profile transfer

Both `get` and `put` carry a **window** of the profile's raw JSON text — the
exact bytes stored at `/profiles/<id>.json` — rather than a profile object.

The window is **base64**-encoded. It is not embedded directly as a JSON string
because the payload is itself JSON: escaping would expand it by an amount that
depends on the profile's content, so the frame size would cross 512 for some
profiles and not others. Base64 expands by a fixed 4/3, keeping every frame the
same predictable size whatever the profile contains.

| Constant | Value | Meaning |
|----------|-------|---------|
| `CHUNK_RAW` | 288 | raw bytes per window (→ 384 base64 chars) |
| `MAX_PROFILE` | 4096 | largest profile the firmware will reassemble |

Both sides must agree on `CHUNK_RAW`: `ble_config.c` and `index.html` each
define it.

Offsets are **byte** offsets into the UTF-8 text, so a window boundary can fall
inside a multi-byte character. Decode to text only after the whole profile is
reassembled — never per chunk.

---

### `get` — Read one profile

Fetches one window. The client repeats with `off` advanced by the number of
bytes decoded until it has `total` bytes.

**Request** — `off` may be omitted, and defaults to 0
```json
{ "cmd": "get", "id": "m9k2r4", "off": 0 }
```

**Response**
```json
{ "off": 0, "total": 1996, "data": "<base64 of up to 288 bytes>" }
```

`total` is the profile's full size in bytes. The final window is short. A
response with an empty `data` before `total` bytes have arrived is an error —
the client must fail rather than loop.

Errors: `invalid id`, `not found`, `bad offset`, `read failed`, `encode failed`.

---

### `put` — Write / create / update a profile

Sends one window. The firmware buffers in RAM and **only touches flash once the
final window arrives**, so a transfer abandoned partway leaves the stored
profile exactly as it was rather than overwriting it with a prefix.

**Request**
```json
{
  "cmd": "put",
  "id": "m9k2r4",
  "off": 0,
  "total": 1996,
  "data": "<base64 of up to 288 bytes>"
}
```

**Response** — intermediate window
```json
{ "ok": true, "got": 288 }
```

**Response** — final window, after the profile is validated and stored
```json
{ "ok": true, "done": true }
```

**Sequencing.** `off: 0` begins a new transfer and discards any partial one.
Every later window must continue the transfer in progress: same `id`, same
`total`, and `off` exactly equal to the bytes received so far. Anything else is
rejected with `out of sequence` and the partial transfer is dropped — this is
what stops a retried or reordered window from stitching two different profiles
into one corrupt file.

Only one transfer may be in flight. The firmware serves a single client and
tracks a single transfer, so **the client must not interleave other commands
between a transfer's chunks** — the whole chunk loop belongs inside one
serialized block. A partial transfer is also dropped on disconnect.

On the final window the firmware parses the reassembled text and checks that its
`id` matches the one the chunks were sent under, before writing. A profile that
fails either check is rejected rather than stored, since a stored profile that
doesn't parse would read back as `not found` forever.

Errors: `invalid id`, `missing data`, `missing off/total`, `bad offset`,
`bad data`, `chunk past total`, `out of sequence`, `out of memory`,
`profile parse error`, `id mismatch`, `write failed`.

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

### `name` — Rename the controller

Sets the advertised BLE device name — what the OS Bluetooth list shows. Stored
in `/config.json` and reapplied on every boot.

**Request**
```json
{ "cmd": "name", "name": "Left Hand" }
```

**Response**
```json
{ "ok": true }
```

Surrounding spaces are trimmed. The name must be 1–26 characters once trimmed,
and may not contain control characters; anything else is rejected with
`{"error":"name must be 1-26 printable characters"}` rather than truncated.

The 26-character ceiling is the advertising payload budget: the primary AD is 31
bytes, of which 3 go to flags and 2 to the name's own header.

The GAP name characteristic updates immediately, so a connected client sees the
new name without reconnecting. The advertisement carries it from the next time
advertising starts. Hosts cache the name from their last scan, so the system
Bluetooth list may keep showing the old one until it re-scans.

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
const p1Held = !!(bits & (1 << 0));
```

**Bit order** — matches `INPUT_IDS[]` / `input_id_t` in the firmware's
`src/config.c` and `include/config.h`, and `LIVE_INPUT_ORDER` in the web app. The wire carries positions, not names, so
reordering one side without the other silently mismaps inputs instead of failing:

| Bit | Input | Bit | Input | Bit | Input |
|-----|-------|-----|-------|-----|-------|
| 0–2 | `P1`…`P3` | 3–5 | `R1`…`R3` | 6–8 | `M1`…`M3` |
| 9–11 | `I1`…`I3` | 12 | `T1` | 13 | `T2` |
| 14 | `HAT_U` | 15 | `HAT_D` | 16 | `HAT_L` |
| 17 | `HAT_R` | 18 | `HAT_C` | 19 | `JD_U` |
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
/config.json          ← active profile + device name
```

### `/config.json`
```json
{
  "liveId": "m9k2r4",
  "name": "Glide Controller"
}
```

`name` is absent until the controller is renamed; the firmware falls back to
`"Glide Controller"`. Both keys live in this one file, so `live` and `name` each
read-modify-write it rather than rewriting it from scratch.

### `/profiles/<id>.json`
Full profile object — see Profile Schema below.

---

## Profile Schema

```json
{
  "version": 5,
  "id": "m9k2r4",
  "name": "Racing Game",
  "group": "Gaming",
  "bindings": {
    "P1": { "type": "key", "key": "A" },
    "P2": { "type": "key", "key": "C", "mods": ["CTRL"] },
    "HAT_U": { "type": "mouse", "action": "scroll_up" },
    "T1": { "type": "macro", "steps": [
      { "type": "key", "keys": ["C"], "mods": ["CTRL"] },
      { "type": "delay", "ms": 250 },
      { "type": "gamepad", "button": "A" }
    ] }
  },
  "joystick": {
    "mode": "analog"
  },
  "leds": {
    "P1": "#ff2d2d",
    "P2": "#2d6bff"
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `version` | integer | `5`. Versions `3` and `4` also load: they differ in macro step form (see Macro below) and use the pre-rename `K1`..`K12`/`S1`/`S2` input names, which are still accepted and mapped to the same physical keys |
| `id` | string | Immutable, `[0-9a-z]` only, used as filename |
| `name` | string | Human-readable, freely editable |
| `group` | string | Group label (e.g. `"Gaming"`). Empty string = ungrouped |
| `bindings` | object | Map of input ID → binding. Omitted inputs have no assignment |
| `joystick` | object | `{ "mode": "analog" \| "digital" \| "mouse" }` |
| `leds` | object | Optional. Map of grid/thumb key → `"#rrggbb"`. Omitted keys stay unlit. See LED Strip below |

---

## LED Strip

The device has one addressable RGB LED per key — the twelve grid keys and both
thumb keys. The hat switch and joystick have no LEDs. Each key shows the color
assigned in `leds` as a dim idle glow, jumping to full brightness while that
key is held. A key with no entry in `leds` (or an all-zero color) stays off.

`leds` is a plain map, independent of `bindings` — a key can have a light with
no binding, a binding with no light, both, or neither. Colors are `"#rrggbb"`
hex strings (lowercase or uppercase, `#` required); anything else for a given
key is rejected and that key is left unlit rather than failing the whole
profile. Missing from a profile entirely is the same as an empty `leds`
object — every key stays off.

Firmware written before this field existed simply has no `leds` handling, so a
profile carrying it still loads there — it just has no visible effect until
that firmware is updated.

---

## Input IDs

Inputs are named by finger and position: **P**inky, **R**ing, **M**iddle,
**I**ndex, **T**humb, numbered 1..3 outward from the palm.

### Grid keys
`P1` `P2` `P3` `R1` `R2` `R3` `M1` `M2` `M3` `I1` `I2` `I3`

### Hat switch
`HAT_U` `HAT_D` `HAT_L` `HAT_R` `HAT_C`

### Thumb buttons
`T1` `T2`

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
Valid keys: `A`–`Z`, `0`–`9`, `F1`–`F12`, `ENTER`, `SPACE`, `ESCAPE`, `TAB`, `BACKSPACE`, `DELETE`, `UP`, `DOWN`, `LEFT`, `RIGHT`, `HOME`, `END`, `PGUP`, `PGDN`, `CAPS_LOCK`, `INSERT`, `PRTSC`, `SCROLL_LOCK`, `NUM_LOCK`, `PAUSE`, `MENU`

The lock/system keys are ordinary keycodes, not modifiers — `CAPS_LOCK` toggles
on press exactly as it does on a keyboard, rather than staying on while held.
`MENU` is the context-menu ("application") key.

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
| Binding/joystick/LED edited | `put` for the one changed profile (debounced 500 ms) |
| New profile created | `put` for the new profile |
| Profile imported from file | `put` for the updated profile |
| Profile deleted | `del` for the removed id; `live` if it was the active profile |
| Set Active pressed | `live` only — no profile data write |
