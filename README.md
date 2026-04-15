# SEAM — Serial Enumeration and Action Model

**Version**: 4.1.0  
**Encoding**: ASCII / UTF-8  
**Status**: Draft  

## Related

| Repo | Description |
|---|---|
| [seam-host](https://github.com/tarasfilonenko/seam-host) | Generic desktop app for any SEAM device |
| [modbench](https://github.com/tarasfilonenko/modbench) | ModBench hardware toolkit — an example SEAM implementation |

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
(`GET`, `SET`, `DO`) to interact with the device. No host-side configuration
or code generation is required to support a new device type.

### Design goals

- **Human-readable** — works with any serial terminal, no binary decoder needed
- **Self-describing** — the device tells the host everything; the host needs no prior knowledge
- **Transport-agnostic** — designed for USB CDC-ACM but works over any byte stream
- **MCU-friendly** — implementable on a microcontroller with minimal RAM and no dynamic allocation
- **Minimal** — one handshake command (`CAPS`), five interaction commands

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

> **Note:** Quoted strings appear only in `STATUS` responses (Section 5.5). All other
> variable-length content uses the length-prefixed frame format (Section 12).

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

This pattern is used by `CAPS`, `GROUP`, `PARAM`, `ACTION`, `ARG`, and `STREAM` blocks,
and by the `DO` command block.

---

## 5. Commands (Host → Device)

### 5.1 `CAPS`

Request full capability description. This is the **only handshake command**. CAPS returns
device identity, firmware version, and all declared capabilities. There is no separate
HELLO command.

```
CAPS\r\n
```

Response: a CAPS block (see Section 6).

The device must respond to `CAPS` at any time, including immediately after power-on or
port open. Response must arrive within **500ms**.

---

### 5.2 `GET`

Read a single parameter value.

```
GET <id>\r\n
```

Response:
```
VALUE <id> <length>\r\n
<data>\r\n
```

Errors:
```
ERR UNKNOWN_PARAM <id>\r\n
ERR NOT_READABLE <id>\r\n
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

Response:
```
OK\r\n
```

Errors:
```
ERR UNKNOWN_PARAM <id>\r\n
ERR NOT_WRITABLE <id>\r\n
ERR OUT_OF_RANGE <id> <min> <max>\r\n
ERR INVALID_VALUE <id>\r\n
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
parse failure it responds with `ERR INVALID_VALUE <id>`.

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

Response:
```
OK\r\n
```

Errors:
```
ERR UNKNOWN_ACTION <id>\r\n
ERR BAD_ARGS <id>\r\n
ERR BUSY\r\n
ERR FAILED "<message>"\r\n
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
All declared arguments are required; omitting any results in `ERR BAD_ARGS`.

---

### 5.5 `STATUS`

Request a human-readable status string. Intended for debugging and display. The content
is free-form and device-defined.

```
STATUS\r\n
```

Response:
```
OK "<status_string>"\r\n
```

Example:
```
>> STATUS
<< OK "Idle. PWM enabled at 1500us, 50Hz."
```

---

### 5.6 `WATCH` / `UNWATCH`

Subscribe to or unsubscribe from change notifications for a watchable parameter. The device
emits a `CHANGED` line asynchronously whenever the value changes — see Section 13.

```
WATCH <id>\r\n
UNWATCH <id>\r\n
```

Response:
```
OK\r\n
```

Errors:
```
ERR UNKNOWN_PARAM <id>\r\n
ERR NOT_WATCHABLE <id>\r\n
ERR ALREADY_WATCHING <id>\r\n
ERR NOT_WATCHING <id>\r\n
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
[enabled:<cel_expression>]
[visible:<cel_expression>]

PARAM BEGIN ...
ACTION BEGIN ...
STREAM BEGIN ...

GROUP END
```

| Key | Mandatory | Description |
|---|---|---|
| `label` | yes | Human-readable group label |
| `enabled` | no | CEL expression; when false, all items in the group are disabled — see Section 14 |
| `visible` | no | CEL expression; when false, the host should not render this group or any of its items — see Section 15 |

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
[watchable:true]
[enabled:<cel_expression>]
[visible:<cel_expression>]
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
| `watchable` | no | `true` — host may subscribe via `WATCH` |
| `enabled` | no | CEL expression; when false, the host should disable interaction with this parameter — see Section 14 |
| `visible` | no | CEL expression; when false, the host should not render this parameter — see Section 15 |

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
description:Pulse width in microseconds
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
```

---

## 9. ACTION Block

```
ACTION BEGIN <id>
label:<text>
[description:<text>]
[enabled:<cel_expression>]
[visible:<cel_expression>]

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
| `enabled` | no | CEL expression; when false, the host should disable interaction with this action — see Section 14 |
| `visible` | no | CEL expression; when false, the host should not render this action — see Section 15 |

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
[visible:<cel_expression>]
STREAM END
```

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of emitted values |
| `label` | yes | Short human-readable display name |
| `description` | no | Longer text for tooltip, hint, or footnote |
| `enabled` | no | CEL expression; when false, the host should not display or interact with this stream — see Section 14 |
| `visible` | no | CEL expression; when false, the host should not render this stream — see Section 15 |

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

## 14. Conditional Enabling (`enabled:`)

The optional `enabled:` field applies to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks:

```
[enabled:<cel_expression>]
```

| Key | Mandatory | Description |
|---|---|---|
| `enabled` | no | A CEL expression evaluated against current parameter values. When false, the host should prevent interaction with this item. Omitting `enabled` is equivalent to `enabled:true`. |

**Semantics:**

- The expression is a [CEL (Common Expression Language)](https://cel.dev) string evaluated by the host, with current parameter values as context, keyed by param id. See the [CEL language definition](https://github.com/google/cel-spec/blob/master/doc/langdef.md) for supported operators and syntax.
- When a `GROUP` `enabled:` evaluates to false, all items within the group are considered disabled regardless of their own `enabled:` expressions.
- When `enabled:` is false, the host should not send `GET`, `SET`, or `DO` commands targeting the item.
- The device may return `ERR BUSY` or `ERR FAILED` if a command is received for a disabled item, but correct host behaviour makes this unnecessary.
- The host is responsible for keeping `enabled:` expressions up to date. Strategies such as re-evaluating on `CHANGED` notifications, polling, or a combination of both are all valid — the protocol does not mandate a specific approach.

> **Recommendation:** Parameters referenced in `enabled:` expressions should be declared `watchable:true` to allow the host to react promptly to value changes. This is not enforced by the protocol.

**Python host implementation:** The `cel-python` package (`pip install cel-python`) provides a pure Python CEL evaluator suitable for the host application.

---

## 15. Visibility (`visible:`)

The optional `visible:` field applies to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks:

```
[visible:<cel_expression>]
```

| Key | Mandatory | Description |
|---|---|---|
| `visible` | no | A CEL expression evaluated against current parameter values. When false, the host should not render this item in the UI. Omitting `visible` is equivalent to `visible:true`. |

**Semantics:**

- `visible:` controls rendering. `enabled:` controls interactivity. They are independent: an item can be visible but disabled (greyed out), or invisible but still callable by the host programmatically.
- When a `GROUP` `visible:` evaluates to false, all items within the group are considered invisible regardless of their own `visible:` expressions.
- A non-visible item may still be targeted by `GET`, `SET`, `DO`, or `STREAM` commands — the host is responsible for deciding when programmatic access to a non-visible item is appropriate.
- The host is responsible for re-evaluating `visible:` expressions after the initial `GET` sweep and after each `CHANGED` notification.

**Common use case — host-invoked actions:** An ACTION declared `visible:false` with a static literal is never rendered as a button but remains callable by the host in response to UI events such as SVG element clicks (see Appendix D).

```
ACTION BEGIN diagram_click
label:Diagram Click
visible:false

ARG BEGIN element_id
type:seam/string
label:Element ID
ARG END

ACTION END
```

**Example:**

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

---

## 16. Error Codes

| Code | Meaning |
|---|---|
| `UNKNOWN_CMD` | Unrecognized command keyword |
| `UNKNOWN_PARAM` | No parameter with that ID |
| `UNKNOWN_ACTION` | No action with that ID |
| `NOT_READABLE` | Parameter is write-only |
| `NOT_WRITABLE` | Parameter is read-only |
| `OUT_OF_RANGE` | Numeric value outside declared min/max |
| `INVALID_VALUE` | Value cannot be parsed for the declared type |
| `BAD_ARGS` | Unknown argument name, missing required argument, or value invalid for declared type |
| `BUSY` | Device cannot accept this command right now |
| `FAILED` | Action attempted but failed (with optional message) |
| `NOT_WATCHABLE` | `WATCH` issued for a param not declared `watchable:true` |
| `ALREADY_WATCHING` | `WATCH` issued while already subscribed |
| `NOT_WATCHING` | `UNWATCH` issued while not subscribed |

---

## 17. USB CDC-ACM Identification Convention

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

## 18. Device Implementation Requirements

A conforming SEAM device must:

- Respond to `CAPS\r\n` within **500ms**, at any time after the connection is opened
- Return `ERR UNKNOWN_CMD\r\n` for any unrecognized command — never hang or ignore
- Never send unsolicited output except `DATA` frames and `CHANGED` lines for active
  `WATCH` subscriptions
- Reset internal streaming and watch state cleanly when the connection is closed and reopened
- Accept any baud rate if using USB CDC-ACM (baud rate is irrelevant for CDC-ACM)

A conforming SEAM device should:

- Respond to commands within **100ms** under normal conditions
- Declare sensible `default` values in CAPS where applicable
- Use descriptive IDs and descriptions to aid human understanding

---

## 19. Host Implementation Requirements

A conforming SEAM host must:

- Send `CAPS` immediately after opening a connection
- Perform a `GET` sweep of all readable parameters after `CAPS` to obtain live values
- Parse the frame format: `KEYWORD <id> <length>\r\n<data>\r\n` for `VALUE` and `DATA`
- Handle `ERR` responses gracefully without crashing
- Issue `GET <id>` upon receiving a `CHANGED <id>` notification to retrieve the new value
- Not send a new command until the previous command's response has been received,
  except when active streams are running
- Evaluate all `enabled:` CEL expressions after the initial `GET` sweep and after each
  `CHANGED` notification, and suppress interaction with any item whose expression
  evaluates to false

---

## 20. Versioning

This specification follows [Semantic Versioning](https://semver.org/).

- **MAJOR**: breaking changes to the wire format or command semantics
- **MINOR**: new commands, new block keys, backward-compatible additions
- **PATCH**: clarifications, corrections, no wire format change

The `version` field in a CAPS response reflects the **device firmware version**, not
the SEAM protocol version. Protocol version is tracked in this document.

### Changelog

**4.1.0**
- `visible:<cel_expression>` optional field added to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks
- When `visible:` evaluates to false, the host should not render the item in the UI; the item remains accessible programmatically via `GET`, `SET`, and `DO`
- When a `GROUP` `visible:` is false, all items within the group are considered invisible regardless of their own `visible:` expressions
- Primary use case: `ACTION` blocks declared `visible:false` act as host-invoked event handlers (e.g. SVG click handlers) without appearing as buttons — see Appendix D

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
<< OK "PWM enabled at 1200us, 50Hz, continuous mode."
```

---

## Appendix D — Interactive SVG Pattern

A module can expose an `image/svg+xml` parameter and receive click events from the host
through a paired ACTION. This section describes the full pattern.

### D.1 Concept

The device exposes a readable SVG parameter. The SVG markup must include `id` attributes
on all interactive elements — the host uses these IDs as stable identifiers for user
interactions. When the user clicks an element in the rendered SVG, the host sends a `DO`
command with the clicked element's ID. The device maps IDs to internal logic and responds.

```
device exposes SVG → host renders it → user clicks element → host calls DO with element id → device acts
```

### D.2 CAPS Declaration

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

ACTION BEGIN on_svg_click
label:SVG Click
description:Called by the host when the user clicks a named SVG element
visible:false

ARG BEGIN element_id
type:seam/string
label:Element ID
description:The id attribute of the clicked SVG element
ARG END

ACTION END

GROUP END
```

### D.3 SVG Structure Requirements

The SVG content returned by `GET diagram` must include `id` attributes on every element
the host should treat as interactive. Non-interactive elements should have no `id`, or
the host should be configured to ignore them.

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

### D.4 Host Responsibilities

1. Issue `GET diagram` after `CAPS` and render the SVG.
2. Attach click handlers to all SVG elements that have an `id` attribute.
3. On click, send `DO BEGIN on_svg_click` with `element_id` set to the element's `id` value.
4. If `diagram` is `watchable:true`, subscribe via `WATCH` and re-fetch on `CHANGED` to
   keep the rendered SVG in sync (the device may update element styles to reflect state).

### D.5 Wire Protocol Example

```
>> GET diagram
<< VALUE diagram 374
<< <374 bytes of SVG markup>

# user clicks the element with id="relay_k1"

>> DO BEGIN on_svg_click
>> IN element_id 8
>> relay_k1
>> DO END
<< OK

# user clicks the element with id="led_power"

>> DO BEGIN on_svg_click
>> IN element_id 9
>> led_power
>> DO END
<< OK
```

### D.6 Returning Updated State

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

### D.7 Optional: Passing Click Coordinates

If the device needs sub-element precision (e.g. a canvas-style SVG), the ACTION may
declare additional `x` and `y` arguments in SVG user-unit coordinates:

```
ACTION BEGIN on_svg_click
label:SVG Click
visible:false

ARG BEGIN element_id
type:seam/string
label:Element ID
description:The id attribute of the clicked SVG element; empty string if none
ARG END

ARG BEGIN x
type:seam/float
label:X
description:Click X in SVG user units
ARG END

ARG BEGIN y
type:seam/float
label:Y
description:Click Y in SVG user units
ARG END

ACTION END
```

Wire protocol with coordinates:

```
>> DO BEGIN on_svg_click
>> IN element_id 8
>> relay_k1
>> IN x 4
>> 34.5
>> IN y 4
>> 22.0
>> DO END
<< OK
```
