# SEAM — Serial Enumeration and Action Model

<p align="center">
  <img src="assets/logo/seam-logo.svg" alt="SEAM logo" width="520" />
</p>

**Version**: 5.0.0  
**Encoding**: ASCII / UTF-8  
**Status**: Draft  

## Related

| Repo | Description |
|---|---|
| [seam-host](https://github.com/tarasfilonenko/seam-host) | Generic desktop app for any SEAM device |
| [seam-kit](https://github.com/tarasfilonenko/seam-kit) | SEAM Kit hardware toolkit — an example SEAM implementation |

---

## 1. Overview

SEAM is a lightweight, human-readable protocol that allows any device — microcontroller,
single-board computer, or software process — to describe its capabilities to a host over
a serial connection. The host can then discover what the device can do and interact with
it dynamically, without needing prior knowledge of the device type.

A SEAM device exposes:
- **Parameters** — named values that can be read and/or written; types are expressed as MIME types
- **Actions** — named functions that can be invoked, with typed arguments
- **Streams** — named data channels that emit values continuously

The host queries capabilities once on connect, then uses a small set of commands
(`GET`, `SET`, `DO`, `STATUS`, `WATCH`, `UNWATCH`) to interact with the device. No
host-side configuration or code generation is required to support a new device type.

### Design goals

- **Human-readable** — works with any serial terminal, no binary decoder needed
- **Self-describing** — the device tells the host everything; the host needs no prior knowledge
- **Transport-agnostic** — designed for USB CDC-ACM but works over any byte stream
- **MCU-friendly** — implementable on a microcontroller with minimal RAM and no dynamic allocation
- **Minimal** — one handshake command (`CAPS`), six interaction commands

### Relationship to existing protocols

| Protocol | Self-describing | Human-readable | Actions | Streams |
|---|---|---|---|---|
| Firmata | Pin modes only | No (binary) | No | No |
| Electric UI | Variable list | No (binary) | Callback only | No |
| SEAM | Full: params, actions, streams, groups | **Yes** | **Yes** | **Yes** |

---

## 2. Transport

SEAM operates over any reliable byte stream. The reference transport is **USB CDC-ACM**
(virtual serial port), but SEAM may also be used over:

- Hardware UART
- BLE serial (SPP / NUS)
- TCP socket
- Any other stream that preserves byte order and delivers data intact

Transport-specific considerations:

- **USB CDC-ACM**: baud rate setting is irrelevant; implementations must accept any value
- **UART**: both sides must agree on baud rate out of band
- **TCP**: the connection itself provides framing; no additional wrapper needed

SEAM does not define device discovery or addressing. Those are handled by the transport
or the host application. For USB CDC-ACM, the host enumerates available ports and opens
them individually.

---

## 3. Line Format

All SEAM communication uses text lines. Every line follows this structure:

```
<KEYWORD> [<arg1> [<arg2> ...]] \r\n
```

Rules:

- Lines are terminated with **`\r\n`** (CR LF, 0x0D 0x0A).
- Fields are separated by **one or more spaces**.
- Keywords are **UPPERCASE**.
- IDs (parameter names, action names, stream IDs, group IDs, device types) use
  **lowercase alphanumeric with underscores** only: `[a-z0-9_]+`.
- Lines beginning with `#` are **comments** and must be ignored by the receiver.
- Data payloads are length-prefixed and not subject to line length constraints — see Section 12.

> **Note:** Variable-length content uses either the block format (Section 4), the
> length-prefixed frame format (Section 12), or optional trailing text on `OK` lines.

---

## 4. Block Structure

All SEAM structural declarations follow a universal pattern:

```
KEYWORD BEGIN [id]
key:value
...
KEYWORD END
```

Rules:

- `KEYWORD BEGIN` opens a block, optionally followed by an id on the same line.
- `KEYWORD END` closes the block.
- Block contents consist of `key:value` lines and/or nested blocks.
- Blocks may be nested.
- Unknown `key:value` lines must be silently ignored — forward compatibility.

This pattern is used by `CAPS`, `GROUP`, `PARAM`, `ACTION`, `ARG`, `STREAM`, and `ERR`
blocks, and by the `DO` command block.

---

## 5. Commands (Host → Device)

Each host-issued command must receive exactly one **terminal response** from the device.
Terminal responses are:

- `CAPS BEGIN ... CAPS END`
- `VALUE <id> <length>\r\n<data>\r\n`
- `OK[ <text>]\r\n`
- `ERR BEGIN <code> ... ERR END`

On `OK` responses, any trailing text after `OK ` is optional human-readable detail. Hosts
must not use that text for control flow.

`DATA` frames and `CHANGED` notifications are **asynchronous notifications**, not terminal
responses. They may interleave with command processing and do not satisfy a pending
command. A host must wait for the terminal response to the current command before issuing
the next command.

### 5.1 `CAPS`

Request full capability description. This is the **only handshake command**. CAPS returns
device identity, firmware version, and all declared capabilities. There is no separate
HELLO command.

```
CAPS\r\n
```

Terminal response: a CAPS block (see Section 6).

The device must respond to `CAPS` at any time, including immediately after power-on or
port open. Response must arrive within **500ms**.

---

### 5.2 `GET`

Read a single parameter value.

```
GET <id>\r\n
```

Success response:
```
VALUE <id> <length>\r\n
<data>\r\n
```

Error response:
```
ERR BEGIN UNKNOWN_PARAM
id:<id>
ERR END

ERR BEGIN NOT_READABLE
id:<id>
ERR END
```

Example:
```
>> GET pulse_width_us
<< VALUE pulse_width_us 4
<< 1500

>> GET label
<< VALUE label 7
<< Servo 1

>> GET schematic
<< VALUE schematic 4096
<< <4096 bytes of PNG data>
```

> **Note:** Multi-parameter GET is not supported. Issue one `GET` per parameter.

---

### 5.3 `SET`

Write a single parameter value.

```
SET <id> <length>\r\n
<data>\r\n
```

Success response:
```
OK[ <text>]\r\n
```

Error response:
```
ERR BEGIN UNKNOWN_PARAM
id:<id>
ERR END

ERR BEGIN NOT_WRITABLE
id:<id>
ERR END

ERR BEGIN OUT_OF_RANGE
id:<id>
min:<value>
max:<value>
ERR END

ERR BEGIN INVALID_VALUE
id:<id>
[message:<text>]
ERR END
```

Example:
```
>> SET pulse_width_us 4
>> 1200
<< OK

>> SET display 3072
>> <3072 bytes of JPEG data>
<< OK
```

The device validates incoming data against the declared MIME type. On type mismatch or
parse failure it responds with an `INVALID_VALUE` error block.

> **Note:** Multi-parameter SET is not supported. Issue one `SET` per parameter.

---

### 5.4 `DO`

Invoke a named action. `DO BEGIN` opens the call block; each argument is an `IN` frame
identified by name; `DO END` closes the block and triggers execution.

```
DO BEGIN <id>\r\n
[IN <id> <length>\r\n
<data>\r\n]
DO END\r\n
```

Success response:
```
OK[ <text>]\r\n
```

Error response:
```
ERR BEGIN UNKNOWN_ACTION
id:<id>
ERR END

ERR BEGIN BAD_ARGS
id:<id>
[missing:<arg_id>]
[message:<text>]
ERR END

ERR BEGIN BUSY
[id:<id>]
[message:<text>]
ERR END

ERR BEGIN FAILED
[id:<id>]
[message:<text>]
ERR END
```

Example:
```
>> DO BEGIN center
>> DO END
<< OK

>> DO BEGIN sweep
>> IN start_us 4
>> 1000
>> IN end_us 4
>> 2000
>> DO END
<< OK

>> DO BEGIN load_frame
>> IN frame 3072
>> <3072 bytes of JPEG data>
>> DO END
<< OK
```

Argument data uses the same encoding as `SET` for the declared MIME type. `seam/` scalar
values are represented as ASCII text (e.g. `1000` for a `seam/int`). Arguments may be
sent in any order — the device matches them by name against the ACTION declaration.
All declared arguments are required; omitting any results in a `BAD_ARGS` error block.

---

### 5.5 `STATUS`

Request a human-readable status string. Intended for debugging and display. The content
is free-form and device-defined.

```
STATUS\r\n
```

Success response:
```
OK[ <text>]\r\n
```

For `STATUS`, the optional trailing text is the human-readable status string. Any text
after `OK ` occupies the remainder of the line.

Example:
```
>> STATUS
<< OK Idle. PWM enabled at 1500us, 50Hz.
```

---

### 5.6 `WATCH` / `UNWATCH`

Subscribe to or unsubscribe from change notifications for a watchable parameter. The device
emits a `CHANGED` line asynchronously whenever the value changes — see Section 13.

```
WATCH <id>\r\n
UNWATCH <id>\r\n
```

Success response:
```
OK[ <text>]\r\n
```

Error response:
```
ERR BEGIN UNKNOWN_PARAM
id:<id>
ERR END

ERR BEGIN NOT_WATCHABLE
id:<id>
ERR END

ERR BEGIN ALREADY_WATCHING
id:<id>
ERR END

ERR BEGIN NOT_WATCHING
id:<id>
ERR END
```

`CHANGED` notifications are only emitted when the value actually changes — not on every
`SET` if the value is unchanged. The device does not emit an initial value on `WATCH`;
the host should `GET` the current value immediately after subscribing if needed.

---

## 6. CAPS Block

The CAPS response is a multi-line block bounded by `CAPS BEGIN` and `CAPS END`. It contains
device identity fields followed by one or more group blocks.

### 6.1 Structure

```
CAPS BEGIN
type:<device_type>
name:<display_name>
version:<semver>

GROUP BEGIN <id>
...
GROUP END

[GROUP BEGIN <id>
...
GROUP END]

CAPS END
```

### 6.2 Identity Fields

| Key | Value | Description |
|---|---|---|
| `type` | `[a-z0-9_]+` | Machine-readable device identifier |
| `name` | string | Human-readable device name |
| `version` | semver | Firmware version |

All three are mandatory and must appear before any `GROUP BEGIN`.

---

## 7. GROUP Block

```
GROUP BEGIN <id>
label:<display_label>
[visible:<cel_expression>]
[enabled:<cel_expression>]

PARAM BEGIN ...
ACTION BEGIN ...
STREAM BEGIN ...

GROUP END
```

| Key | Mandatory | Description |
|---|---|---|
| `label` | yes | Human-readable group label |
| `visible` | no | CEL expression; when false, the host should hide this group from normal UI while keeping it in the capability model — see Section 14 |
| `enabled` | no | CEL expression; when false, all items in the group are disabled — see Section 14 |

Groups organise parameters, actions, and streams into logical sections. All `PARAM`,
`ACTION`, and `STREAM` declarations must appear inside a group. Empty groups are valid.

Groups are logical only — they carry no layout hints. Host applications may present
groups in any order they choose.

---

## 8. PARAM Block

```
PARAM BEGIN <id>
type:<mime_type>
access:<r|w|rw>
label:<text>
[description:<text>]
[default:<value>]
[min:<value>]
[max:<value>]
[options:<value> ...]
[flags:<name> ...]
[watchable:true]
[persist:true]
[visible:<cel_expression>]
[enabled:<cel_expression>]
PARAM END
```

### 8.1 Fields

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of the parameter value |
| `access` | yes | `r`, `w`, or `rw` |
| `label` | yes | Short human-readable display name |
| `description` | no | Longer text for tooltip, hint, or footnote |
| `default` | no | Informational default value |
| `min` | no | Minimum value — `seam/int` and `seam/float` only |
| `max` | no | Maximum value — `seam/int` and `seam/float` only |
| `options` | no | Space-separated option list — `seam/enum` only |
| `flags` | no | Space-separated list of flag names — `seam/flags` only; host renders one labeled toggle per name; wire value is space-separated names of the active (true) flags; absent name means false; empty value means no flags set |
| `watchable` | no | `true` — host may subscribe via `WATCH` |
| `persist` | no | `true` — host should save this parameter's value and restore it via `SET` on reconnection; only meaningful on params with `access:rw` or `access:w` |
| `visible` | no | CEL expression; when false, the host should hide this parameter from normal UI while keeping it in the capability model — see Section 14 |
| `enabled` | no | CEL expression; when false, the host should disable interaction with this parameter — see Section 14 |

**Note on defaults:** The `default` field is informational. Hosts should always `GET`
live values after `CAPS` before populating their UI.

### 8.2 Examples

```
PARAM BEGIN pulse_width_us
type:seam/int
access:rw
label:Pulse Width
min:500
max:2500
default:1500
persist:true
description:Pulse width in microseconds
PARAM END

PARAM BEGIN enabled
type:seam/bool
access:rw
label:Enabled
default:true
persist:true
description:Enable PWM output
PARAM END

PARAM BEGIN mode
type:seam/enum
access:rw
label:Mode
default:continuous
options:continuous single sweep
persist:true
description:Operating mode
PARAM END

PARAM BEGIN label
type:seam/string
access:rw
label:Label
default:Servo 1
description:Channel label
PARAM END

PARAM BEGIN schematic
type:image/png
access:r
label:Schematic
watchable:true
description:Module schematic diagram
PARAM END

PARAM BEGIN display
type:image/jpeg
access:w
label:Display
description:Display framebuffer
PARAM END

PARAM BEGIN enabled_channels
type:seam/flags
access:rw
label:Enabled Channels
flags:ch1 ch2 ch3 ch4 ch5 ch6 ch7 ch8 ch9 ch10 ch11 ch12 ch13 ch14 ch15 ch16
default:
persist:true
description:Active servo channels; absent flag means channel is disabled
PARAM END
```

---

## 9. ACTION Block

```
ACTION BEGIN <id>
label:<text>
[description:<text>]
[visible:<cel_expression>]
[enabled:<cel_expression>]
[trigger:<param_id>]

[ARG BEGIN <id>
type:<mime_type>
label:<text>
[description:<text>]
[min:<value>]
[max:<value>]
[options:<value> ...]
ARG END]

ACTION END
```

### 9.1 ACTION Fields

| Key | Mandatory | Description |
|---|---|---|
| `label` | yes | Short human-readable display name |
| `description` | no | Longer text for tooltip, hint, or footnote |
| `visible` | no | CEL expression; when false, the host should hide this action from normal UI while keeping it in the capability model — see Section 14 |
| `enabled` | no | CEL expression; when false, the host should disable interaction with this action — see Section 14 |
| `trigger` | no | ID of the param that drives this action. The host does not render a button; instead it invokes this action programmatically in response to interaction events on the referenced param (e.g. element clicks on an `image/svg+xml` param — see Appendix D) |

### 9.2 ARG Fields

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of the argument value |
| `label` | yes | Short human-readable display name |
| `description` | no | Longer text for tooltip, hint, or footnote |
| `min` | no | Minimum value — `seam/int` and `seam/float` only |
| `max` | no | Maximum value — `seam/int` and `seam/float` only |
| `options` | no | Space-separated option list — `seam/enum` only |

Any MIME type may appear in `type`, including `image/*` and `application/octet-stream`.
If an action produces output, model it as a `PARAM` (readable after the action completes)
or a `STREAM`. See Section 5.4 for the `DO` wire protocol.

### 9.3 Examples

```
ACTION BEGIN center
label:Center
description:Move to center position
visible:mode == "manual"
ACTION END

ACTION BEGIN sweep
label:Sweep
description:Sweep between two pulse widths in microseconds

ARG BEGIN start_us
type:seam/int
label:Start
min:500
max:2500
description:Start pulse width in microseconds
ARG END

ARG BEGIN end_us
type:seam/int
label:End
min:500
max:2500
description:End pulse width in microseconds
ARG END

ACTION END

ACTION BEGIN load_frame
label:Load Frame
description:Load JPEG image into display framebuffer

ARG BEGIN frame
type:image/jpeg
label:Frame
description:Frame data
ARG END

ACTION END
```

---

## 10. STREAM Block

```
STREAM BEGIN <id>
type:<mime_type>
label:<text>
[description:<text>]
[enabled:<cel_expression>]
STREAM END
```

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of emitted values |
| `label` | yes | Short human-readable display name |
| `description` | no | Longer text for tooltip, hint, or footnote |
| `enabled` | no | CEL expression; when false, the host should not display or interact with this stream — see Section 14 |

Any MIME type may be used, including `image/*` and `application/octet-stream`.

### 10.1 Examples

```
STREAM BEGIN position
type:seam/float
label:Position
description:Live position feedback in microseconds
STREAM END

STREAM BEGIN capture
type:application/octet-stream
label:Capture
description:Raw capture data
STREAM END
```

---

## 11. MIME Type System

The `type` field in `PARAM`, `ACTION`, and `STREAM` blocks uses MIME types throughout.

### 11.1 SEAM Informal Namespace

SEAM defines a `seam/` informal namespace for primitive scalar types. These are not
registered MIME types — they are SEAM-internal.

| Type | Description |
|---|---|
| `seam/int` | Integer, represented as decimal ASCII |
| `seam/float` | Floating point, represented as decimal ASCII |
| `seam/bool` | Boolean — ASCII `true` or `false` |
| `seam/string` | UTF-8 string |
| `seam/enum` | One of a declared set of string options |
| `seam/flags` | Named flag set — space-separated active flag names; all declared names via `flags:` — see Section 8.1 |

For CEL evaluation of `enabled:` expressions, hosts should expose standard SEAM scalar types as follows:
- `seam/int` as a CEL integer
- `seam/float` as a CEL double
- `seam/bool` as a CEL boolean
- `seam/string` as a CEL string
- `seam/enum` as a CEL string
- `seam/flags` as a CEL list of strings containing the active flag names, in wire order

This CEL mapping is host-side only. The wire representation remains unchanged.

### 11.2 Standard MIME Types

Any valid MIME type may be used. Common examples:

| Type | Description |
|---|---|
| `image/png` | PNG image |
| `image/jpeg` | JPEG image |
| `image/bmp` | BMP image |
| `image/svg+xml` | SVG vector image; elements with `id` attributes are addressable for click events — see Appendix D |
| `application/octet-stream` | Raw binary data |

This list is not exhaustive.

---

## 12. Frame Format

All variable-length data exchange — `GET` responses, `SET` commands, `DO` arguments, and
stream `DATA` — uses a single frame format:

```
KEYWORD <id> <length>\r\n
<data>\r\n
```

- `KEYWORD` — one of `VALUE`, `SET`, `DATA`, `IN`
- `<id>` — parameter, argument, or stream identifier
- `<length>` — byte count of `<data>`; the trailing `\r\n` is not counted
- `<data>` — exactly `<length>` bytes

`IN` frames appear inside `DO BEGIN` / `DO END` blocks (see Section 5.4). The length
field defines the data boundary — no closing marker is needed.

| Frame | Direction | Context |
|---|---|---|
| `VALUE` | device → host | `GET` response |
| `SET` | host → device | write a parameter value |
| `DATA` | device → host | active stream frame |
| `IN` | host → device | argument inside a `DO` block |

### 12.1 VALUE — GET Response

```
>> GET pulse_width_us
<< VALUE pulse_width_us 4
<< 1500

>> GET label
<< VALUE label 7
<< Servo 1

>> GET schematic
<< VALUE schematic 4096
<< <4096 bytes of PNG data>
```

### 12.2 SET — Write Command

```
>> SET pulse_width_us 4
>> 1200
<< OK

>> SET display 3072
>> <3072 bytes of JPEG data>
<< OK
```

### 12.3 DATA — Stream Frames

The device emits `DATA` frames asynchronously for any active stream. Stream lifecycle is
controlled entirely by the device — typically via `ACTION` invocations declared in CAPS,
or automatically on connect for always-on sensors:

```
DATA position 6
1523.5

DATA capture 512
<512 raw bytes>
```

Multiple streams may be active simultaneously. `DATA` frames from different streams
interleave freely; the `<id>` tag on each frame identifies the source.

---

## 13. CHANGED Notification

For parameters declared `watchable:true` and subscribed via `WATCH`, the device emits a
`CHANGED` line asynchronously whenever the value changes:

```
CHANGED <id>\r\n
```

The host must issue `GET <id>` to retrieve the new value.

`CHANGED` lines may interleave freely with `DATA` frames from active streams.

Example:
```
CHANGED schematic
CHANGED pulse_width_us
```

> **Change from 1.0.0:** The 1.0.0 `CHANGED` format included the new value inline:
> `CHANGED <id> <value>`. In 2.0.0, `CHANGED` emits the id only, and the host retrieves
> the value with a subsequent `GET`. This change enables uniform handling of all MIME
> types — a `CHANGED` line carrying an image payload would break the text line model.
> The trade-off is one extra `GET` round-trip per notification for scalar parameters.

---

## 14. Conditional Expressions (`enabled:` and `visible:`)

The optional `enabled:` field applies to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks:

```
[enabled:<cel_expression>]
```

| Key | Mandatory | Description |
|---|---|---|
| `enabled` | no | A CEL expression evaluated against current parameter values. When false, the host should prevent interaction with this item. Omitting `enabled` is equivalent to `enabled:true`. |

**Semantics:**

- The expression is a [CEL (Common Expression Language)](https://cel.dev) string evaluated by the host, with current parameter values as context, keyed by param id. See the [CEL language definition](https://github.com/google/cel-spec/blob/master/doc/langdef.md) for supported operators and syntax.
- Hosts should convert current parameter values to CEL values by SEAM type before evaluation. Standard SEAM types map as follows: `seam/int` -> CEL integer, `seam/float` -> CEL double, `seam/bool` -> CEL boolean, `seam/string` -> CEL string, `seam/enum` -> CEL string, and `seam/flags` -> CEL list of strings containing the active flag names in wire order. This mapping is host-side only and does not change the wire format.
- When a `GROUP` `enabled:` evaluates to false, all items within the group are considered disabled regardless of their own `enabled:` expressions.
- When `enabled:` is false, the host should not send `GET`, `SET`, or `DO` commands targeting the item.
- The device may return a `BUSY` or `FAILED` error block if a command is received for a disabled item, but correct host behaviour makes this unnecessary.
- The host is responsible for keeping `enabled:` expressions up to date. Strategies such as re-evaluating on `CHANGED` notifications, polling, or a combination of both are all valid — the protocol does not mandate a specific approach.

> **Recommendation:** Parameters referenced in `enabled:` expressions should be declared `watchable:true` to allow the host to react promptly to value changes. This is not enforced by the protocol.

**Python host implementation:** The `cel-python` package (`pip install cel-python`) provides a pure Python CEL evaluator suitable for the host application.

The optional `visible:` field applies to `GROUP`, `PARAM`, and `ACTION` blocks:

```
[visible:<cel_expression>]
```

| Key | Mandatory | Description |
|---|---|---|
| `visible` | no | A CEL expression evaluated against current parameter values. When false, the host should hide this item from normal UI. Omitting `visible` is equivalent to `visible:true`. |

**Semantics:**

- `visible:` uses the same CEL evaluation context and SEAM-to-CEL type mapping as `enabled:`.
- When a `GROUP` `visible:` evaluates to false, the host should omit that group and all items within it from its normal operator-facing UI regardless of the contained items' own `visible:` expressions.
- When a `PARAM` `visible:` evaluates to false, the host should omit that parameter from its normal operator-facing UI.
- When an `ACTION` `visible:` evaluates to false, the host should omit its explicit control from its normal operator-facing UI.
- `visible:` does not change wire-level accessibility. Hidden groups, parameters, and actions remain part of the declared model, and hosts may still `GET`, `SET`, `WATCH`, persist, reference hidden parameters from other CEL expressions, or `DO` hidden actions as appropriate.
- Hosts may optionally surface hidden groups, parameters, and actions in advanced, diagnostic, or developer-oriented views.
- `visible:` is a presentation hint, not a security boundary or capability gate. Use `enabled:` when interaction must be suppressed.

> **Recommendation:** Parameters referenced in `visible:` expressions should be declared `watchable:true` when practical, for the same reason as `enabled:`.

**Examples:**

```
GROUP BEGIN channels
label:Channels
enabled:"ch1" in enabled_channels

PARAM BEGIN enabled_channels
type:seam/flags
access:rw
label:Enabled Channels
flags:ch1 ch2 ch3 ch4
description:Visible channels
PARAM END

GROUP END
```

```
GROUP BEGIN sweep
label:Sweep Control
enabled:mode == "sweep"

PARAM BEGIN start_us
type:deco/int
access:rw
label:Start
min:500
max:2500
description:Sweep start in microseconds
PARAM END

PARAM BEGIN end_us
type:deco/int
access:rw
label:End
min:500
max:2500
enabled:start_us < end_us
description:Sweep end in microseconds
PARAM END

GROUP END
```

```
GROUP BEGIN tuning
label:Tuning
visible:show_advanced

PARAM BEGIN show_advanced
type:seam/bool
access:rw
label:Show Advanced
default:false
watchable:true
description:Reveal advanced tuning controls
PARAM END

PARAM BEGIN pid_derivative
type:seam/float
access:rw
label:D Gain
default:0.0
description:Derivative gain for advanced tuning
PARAM END

GROUP END
```

---

## 15. Error Codes

All failed commands terminate with an `ERR` block:

```text
ERR BEGIN <code>
[id:<target_id>]
[message:<text>]
... code-specific fields ...
ERR END
```

The opening line carries the error code. The block body may contain:

- `id:` — target parameter, action, or watch subscription when applicable
- `message:` — optional human-readable detail for logs or UI
- additional code-specific fields defined below

Hosts must branch on the error code and any known fields defined for that code. Unknown
fields must be ignored. `message:` is advisory only and must not be used for control flow.

| Code | Fields | Meaning |
|---|---|---|
| `UNKNOWN_CMD` | `[message]` | Unrecognized command keyword |
| `UNKNOWN_PARAM` | `id` | No parameter with that ID |
| `UNKNOWN_ACTION` | `id` | No action with that ID |
| `NOT_READABLE` | `id` | Parameter is write-only |
| `NOT_WRITABLE` | `id` | Parameter is read-only |
| `OUT_OF_RANGE` | `id`, `min`, `max` | Numeric value outside declared min/max |
| `INVALID_VALUE` | `id`, `[message]` | Value cannot be parsed for the declared type |
| `BAD_ARGS` | `id`, `[missing]`, `[message]` | Unknown argument name, missing required argument, or value invalid for declared type |
| `BUSY` | `[id]`, `[message]` | Device cannot accept this command right now |
| `FAILED` | `[id]`, `[message]` | Action attempted but failed |
| `NOT_WATCHABLE` | `id` | `WATCH` issued for a param not declared `watchable:true` |
| `ALREADY_WATCHING` | `id` | `WATCH` issued while already subscribed |
| `NOT_WATCHING` | `id` | `UNWATCH` issued while not subscribed |

Examples:

```text
ERR BEGIN UNKNOWN_PARAM
id:pulse_width_us
ERR END
```

```text
ERR BEGIN OUT_OF_RANGE
id:pulse_width_us
min:500
max:2500
message:Value must be between 500 and 2500
ERR END
```

```text
ERR BEGIN BAD_ARGS
id:sweep
missing:end_us
ERR END
```

---

## 16. USB CDC-ACM Identification Convention

When a SEAM device uses USB CDC-ACM as its transport, it should set the USB manufacturer
string to `"SEAM"`. This allows host applications and watcher scripts to identify SEAM
devices by scanning serial ports without opening them.

```c
#define USB_MANUFACTURER  "SEAM"           // marks this as a SEAM device
#define USB_PRODUCT       "Servo Tester"   // human-readable, should match name in CAPS
```

This convention is optional — SEAM works over any transport and the protocol itself has
no dependency on USB descriptor strings. However, implementing it enables:

- Automatic device discovery by SEAM Host and similar tools
- Filtering SEAM ports from unrelated serial devices
- Instant display of a human-readable name before CAPS is received

The `USB_PRODUCT` string should match the `name` field in CAPS exactly.

---

## 17. Device Implementation Requirements

A conforming SEAM device must:

- Respond to `CAPS\r\n` within **500ms**, at any time after the connection is opened
- Return exactly one terminal response for every command: a `CAPS` block, `VALUE`,
  `OK[ <text>]`, or `ERR`
- Return an `UNKNOWN_CMD` error block for any unrecognized command — never hang or ignore
- Never send unsolicited output except `DATA` frames and `CHANGED` lines for active
  `WATCH` subscriptions
- Reset internal streaming and watch state cleanly when the connection is closed and reopened
- Accept any baud rate if using USB CDC-ACM (baud rate is irrelevant for CDC-ACM)

A conforming SEAM device should:

- Respond to commands within **100ms** under normal conditions
- Declare sensible `default` values in CAPS where applicable
- Use descriptive IDs and descriptions to aid human understanding

---

## 18. Host Implementation Requirements

A conforming SEAM host must:

- Send `CAPS` immediately after opening a connection
- Perform a `GET` sweep of all readable parameters after `CAPS` to obtain live values
- Parse `OK` responses with or without optional trailing text
- Parse the frame format: `KEYWORD <id> <length>\r\n<data>\r\n` for `VALUE` and `DATA`
- Handle `ERR` blocks gracefully without crashing
- Issue `GET <id>` upon receiving a `CHANGED <id>` notification to retrieve the new value
- Not send a new command until the previous command's terminal response has been received
- Continue parsing asynchronous `DATA` frames and `CHANGED` notifications while waiting
  for a terminal response; they do not satisfy the pending command
- Evaluate all `enabled:` CEL expressions after the initial `GET` sweep and after each
  `CHANGED` notification, and suppress interaction with any item whose expression
  evaluates to false
- Evaluate all `visible:` CEL expressions for groups, params, and actions after the
  initial `GET` sweep and after each `CHANGED` notification, and hide items whose
  expressions evaluate to false from normal UI
- For each param declared `persist:true`, save its value whenever it is read (via `GET`
  or `CHANGED`) and restore it via `SET` after the initial `GET` sweep on reconnection

---

## 19. Versioning

This specification follows [Semantic Versioning](https://semver.org/).

- **MAJOR**: breaking changes to the wire format or command semantics
- **MINOR**: new commands, new block keys, backward-compatible additions
- **PATCH**: clarifications, corrections, no wire format change

The `version` field in a CAPS response reflects the **device firmware version**, not
the SEAM protocol version. Protocol version is tracked in this document.

### Changelog

**5.0.0**
- Error wire format changed from single-line `ERR ...` responses to structured
  `ERR BEGIN <code>` / `ERR END` blocks
- Error details such as `id:`, `message:`, `min:`, `max:`, and `missing:` are now carried
  as block fields instead of positional line arguments
- Success responses unified as `OK[ <text>]`; `STATUS` now uses the same `OK` form with
  optional trailing human-readable text

**4.6.0**
- Command/response model clarified: every host-issued command receives exactly one
  terminal response
- Terminal responses explicitly defined as `CAPS` block, `VALUE`, `OK`, `OK <status>`,
  or `ERR`
- `DATA` frames and `CHANGED` notifications explicitly defined as asynchronous
  notifications that may interleave but do not satisfy a pending command
- Command sections now consistently distinguish success responses from error responses

**4.5.0**
- `visible:<cel_expression>` optional field added to `ACTION` blocks
- When an `ACTION` `visible:` evaluates to false, the host hides its explicit control from normal UI
- `visible:` remains presentation-only and does not change wire-level accessibility

**4.4.0**
- `visible:<cel_expression>` optional field added to `GROUP` blocks
- When a `GROUP` `visible:` evaluates to false, the host hides the group and all contained items from normal UI regardless of the contained items' own `visible:` expressions
- `visible:` remains presentation-only and does not change wire-level accessibility

**4.3.0**
- `visible:<cel_expression>` optional field added to `PARAM` blocks
- `visible:` is a host-side presentation hint only: when false, the parameter is hidden from normal UI but remains part of the capability model and wire protocol
- Host must re-evaluate `visible:` expressions after the initial `GET` sweep and after each `CHANGED` notification

**4.2.0**
- `persist:true` optional field added to `PARAM` blocks
- When declared, the host saves the parameter's value on each read and restores it via `SET` after the initial `GET` sweep on reconnection
- Only meaningful on params with `access:rw` or `access:w`

**4.1.0**
- `trigger:<param_id>` optional field added to `ACTION` blocks
- An `ACTION` with `trigger:` is a host-invoked event handler: the host does not render a button for it but invokes it programmatically in response to interaction events on the referenced param (e.g. element clicks on an `image/svg+xml` param — see Appendix D)
- `image/svg+xml` added to the standard MIME type table

**4.0.0**
- `STREAM <id> START` and `STREAM <id> STOP` commands removed — stream lifecycle is now entirely the device's responsibility
- Devices that want host-controlled stream start/stop declare `ACTION` blocks for that purpose; the host invokes them via `DO` like any other action
- Devices may begin emitting `DATA` frames at any time after the CAPS exchange without any prior command from the host (auto-streaming)
- Removed error codes: `UNKNOWN_STREAM`, `ALREADY_STREAMING`, `NOT_STREAMING`
- Host must not send `GET`, `SET`, or `DO` commands targeting a disabled item (was `GET`, `SET`, `DO`, or `STREAM`)

**3.0.0**
- `label:` replaces `description:` as the mandatory short display string on `PARAM`, `ACTION`, `STREAM`, and `ARG` blocks — consistent with `GROUP` which has always used `label:`
- `description:` is now optional on all block types; intended for tooltip, hint, or footnote text
- `enabled:<cel_expression>` optional field added to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks
- CEL (Common Expression Language) used for conditional enabling; the host evaluates expressions against current parameter values keyed by param id
- When a `GROUP` `enabled:` is false, all items in the group are treated as disabled regardless of their own `enabled:` expressions
- Host must re-evaluate `enabled:` expressions after the initial `GET` sweep and after each `CHANGED` notification

**2.0.0**
- Universal block structure: `KEYWORD BEGIN [id]` / `KEYWORD END` throughout
- CAPS, GROUP, PARAM, ACTION, STREAM all follow the same block pattern
- Identity fields use `key:value` format (`type`, `name`, `version`), replacing `TYPE`, `NAME`, `VERSION` keyword lines
- MIME type system replaces primitive type names; `seam/` informal namespace for scalar primitives
- `image/*` and arbitrary MIME types supported in `PARAM`, `ACTION` args, and `STREAM`
- Unified frame format: `KEYWORD <id> <length>\r\n<data>\r\n` for `VALUE`, `SET`, `DATA`, `ARG`
- `VALUE <id> <length>` replaces `OK <value>` for GET responses
- ACTION arguments declared as named `ARG BEGIN` / `ARG END` declaration blocks with `type`, `description`, and optional `min`/`max`/`options`
- `DO` uses block format: `DO BEGIN <id>` / `IN <id> <length>` / `DO END`; args matched by name, any order, all required
- ACTION `returns` removed — output modelled as `PARAM` or `STREAM`
- `CHANGED <id>` emits id only — host issues `GET` to retrieve value (was `CHANGED <id> <value>` in 1.0.0)
- `watchable:true` replaces bare `watchable` flag
- `STREAM_BEGIN`/`STREAM_END` binary framing removed — all streams use `DATA` frames
- Old inline `PARAM`, `ACTION`, `STREAM`, `GROUP` declaration formats removed
- Multi-parameter `GET` and `SET` removed
- Line length restriction removed
- Empty `GROUP` blocks are valid

**1.0.0** — Initial release.

---

## Appendix A — Minimal CAPS Example

The smallest valid SEAM CAPS response:

```
CAPS BEGIN
type:temperature_sensor
name:Temperature Sensor
version:1.0.0

GROUP BEGIN readings
label:Readings

PARAM BEGIN temp_c
type:seam/float
access:r
label:Temperature
description:Temperature in Celsius
PARAM END

STREAM BEGIN temp
type:seam/float
label:Temperature
description:Live temperature stream
STREAM END

GROUP END

CAPS END
```

---

## Appendix B — Complete CAPS Example (Servo Tester)

```
CAPS BEGIN
type:servo_tester
name:Servo Tester
version:2.0.0

GROUP BEGIN pwm
label:PWM Control

PARAM BEGIN pulse_width_us
type:seam/int
access:rw
label:Pulse Width
min:500
max:2500
default:1500
description:Pulse width in microseconds
PARAM END

PARAM BEGIN frequency_hz
type:seam/int
access:rw
label:Frequency
min:50
max:400
default:50
description:PWM frequency in Hz
PARAM END

PARAM BEGIN enabled
type:seam/bool
access:rw
label:Enabled
default:true
description:Enable PWM output
PARAM END

PARAM BEGIN mode
type:seam/enum
access:rw
label:Mode
default:continuous
options:continuous single sweep
description:Operating mode
PARAM END

PARAM BEGIN schematic
type:image/png
access:r
label:Schematic
watchable:true
description:Module schematic diagram
PARAM END

PARAM BEGIN display
type:image/jpeg
access:w
label:Display
description:Display framebuffer
PARAM END

ACTION BEGIN center
label:Center
description:Move to center position
ACTION END

ACTION BEGIN sweep
label:Sweep
description:Sweep between two pulse widths in microseconds

ARG BEGIN start_us
type:seam/int
label:Start
min:500
max:2500
description:Start pulse width in microseconds
ARG END

ARG BEGIN end_us
type:seam/int
label:End
min:500
max:2500
description:End pulse width in microseconds
ARG END

ACTION END

STREAM BEGIN position
type:seam/float
label:Position
description:Live position feedback in microseconds
STREAM END

ACTION BEGIN start_position
label:Start Position Stream
description:Begin streaming live position feedback
ACTION END

ACTION BEGIN stop_position
label:Stop Position Stream
description:Stop streaming live position feedback
ACTION END

GROUP END

GROUP BEGIN info
label:Module Info

PARAM BEGIN label
type:seam/string
access:rw
label:Label
default:Servo 1
description:Channel label
PARAM END

PARAM BEGIN uptime_s
type:seam/int
access:r
label:Uptime
description:Uptime in seconds
PARAM END

GROUP END

CAPS END
```

---

## Appendix C — Example Session (Servo Tester)

```
>> CAPS
<< CAPS BEGIN
<< type:servo_tester
<< name:Servo Tester
<< version:2.0.0
<<
<< GROUP BEGIN pwm
<< label:PWM Control
<<
<< PARAM BEGIN pulse_width_us
<< type:seam/int
<< access:rw
<< label:Pulse Width
<< min:500
<< max:2500
<< default:1500
<< description:Pulse width in microseconds
<< PARAM END
<<
<< ...
<<
<< CAPS END

>> GET pulse_width_us
<< VALUE pulse_width_us 4
<< 1500

>> SET pulse_width_us 4
>> 1200
<< OK

>> DO BEGIN center
>> DO END
<< OK

>> DO BEGIN sweep
>> IN start_us 4
>> 1000
>> IN end_us 4
>> 2000
>> DO END
<< OK

>> WATCH schematic
<< OK

<< CHANGED schematic

>> GET schematic
<< VALUE schematic 4096
<< <4096 bytes of PNG data>

>> DO BEGIN start_position
>> DO END
<< OK
<< DATA position 6
<< 1487.3
<< DATA position 6
<< 1488.1

>> DO BEGIN stop_position
>> DO END
<< OK

>> SET display 3072
>> <3072 bytes of JPEG data>
<< OK

>> STATUS
<< OK PWM enabled at 1200us, 50Hz, continuous mode.
```

---

## Appendix D — Interactive SVG Pattern

A module can expose an `image/svg+xml` parameter and receive click events from the host
through a paired ACTION. This section describes the full pattern.

### D.1 Concept

The device exposes a readable SVG parameter. The SVG markup must include `id` attributes
on all interactive elements — the host uses these IDs as stable identifiers for user
interactions. When the user clicks an element in the rendered SVG, the host invokes the
linked action with data from the click event.

```
device exposes SVG → host renders it → user clicks element → host calls DO with event data → device acts
```

### D.2 Event Schema for `image/svg+xml`

A click on an `image/svg+xml` param produces the following event data:

| Arg name | Type | Description |
|---|---|---|
| `source` | `seam/string` | ID of the param that triggered the event (e.g. `diagram`) |
| `element_id` | `seam/string` | `id` attribute of the clicked SVG element; empty string if the element has no `id` |
| `x` | `seam/float` | Click X coordinate in SVG user units |
| `y` | `seam/float` | Click Y coordinate in SVG user units |

The ACTION declares whichever args it needs, using these exact names. The host sends
only the args that are declared — acting as a subset selector over the schema. If the
ACTION declares an arg name not in this table, the host will not send it; how the device
handles the absent argument is its own concern.

### D.3 CAPS Declaration

The ACTION links to its trigger param via `trigger:` and declares whichever schema args
the device needs:

```
GROUP BEGIN ui
label:Interactive Diagram

PARAM BEGIN diagram
type:image/svg+xml
access:r
label:Diagram
watchable:true
description:Interactive module diagram; element ids map to controls
PARAM END

# device only needs to know which element was clicked
ACTION BEGIN on_svg_click
label:SVG Click
trigger:diagram

ARG BEGIN source
type:seam/string
label:Source
ARG END

ARG BEGIN element_id
type:seam/string
label:Element ID
ARG END

ACTION END

GROUP END
```

To also receive click coordinates, add the remaining schema args:

```
ACTION BEGIN on_svg_click
label:SVG Click
trigger:diagram

ARG BEGIN source
type:seam/string
label:Source
ARG END

ARG BEGIN element_id
type:seam/string
label:Element ID
ARG END

ARG BEGIN x
type:seam/float
label:X
ARG END

ARG BEGIN y
type:seam/float
label:Y
ARG END

ACTION END
```

### D.4 SVG Structure Requirements

The SVG content returned by `GET diagram` must include `id` attributes on every element
the host should treat as interactive. Non-interactive elements should have no `id`.

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 100">
  <!-- clickable elements carry id attributes -->
  <rect id="relay_k1" x="10" y="10" width="50" height="30"/>
  <rect id="relay_k2" x="70" y="10" width="50" height="30"/>
  <circle id="led_power" cx="160" cy="25" r="10"/>
  <!-- decorative elements have no id -->
  <text x="10" y="80">K1</text>
  <text x="70" y="80">K2</text>
</svg>
```

The host should add pointer-cursor styling and click handlers only to elements that carry
an `id`. All other elements should remain inert.

### D.5 Host Responsibilities

1. Issue `GET diagram` after `CAPS` and render the SVG.
2. Scan all ACTIONs for `trigger:diagram`; build a list of linked actions.
3. Attach click handlers to all SVG elements that have an `id` attribute.
4. On click, invoke each linked action via `DO`, sending only the args declared by that action.
5. If `diagram` is `watchable:true`, subscribe via `WATCH` and re-fetch on `CHANGED` to
   keep the rendered SVG in sync (the device may update element styles to reflect state).

### D.6 Wire Protocol Example

```
>> GET diagram
<< VALUE diagram 374
<< <374 bytes of SVG markup>

# user clicks the element with id="relay_k1"

>> DO BEGIN on_svg_click
>> IN source 7
>> diagram
>> IN element_id 8
>> relay_k1
>> DO END
<< OK
```

With coordinates declared:

```
>> DO BEGIN on_svg_click
>> IN source 7
>> diagram
>> IN element_id 8
>> relay_k1
>> IN x 4
>> 34.5
>> IN y 4
>> 22.0
>> DO END
<< OK
```

### D.7 Returning Updated State

If the device changes the SVG after a click (e.g. toggling a relay changes element fill
colour), it emits `CHANGED diagram`. The host re-fetches and re-renders:

```
>> WATCH diagram
<< OK

>> DO BEGIN on_svg_click
>> IN element_id 8
>> relay_k1
>> DO END
<< OK
<< CHANGED diagram

>> GET diagram
<< VALUE diagram 374
<< <374 bytes of updated SVG markup>
```
