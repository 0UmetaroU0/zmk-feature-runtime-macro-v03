# Design: Runtime Macro Flexibility Improvements

Status: proposal (no implementation yet)
Audience: implementation agent / contributors
Scope: `zmk-feature-runtime-macro` (with one small, optional API addition to
`zmk-feature-custom-settings`)

## 1. Background and current architecture

Runtime Macro stores per-slot macros in two custom-settings arrays under
subsystem `cormoran__runtime_macro`:

- `names[]` — string array, display name per slot
- `macros[]` — bytes array, versioned binary body per slot (varint opcodes:
  down / up / tap / delay / packed key-tap sequence)
- `tap_ms` — global scalar tap duration

Playback (`src/runtime_macro.c`) reads the body from custom settings, decodes
it into a bounded queue (`CONFIG_ZMK_RUNTIME_MACRO_QUEUE_SIZE`), and replays it
through `zmk_behavior_queue_add` on the low-priority work queue. Editing is
done exclusively through the custom Studio RPC (`proto/…/runtime_macro.proto`)
and the Web UI.

### Current limitations

1. **No compile-time defaults.** All slots start empty. A user flashing a fresh
   board (or after `settings_reset`) has `&rmacro N` keys that do nothing until
   they connect the Web UI. Macros cannot be version-controlled in the user's
   `zmk-config` the way keymaps, combos, and native macros are.
2. **Editing requires a host.** There is no way to (re)define a macro from the
   keyboard itself.
3. **Body size is capped at `CONFIG_ZMK_CUSTOM_SETTINGS_VALUE_MAX_SIZE`**
   (default 64 bytes, and the custom-settings protobuf schema currently caps it
   at 64).
4. **Playback is single-shot and lossy.** A second `&rmacro` press while a
   macro is playing returns `-EBUSY` and is silently dropped (only a log line).
5. **Packed key sequences cover HID usages 0x04–0x38 only** (printable ASCII
   range). F-keys, arrows, and navigation keys always cost a full tap step.
6. **Web UI ergonomics.** Behaviors are addressed by raw local ID; there is no
   whole-config backup/restore.

This document proposes improvements in priority order. **P1 is the concrete,
fully-specified deliverable for the next implementation PR.** P2/P3 items are
sketched so they can become follow-up PRs without re-analysis.

---

## 2. P1 — Compile-time default macros in the keymap (devicetree)

### 2.1 Goals

- Users can define default macros in `.keymap` / `.overlay` files, checked into
  their zmk-config, built into the firmware.
- Defaults behave exactly like "factory values": they are active until the user
  overwrites a slot via RPC, and they come back after *Discard Pending* /
  *Reset* / settings erase.
- Defaults are visible and editable in the Web UI like any other macro (they
  appear as normal steps; editing then saving creates a user value that shadows
  the default).
- No change to the stored binary format and no RPC breakage.

### 2.2 User-facing devicetree API

New binding `cormoran,runtime-macro-default` (one node per slot):

```dts
#include <behaviors.dtsi>
#include <behaviors/runtime_macro.dtsi>

/ {
    runtime_macro_defaults {
        rmacro_email: rmacro_default_0 {
            compatible = "cormoran,runtime-macro-default";
            slot = <0>;
            display-name = "Email";
            text = "user@example.com";
        };

        rmacro_default_1 {
            compatible = "cormoran,runtime-macro-default";
            slot = <1>;
            display-name = "Vim save";
            bindings = <&macro_tap &kp ESC>
                     , <&macro_press &kp LSHFT>
                     , <&macro_tap &kp SEMI>
                     , <&macro_release &kp LSHFT>
                     , <&macro_tap &kp W &kp RET>;
        };

        rmacro_default_2 {
            compatible = "cormoran,runtime-macro-default";
            slot = <2>;
            display-name = "Sign off";
            text = "Best regards,";
            bindings = <&macro_tap &kp RET>;  /* played after `text` */
            wait-ms = <10>;
        };
    };
};
```

Binding file `dts/bindings/cormoran,runtime-macro-default.yaml`:

| property       | type          | required | meaning                                                                  |
| -------------- | ------------- | -------- | ------------------------------------------------------------------------ |
| `slot`         | int           | yes      | target macro index, `0 <= slot < CONFIG_ZMK_RUNTIME_MACRO_COUNT`          |
| `display-name` | string        | no       | default `names[slot]`; falls back to the node name                        |
| `text`         | string        | no       | ASCII text encoded as packed key-tap sequence(s)                          |
| `bindings`     | phandle-array | no       | ZMK-native macro step list (subset, see below)                            |
| `wait-ms`      | int           | no       | delay inserted between consecutive `bindings` steps (default 0 = none)    |

At least one of `text` / `bindings` must be present. When both are present,
`text` is encoded first, then `bindings` (covers the common "type text, press
Enter" case).

`bindings` reuses the native ZMK macro vocabulary so users don't learn a new
syntax. Supported subset:

- `&macro_tap`, `&macro_press`, `&macro_release` — switch the mode applied to
  subsequent behavior entries (→ `TAP` / `DOWN` / `UP` opcodes). Initial mode
  is tap.
- `&macro_wait_time <ms>` — emits a `DELAY` opcode.
- Any other behavior entry (`&kp A`, `&mo 1`, `&bt BT_CLR`, …) — encoded with
  the current mode as `(behavior local id, param1, param2)`.

Unsupported control behaviors (`&macro_tap_time`, `&macro_pause_for_release`,
`&macro_param_1to1` etc.) must fail loudly (see 2.5); tap duration always
comes from the global `tap_ms` setting, consistent with RPC-defined macros.

### 2.3 Encoding pipeline

Compile time (new `src/runtime_macro_dt_defaults.c`):

- `DT_INST_FOREACH_STATUS_OKAY` over `cormoran,runtime-macro-default` extracts
  each node's `bindings` into a `static const struct zmk_behavior_binding[]`
  using the same extraction macros ZMK's keymap/macro code uses, plus the
  `text`, `display-name`, `slot`, `wait-ms` literals.
- `BUILD_ASSERT(slot < CONFIG_ZMK_RUNTIME_MACRO_COUNT)` per node.

Boot time (SYS_INIT, `APPLICATION` level, priority after custom-settings init —
behavior local IDs are available from `POST_KERNEL`):

1. For each DT default, encode into the existing v1 binary format:
   - `text` → packed key-tap sequence opcode(s) via
     `zmk_runtime_macro_pack_key_tap` (reject non-encodable characters).
   - `bindings` → walk the array; classify control behaviors by comparing the
     resolved `behavior_dev` against `DEVICE_DT_GET(DT_NODELABEL(macro_tap))`
     etc.; other entries become `DOWN`/`UP`/`TAP` opcodes with
     `zmk_behavior_get_local_id()` + varint params. Insert `DELAY wait-ms`
     between steps when `wait-ms > 0`.
   - Run consecutive eligible `&kp` taps through the same packing used by the
     Web UI so DT defaults get the 1-byte-per-key compression automatically.
2. Validate with `zmk_runtime_macro_validate_encoded()` and check the size fits
   `CONFIG_ZMK_CUSTOM_SETTINGS_VALUE_MAX_SIZE`.
3. Install the encoded body as the **default value** of `macros[slot]` and the
   display name as the default of `names[slot]` (see 2.4).

Duplicate `slot` values: detect at boot, keep the first, `LOG_ERR` the rest.
(A devicetree-level duplicate check is not practical with DT macros alone.)

### 2.4 Integration with custom-settings — two options

**Option A (recommended): runtime-default API in zmk-feature-custom-settings.**

Add to `zmk-feature-custom-settings`:

```c
/* Replace a setting's default value at runtime (e.g. from devicetree data
 * that needs boot-time resolution). Does not touch persistent/memory values. */
int zmk_custom_setting_set_default(const struct zmk_custom_setting *setting,
                                   const struct zmk_custom_setting_value *value);
```

Registry entries are already mutable (`initialized`, `persistent_value`, …
live in the same struct), so this is a copy into `setting->default_value`
plus type validation. Everything else falls out for free:

- Effective-value resolution already falls back to `default_value`, so reads,
  playback, and RPC `GetMacro` all see the DT default when the user has not
  written the slot.
- *Discard Pending* / `zmk_custom_setting_reset()` / settings erase restore the
  DT default — exactly the desired "factory value" semantics.
- The Web UI needs zero changes to display and edit defaults (they arrive as
  ordinary steps).

Constraint to document on the new API: call it before the value is first read
over RPC or played; the natural place is a SYS_INIT right after custom-settings
initialization. This is a small, backward-compatible addition and should be its
own PR in `zmk-feature-custom-settings` (coordinate with the flexibility
redesign proposed there — the API is compatible with both the current and the
proposed storage layout).

**Option B (fallback, module-local only).** If changing custom-settings is
undesirable: keep the encoded DT defaults in this module, and have
`zmk_runtime_macro_read()` / the play path / the RPC handler fall back to the
DT default when the stored body is empty **and** the slot has no persisted or
unsaved value (checkable via `zmk_custom_setting_find_array_element()` +
`has_persistent_value` / `zmk_custom_setting_has_unsaved_value()`). This works
without cross-repo changes but duplicates fallback logic in three call sites
and makes "empty user macro" vs "default" distinguishable only via extra RPC
plumbing. Choose B only if A is rejected.

### 2.5 Error handling

Encoding failures (unsupported control behavior, non-ASCII `text`, body too
large, unknown behavior) must not brick the build silently:

- `LOG_ERR` with slot number and reason; the slot keeps an empty default.
- Additionally expose the failure to tests: the encoder returns per-slot
  status, and the test build (`CONFIG_ZMK_RUNTIME_MACRO_TEST`) asserts all DT
  defaults encoded successfully, so a bad default fails `west zmk-test`.

### 2.6 RPC / Web UI (optional polish, same PR or follow-up)

- Add `bool is_default = 5;` to `MacroSummary` and `MacroSlot` so the UI can
  badge slots that are showing an unedited compile-time default, and offer a
  "Reset to default" action (maps to `zmk_custom_setting_reset`). Proto field
  additions are backward compatible.
- README: new "Default macros in your keymap" section with the DT example.

### 2.7 Kconfig

No new symbol required — compile `runtime_macro_dt_defaults.c` only when
`DT_HAS_CORMORAN_RUNTIME_MACRO_DEFAULT_ENABLED`. (If a guard is still desired,
`CONFIG_ZMK_RUNTIME_MACRO_DT_DEFAULTS`, default `y`.)

### 2.8 Tests

Per repo rules (`tests/<case>` unit tests + `tests/zmk-config` build test):

- `tests/default-macro-text/` — DT default with `text`, mock a press of
  `&rmacro 0`, snapshot expected key taps.
- `tests/default-macro-bindings/` — `bindings` with press/release/tap/wait,
  verify order and delays.
- `tests/default-macro-override/` — write the slot via
  `zmk_runtime_macro_write(persist=true)` in test init, verify the user value
  shadows the default; then `zmk_custom_setting_reset` and verify the default
  is back.
- `tests/zmk-config` — add a default macro node to the build-test keymap and
  assert in `test.py` that the encoder ran cleanly (no `LOG_ERR`).

### 2.9 Risks

- **Init ordering**: default installation must run after custom-settings
  registry init but is independent of settings load (defaults never override
  loaded values). Verify with the override test above.
- **Availability of `&macro_*` control behaviors**: they come from ZMK's
  `behaviors.dtsi`; if a user keymap doesn't include it, `DT_NODELABEL(...)`
  lookups fail at compile time. Guard with `DT_NODE_EXISTS` and fall back to
  treating every entry as tap-mode (plus `LOG_ERR` at boot for unresolvable
  control behaviors).

---

## 3. P2 — On-device macro recording

QMK-style dynamic macros, no host needed:

- New behavior `&rmacro_rec <slot>` (`cormoran,zmk-behavior-runtime-macro-record`,
  one param). First press starts recording into the slot (memory mode), second
  press — or `CONFIG_ZMK_RUNTIME_MACRO_RECORD_TIMEOUT_MS` of inactivity — stops
  it.
- Record `zmk_keycode_state_changed` events as `DOWN`/`UP` opcodes; insert
  `DELAY` opcodes from real inter-event timing, quantized (e.g. 10 ms) and
  capped (e.g. 2 s) to save bytes. Exclude the recording key itself.
- On stop: run the existing tap-compression (down+up pairs → `TAP`, consecutive
  eligible taps → packed sequence), validate size; if the body exceeds the
  limit, truncate at the last complete step and signal via log/UI flag.
- Persistence: keep in memory mode; user persists via the existing Save
  Pending flow (or `CONFIG_ZMK_RUNTIME_MACRO_RECORD_AUTOSAVE=y`).
- Feedback hook (`recording started/stopped`) as a zmk event so users can wire
  RGB/display indicators.

This is a self-contained follow-up PR; it reuses the existing format, storage,
and playback path unchanged.

## 4. P2 — Larger macro bodies (chunking)

Body limited to 64 bytes today (custom-settings protobuf cap). Two-step plan:

1. Short term: document the limit prominently (README does this partially) and
   surface `max_macro_bytes` in the UI before save (already done).
2. Chunked bodies: `CONFIG_ZMK_RUNTIME_MACRO_BODY_CHUNKS` (default 1); body of
   slot `i` is the concatenation of `macros[i*K .. i*K+K-1]`; write/read/RPC
   iterate chunks. **Caveat:** each array element currently costs significant
   RAM in custom-settings (three inline value copies per entry), so chunking
   multiplies that cost. Defer until the custom-settings memory redesign
   (its PR #9) lands; the chunk mapping above is compatible with it.

## 5. P3 — Playback improvements

- **Don't drop presses silently**: replace the single `active` flag with a
  small pending-slot FIFO (depth 2–4, Kconfig) so pressing `&rmacro` during
  playback queues the next macro instead of returning `-EBUSY`.
- **Repeat**: optional `&rmacro_repeat <slot> <count>` behavior (two cells)
  or press-and-hold-to-repeat; low priority, pure playback-layer change.

## 6. P3 — Format and protocol extensions

- **Extend packed key range** from usage `0x04–0x38` to `0x04–0x7f` (bit 7 is
  the shift flag, so 7 usage bits are already reserved). Adds F-keys, arrows,
  navigation and keypad keys at 1 byte each with no format change. Old
  firmware safely rejects new bodies at write validation, so also add a
  capability field (e.g. `packed_key_max_usage`) to `MacroGlobalSettings` for
  UI feature detection.
- **Per-macro tap duration**: new opcode `SET_TAP_MS = 6` applying to the rest
  of the body. Versioning policy: additive opcodes are gated by a capability
  report in `MacroGlobalSettings` (e.g. `supported_opcodes` bitmask); the
  format `version` byte is bumped only for incompatible layout changes.
  Firmware-side `zmk_runtime_macro_validate_encoded` already rejects unknown
  opcodes at write time, so mismatched UI/firmware fail cleanly.

## 7. P3 — Web UI ergonomics

- Behavior picker backed by ZMK Studio core RPC behavior metadata (friendly
  names instead of raw local IDs; `key_press_behavior_id` already points the
  way).
- Whole-config JSON export/import (all slots + names + `tap_ms`) for backup and
  sharing, complementing the per-macro Keyboard Abyss import/export.
- Default-value badges + "Reset to default" (depends on §2.6).

---

## 8. Suggested implementation order

1. **PR 1 (custom-settings)**: `zmk_custom_setting_set_default()` + unit test.
2. **PR 2 (this repo, P1)**: DT default macros — binding yaml, extraction +
   boot encoder, default installation, tests, README. Include §2.6 proto flag
   if convenient (proto → firmware handler → web UI order per repo rules).
3. **PR 3**: packed-key range extension + capability field (small, independent).
4. **PR 4**: on-device recording (§3).
5. Later: playback FIFO (§5), chunking after custom-settings redesign (§4),
   Web UI ergonomics (§7).
