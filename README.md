# DECO — Device Description & Control Protocol

**Version**: 2.1.0  
**Encoding**: ASCII / UTF-8  
**Status**: Draft  

## Related

| Repo | Description |
|---|---|
| [deco-host](https://github.com/tarasfilonenko/deco-host) | Generic desktop app for any DECO device |
| [modbench](https://github.com/tarasfilonenko/modbench) | ModBench hardware toolkit — an example DECO implementation |

---

## 1. Overview

DECO is a lightweight, human-readable protocol that allows any device — microcontroller,
single-board computer, or software process — to describe its capabilities to a host over
a serial connection. The host can then discover what the device can do and interact with
it dynamically, without needing prior knowledge of the device type.

A DECO device exposes:
- **Parameters** — named values that can be read and/or written; types are expressed as MIME types
- **Actions** — named functions that can be invoked, with typed arguments
- **Streams** — named data channels that emit values continuously

The host queries capabilities once on connect, then uses a small set of commands
(`GET`, `SET`, `DO`, `STREAM`) to interact with the device. No host-side configuration
or code generation is required to support a new device type.

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
| DECO | Full: params, actions, streams, groups | **Yes** | **Yes** | **Yes** |

---

## 2. Transport

DECO operates over any reliable byte stream. The reference transport is **USB CDC-ACM**
(virtual serial port), but DECO may also be used over:

- Hardware UART
- BLE serial (SPP / NUS)
- TCP socket
- Any other stream that preserves byte order and delivers data intact

Transport-specific considerations:

- **USB CDC-ACM**: baud rate setting is irrelevant; implementations must accept any value
- **UART**: both sides must agree on baud rate out of band
- **TCP**: the connection itself provides framing; no additional wrapper needed

DECO does not define device discovery or addressing. Those are handled by the transport
or the host application. For USB CDC-ACM, the host enumerates available ports and opens
them individually.

---

## 3. Line Format

All DECO communication uses text lines. Every line follows this structure:

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

All DECO structural declarations follow a universal pattern:

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

Argument data uses the same encoding as `SET` for the declared MIME type. `deco/` scalar
values are represented as ASCII text (e.g. `1000` for a `deco/int`). Arguments may be
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

### 5.6 `STREAM`

Start or stop a declared data stream.

```
STREAM <id> START\r\n
STREAM <id> STOP\r\n
```

Response:
```
OK\r\n
```

Errors:
```
ERR UNKNOWN_STREAM <id>\r\n
ERR ALREADY_STREAMING <id>\r\n
ERR NOT_STREAMING <id>\r\n
```

Once started, the device emits `DATA` frames asynchronously until `STREAM <id> STOP` is
received — see Section 12.3.

---

### 5.7 `WATCH` / `UNWATCH`

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

PARAM BEGIN ...
ACTION BEGIN ...
STREAM BEGIN ...

GROUP END
```

| Key | Mandatory | Description |
|---|---|---|
| `label` | yes | Human-readable group label |
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
description:<text>
[default:<value>]
[min:<value>]
[max:<value>]
[options:<value> ...]
[watchable:true]
[enabled:<cel_expression>]
PARAM END
```

### 8.1 Fields

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of the parameter value |
| `access` | yes | `r`, `w`, or `rw` |
| `description` | yes | Human-readable label |
| `default` | no | Informational default value |
| `min` | no | Minimum value — `deco/int` and `deco/float` only |
| `max` | no | Maximum value — `deco/int` and `deco/float` only |
| `options` | no | Space-separated option list — `deco/enum` only |
| `watchable` | no | `true` — host may subscribe via `WATCH` |
| `enabled` | no | CEL expression; when false, the host should disable interaction with this parameter — see Section 14 |

**Note on defaults:** The `default` field is informational. Hosts should always `GET`
live values after `CAPS` before populating their UI.

### 8.2 Examples

```
PARAM BEGIN pulse_width_us
type:deco/int
access:rw
min:500
max:2500
default:1500
description:Pulse width in microseconds
PARAM END

PARAM BEGIN enabled
type:deco/bool
access:rw
default:true
description:Enable PWM output
PARAM END

PARAM BEGIN mode
type:deco/enum
access:rw
default:continuous
options:continuous single sweep
description:Operating mode
PARAM END

PARAM BEGIN label
type:deco/string
access:rw
default:Servo 1
description:Channel label
PARAM END

PARAM BEGIN schematic
type:image/png
access:r
watchable:true
description:Module schematic diagram
PARAM END

PARAM BEGIN display
type:image/jpeg
access:w
description:Display framebuffer
PARAM END
```

---

## 9. ACTION Block

```
ACTION BEGIN <id>
description:<text>
[enabled:<cel_expression>]

[ARG BEGIN <id>
type:<mime_type>
description:<text>
[min:<value>]
[max:<value>]
[options:<value> ...]
ARG END]

ACTION END
```

### 9.1 ACTION Fields

| Key | Mandatory | Description |
|---|---|---|
| `description` | yes | Human-readable label |
| `enabled` | no | CEL expression; when false, the host should disable interaction with this action — see Section 14 |

### 9.2 ARG Fields

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of the argument value |
| `description` | yes | Human-readable label |
| `min` | no | Minimum value — `deco/int` and `deco/float` only |
| `max` | no | Maximum value — `deco/int` and `deco/float` only |
| `options` | no | Space-separated option list — `deco/enum` only |

Any MIME type may appear in `type`, including `image/*` and `application/octet-stream`.
If an action produces output, model it as a `PARAM` (readable after the action completes)
or a `STREAM`. See Section 5.4 for the `DO` wire protocol.

### 9.3 Examples

```
ACTION BEGIN center
description:Move to center position
ACTION END

ACTION BEGIN sweep
description:Sweep between two pulse widths in microseconds

ARG BEGIN start_us
type:deco/int
min:500
max:2500
description:Start pulse width in microseconds
ARG END

ARG BEGIN end_us
type:deco/int
min:500
max:2500
description:End pulse width in microseconds
ARG END

ACTION END

ACTION BEGIN load_frame
description:Load JPEG image into display framebuffer

ARG BEGIN frame
type:image/jpeg
description:Frame data
ARG END

ACTION END
```

---

## 10. STREAM Block

```
STREAM BEGIN <id>
type:<mime_type>
description:<text>
[enabled:<cel_expression>]
STREAM END
```

| Key | Mandatory | Description |
|---|---|---|
| `type` | yes | MIME type of emitted values |
| `description` | yes | Human-readable label |
| `enabled` | no | CEL expression; when false, the host should not start this stream — see Section 14 |

Any MIME type may be used, including `image/*` and `application/octet-stream`.

### 10.1 Examples

```
STREAM BEGIN position
type:deco/float
description:Live position feedback in microseconds
STREAM END

STREAM BEGIN capture
type:application/octet-stream
description:Raw capture data
STREAM END
```

---

## 11. MIME Type System

The `type` field in `PARAM`, `ACTION`, and `STREAM` blocks uses MIME types throughout.

### 11.1 DECO Informal Namespace

DECO defines a `deco/` informal namespace for primitive scalar types. These are not
registered MIME types — they are DECO-internal.

| Type | Description |
|---|---|
| `deco/int` | Integer, represented as decimal ASCII |
| `deco/float` | Floating point, represented as decimal ASCII |
| `deco/bool` | Boolean — ASCII `true` or `false` |
| `deco/string` | UTF-8 string |
| `deco/enum` | One of a declared set of string options |

### 11.2 Standard MIME Types

Any valid MIME type may be used. Common examples:

| Type | Description |
|---|---|
| `image/png` | PNG image |
| `image/jpeg` | JPEG image |
| `image/bmp` | BMP image |
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

Once a stream is started with `STREAM <id> START`, the device emits frames asynchronously
until `STREAM <id> STOP` is received:

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
- When `enabled:` is false, the host should not send `GET`, `SET`, `DO`, or `STREAM` commands targeting the item.
- The device may return `ERR BUSY` or `ERR FAILED` if a command is received for a disabled item, but correct host behaviour makes this unnecessary.
- The host is responsible for keeping `enabled:` expressions up to date. Strategies such as re-evaluating on `CHANGED` notifications, polling, or a combination of both are all valid — the protocol does not mandate a specific approach.

> **Recommendation:** Parameters referenced in `enabled:` expressions should be declared `watchable:true` to allow the host to react promptly to value changes. This is not enforced by the protocol.

**Python host implementation:** The `cel-python` package (`pip install cel-python`) provides a pure Python CEL evaluator suitable for the host application.

**Example:**

```
GROUP BEGIN sweep
label:Sweep Control
enabled:mode == "sweep"

PARAM BEGIN start_us
type:deco/int
access:rw
min:500
max:2500
description:Sweep start in microseconds
PARAM END

PARAM BEGIN end_us
type:deco/int
access:rw
min:500
max:2500
enabled:start_us < end_us
description:Sweep end in microseconds
PARAM END

GROUP END
```

---

## 15. Error Codes

| Code | Meaning |
|---|---|
| `UNKNOWN_CMD` | Unrecognized command keyword |
| `UNKNOWN_PARAM` | No parameter with that ID |
| `UNKNOWN_ACTION` | No action with that ID |
| `UNKNOWN_STREAM` | No stream with that ID |
| `NOT_READABLE` | Parameter is write-only |
| `NOT_WRITABLE` | Parameter is read-only |
| `OUT_OF_RANGE` | Numeric value outside declared min/max |
| `INVALID_VALUE` | Value cannot be parsed for the declared type |
| `BAD_ARGS` | Unknown argument name, missing required argument, or value invalid for declared type |
| `BUSY` | Device cannot accept this command right now |
| `FAILED` | Action attempted but failed (with optional message) |
| `ALREADY_STREAMING` | `STREAM START` issued while already streaming |
| `NOT_STREAMING` | `STREAM STOP` issued while not streaming |
| `NOT_WATCHABLE` | `WATCH` issued for a param not declared `watchable:true` |
| `ALREADY_WATCHING` | `WATCH` issued while already subscribed |
| `NOT_WATCHING` | `UNWATCH` issued while not subscribed |

---

## 16. USB CDC-ACM Identification Convention

When a DECO device uses USB CDC-ACM as its transport, it should set the USB manufacturer
string to `"DECO"`. This allows host applications and watcher scripts to identify DECO
devices by scanning serial ports without opening them.

```c
#define USB_MANUFACTURER  "DECO"           // marks this as a DECO device
#define USB_PRODUCT       "Servo Tester"   // human-readable, should match name in CAPS
```

This convention is optional — DECO works over any transport and the protocol itself has
no dependency on USB descriptor strings. However, implementing it enables:

- Automatic device discovery by DECO Host and similar tools
- Filtering DECO ports from unrelated serial devices
- Instant display of a human-readable name before CAPS is received

The `USB_PRODUCT` string should match the `name` field in CAPS exactly.

---

## 17. Device Implementation Requirements

A conforming DECO device must:

- Respond to `CAPS\r\n` within **500ms**, at any time after the connection is opened
- Return `ERR UNKNOWN_CMD\r\n` for any unrecognized command — never hang or ignore
- Never send unsolicited output except `DATA` frames for active streams and `CHANGED`
  lines for active `WATCH` subscriptions
- Reset internal streaming and watch state cleanly when the connection is closed and reopened
- Accept any baud rate if using USB CDC-ACM (baud rate is irrelevant for CDC-ACM)

A conforming DECO device should:

- Respond to commands within **100ms** under normal conditions
- Declare sensible `default` values in CAPS where applicable
- Use descriptive IDs and descriptions to aid human understanding

---

## 18. Host Implementation Requirements

A conforming DECO host must:

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

## 19. Versioning

This specification follows [Semantic Versioning](https://semver.org/).

- **MAJOR**: breaking changes to the wire format or command semantics
- **MINOR**: new commands, new block keys, backward-compatible additions
- **PATCH**: clarifications, corrections, no wire format change

The `version` field in a CAPS response reflects the **device firmware version**, not
the DECO protocol version. Protocol version is tracked in this document.

### Changelog

**2.1.0**
- `enabled:<cel_expression>` optional field added to `GROUP`, `PARAM`, `ACTION`, and `STREAM` blocks
- CEL (Common Expression Language) used for conditional enabling; the host evaluates expressions against current parameter values keyed by param id
- When a `GROUP` `enabled:` is false, all items in the group are treated as disabled regardless of their own `enabled:` expressions
- Host must not send `GET`, `SET`, `DO`, or `STREAM` commands targeting a disabled item
- Host must re-evaluate `enabled:` expressions after the initial `GET` sweep and after each `CHANGED` notification

**2.0.0**
- Universal block structure: `KEYWORD BEGIN [id]` / `KEYWORD END` throughout
- CAPS, GROUP, PARAM, ACTION, STREAM all follow the same block pattern
- Identity fields use `key:value` format (`type`, `name`, `version`), replacing `TYPE`, `NAME`, `VERSION` keyword lines
- MIME type system replaces primitive type names; `deco/` informal namespace for scalar primitives
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

The smallest valid DECO CAPS response:

```
CAPS BEGIN
type:temperature_sensor
name:Temperature Sensor
version:1.0.0

GROUP BEGIN readings
label:Readings

PARAM BEGIN temp_c
type:deco/float
access:r
description:Temperature in Celsius
PARAM END

STREAM BEGIN temp
type:deco/float
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
type:deco/int
access:rw
min:500
max:2500
default:1500
description:Pulse width in microseconds
PARAM END

PARAM BEGIN frequency_hz
type:deco/int
access:rw
min:50
max:400
default:50
description:PWM frequency in Hz
PARAM END

PARAM BEGIN enabled
type:deco/bool
access:rw
default:true
description:Enable PWM output
PARAM END

PARAM BEGIN mode
type:deco/enum
access:rw
default:continuous
options:continuous single sweep
description:Operating mode
PARAM END

PARAM BEGIN schematic
type:image/png
access:r
watchable:true
description:Module schematic diagram
PARAM END

PARAM BEGIN display
type:image/jpeg
access:w
description:Display framebuffer
PARAM END

ACTION BEGIN center
description:Move to center position
ACTION END

ACTION BEGIN sweep
description:Sweep between two pulse widths in microseconds

ARG BEGIN start_us
type:deco/int
min:500
max:2500
description:Start pulse width in microseconds
ARG END

ARG BEGIN end_us
type:deco/int
min:500
max:2500
description:End pulse width in microseconds
ARG END

ACTION END

STREAM BEGIN position
type:deco/float
description:Live position feedback in microseconds
STREAM END

GROUP END

GROUP BEGIN info
label:Module Info

PARAM BEGIN label
type:deco/string
access:rw
default:Servo 1
description:Channel label
PARAM END

PARAM BEGIN uptime_s
type:deco/int
access:r
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
<< type:deco/int
<< access:rw
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

>> STREAM position START
<< OK
<< DATA position 6
<< 1487.3
<< DATA position 6
<< 1488.1

>> STREAM position STOP
<< OK

>> SET display 3072
>> <3072 bytes of JPEG data>
<< OK

>> STATUS
<< OK "PWM enabled at 1200us, 50Hz, continuous mode."
```
