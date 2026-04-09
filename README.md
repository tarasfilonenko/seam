# DECO — Device Description & Control Protocol

**Version**: 1.0.0  
**Encoding**: ASCII / UTF-8  
**Status**: Draft  

---

## 1. Overview

DECO is a lightweight, human-readable protocol that allows any device — microcontroller,
single-board computer, or software process — to describe its capabilities to a host over
a serial connection. The host can then discover what the device can do and interact with
it dynamically, without needing prior knowledge of the device type.

A DECO device exposes:
- **Parameters** — named values that can be read and/or written
- **Actions** — named functions that can be invoked
- **Streams** — named data channels that emit values continuously

The host queries capabilities once on connect, then uses a small set of commands
(`GET`, `SET`, `DO`, `STREAM`) to interact with the device. No host-side configuration
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
- Maximum line length: **128 bytes** including the terminator.
- Fields are separated by **one or more spaces**.
- Arguments containing spaces must be **double-quoted**: `"my value"`.
- Escape sequences inside quoted strings: `\"` (literal quote), `\\` (literal backslash).
- Keywords and type names are **UPPERCASE**.
- IDs (parameter names, action names, stream IDs, group IDs, device types) use
  **lowercase alphanumeric with underscores** only: `[a-z0-9_]+`.
- Lines beginning with `#` are **comments** and must be ignored by the receiver.

---

## 4. Commands (Host → Device)

### 4.1 `CAPS`

Request full capability description. This is the **only handshake command**. CAPS returns
everything the host needs: device identity, firmware version, and all capabilities. There
is no separate HELLO command.

```
CAPS\r\n
```

Response: a multi-line CAPS block (see Section 5).

The device must respond to `CAPS` at any time, including immediately after power-on or
port open. Response must arrive within **500ms**.

---

### 4.2 `GET`

Read one or more parameter values.

```
GET <id> [<id> ...]\r\n
```

Response — single parameter:
```
OK <value>\r\n
```

Response — multiple parameters, values in the same order as requested:
```
OK <value1> <value2> ...\r\n
```

Errors:
```
ERR UNKNOWN_PARAM <id>\r\n
ERR NOT_READABLE <id>\r\n
```

Example:
```
>> GET pulse_width_us
<< OK 1500

>> GET pulse_width_us enabled
<< OK 1500 true
```

---

### 4.3 `SET`

Write one or more parameter values. Multiple id/value pairs may be set atomically in
one command.

```
SET <id> <value> [<id> <value> ...]\r\n
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
>> SET pulse_width_us 1200
<< OK

>> SET pulse_width_us 1200 enabled true
<< OK
```

---

### 4.4 `DO`

Invoke a named action with zero or more arguments.

```
DO <id> [<arg1> <arg2> ...]\r\n
```

Response:
```
OK [<return_value>]\r\n
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
>> DO center
<< OK

>> DO sweep 1000 2000
<< OK
```

---

### 4.5 `STATUS`

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

### 4.6 `STREAM`

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

Once started, the device emits stream data asynchronously (see Section 6).

---

## 5. CAPS Block

The CAPS response is a multi-line block bounded by `CAPS BEGIN` and `CAPS END`. It
contains device identity metadata followed by one or more groups, each containing
parameters, actions, and streams.

### 5.1 Full Structure

```
CAPS BEGIN\r\n
TYPE <device_type>\r\n
NAME "<display_name>"\r\n
VERSION <semver>\r\n
GROUP <id> "<label>"\r\n
PARAM ...\r\n
ACTION ...\r\n
STREAM ...\r\n
END GROUP\r\n
[GROUP ...]\r\n
CAPS END\r\n
```

All three identity lines (`TYPE`, `NAME`, `VERSION`) are required and must appear before
any `GROUP` lines.

### 5.2 Identity Lines

| Line | Format | Description |
|---|---|---|
| `TYPE` | `TYPE <device_type>` | Machine-readable device identifier, `[a-z0-9_]+` |
| `NAME` | `NAME "<display_name>"` | Human-readable device name |
| `VERSION` | `VERSION <semver>` | Firmware/software version, semantic versioning |

Example:
```
TYPE servo_tester
NAME "Servo Tester"
VERSION 1.2.0
```

### 5.3 GROUP

Groups organise parameters, actions, and streams into logical sections. A DECO device
must declare at least one group. All PARAMs, ACTIONs, and STREAMs must appear inside
a group.

```
GROUP <id> "<label>"\r\n
...contents...
END GROUP\r\n
```

Groups are logical only — they carry no layout hints. Host applications may present
groups in any order they choose.

### 5.4 PARAM

Declares a readable and/or writable parameter.

```
PARAM <id> <type> <access> [<min> <max>] [<default>] "<description>"\r\n
```

| Field | Values | Notes |
|---|---|---|
| `id` | `[a-z0-9_]+` | Machine-readable name, unique within the device |
| `type` | `int` `float` `bool` `string` `enum` | Value type |
| `access` | `r` `w` `rw` | Read-only / write-only / read-write |
| `min` `max` | numeric | Optional; for `int` and `float` types only |
| `default` | type-appropriate | Optional; informational — host should always GET to verify |
| `description` | quoted string | Human-readable label |

**Enum type:**

Enum params use a different field order — the default is quoted and comes before the
option list. Description is always the last quoted string.

```
PARAM <id> enum <access> "<default>" <option1> <option2> ... "<description>"\r\n
```

**Examples:**

```
PARAM pulse_width_us int  rw 500 2500 1500        "Servo pulse width in microseconds"
PARAM frequency_hz   int  rw 50  400  50          "PWM frequency in Hz"
PARAM enabled        bool rw true                 "Enable PWM output"
PARAM label          string rw "Servo 1"          "Channel label"
PARAM mode           enum rw "continuous" continuous single sweep "Operating mode"
```

**Note on defaults:** The `default` field is informational. Hosts should always perform
a `GET` sweep after `CAPS` to obtain the live values before populating their UI.

### 5.5 ACTION

Declares a callable action — a named function the host can invoke via `DO`.

```
ACTION <id> [<arg_type> ...] "<description>"\r\n
```

| Field | Values | Notes |
|---|---|---|
| `id` | `[a-z0-9_]+` | Machine-readable name |
| `arg_type` | `int` `float` `bool` `string` | Zero or more argument types, in order |
| `description` | quoted string | Human-readable label |

Examples:
```
ACTION center                  "Move servo to center position"
ACTION sweep int int           "Sweep between two pulse widths (us)"
ACTION set_label string        "Set the channel label"
```

### 5.6 STREAM

Declares an available data stream the host can start and stop via the `STREAM` command.

```
STREAM <id> <data_type> "<description>"\r\n
```

| Field | Values | Notes |
|---|---|---|
| `id` | `[a-z0-9_]+` | Stream identifier |
| `data_type` | `int` `float` `bool` `binary` | Type of emitted values |
| `description` | quoted string | Human-readable label |

Examples:
```
STREAM position float  "Current servo position feedback (us)"
STREAM capture  binary "Raw logic capture data"
STREAM fault    bool   "Fault condition alert"
```

### 5.7 Complete CAPS Example

```
CAPS BEGIN
TYPE servo_tester
NAME "Servo Tester"
VERSION 1.0.0
GROUP pwm "PWM Control"
PARAM pulse_width_us int rw 500 2500 1500 "Pulse width in microseconds"
PARAM frequency_hz int rw 50 400 50 "PWM frequency in Hz"
PARAM enabled bool rw true "Enable PWM output"
PARAM mode enum rw "continuous" continuous single sweep "Operating mode"
ACTION center "Move to center position"
ACTION sweep int int "Sweep between two pulse widths (us)"
STREAM position float "Live position feedback (us)"
END GROUP
GROUP info "Module Info"
PARAM label string rw "Servo 1" "Channel label"
PARAM uptime_s int r "Uptime in seconds"
END GROUP
CAPS END
```

---

## 6. Stream Data (Device → Host, Asynchronous)

Once a stream is started with `STREAM <id> START`, the device emits data asynchronously
until the host sends `STREAM <id> STOP`. The device must not emit stream data before
`STREAM START` is received.

### 6.1 Text Streams

For streams with `data_type` of `int`, `float`, or `bool`:

```
DATA <id> <value>\r\n
```

The host should avoid sending commands that expect responses while high-rate DATA lines
are arriving, or must buffer and correlate responses carefully.

Example:
```
DATA position 1523.5
DATA position 1524.1
DATA fault false
```

### 6.2 Binary Streams

For streams with `data_type` `binary`, the device uses inline framing markers to switch
the connection into binary mode for the duration of the payload.

**Frame structure:**

```
STREAM_BEGIN <id> <length>\r\n
<length raw bytes>
STREAM_END <crc32_hex>\r\n
```

- `STREAM_BEGIN` and `STREAM_END` are text lines, `\r\n` terminated.
- `<length>` is the byte count of the raw payload (decimal integer).
- Raw bytes follow immediately after the `\r\n` of `STREAM_BEGIN` with no additional
  framing.
- `<crc32_hex>` is the **lowercase hex** CRC-32 of the raw payload bytes only
  (not the markers).
- After `STREAM_END` the connection returns to text mode.

**Host parser state machine:**

```
TEXT_MODE:
    read line
    if starts with "STREAM_BEGIN" → parse id and length → enter BINARY_MODE
    else → handle as normal response or DATA line

BINARY_MODE:
    read exactly <length> bytes → store as payload
    read next line → expect "STREAM_END <crc32_hex>"
    if CRC matches → deliver payload to application → enter TEXT_MODE
    if CRC fails   → discard payload, signal CRC error → enter TEXT_MODE
```

**Continuous binary streams** emit successive frames until stopped:

```
STREAM_BEGIN capture 512\r\n
<512 bytes>
STREAM_END a3f1c209\r\n
STREAM_BEGIN capture 512\r\n
<512 bytes>
STREAM_END 7b2e4411\r\n
```

When the host sends `STREAM <id> STOP`, the device finishes its current frame before
stopping. It must not stop mid-frame.

---

## 7. Responses

Every command sent by the host receives exactly one response line from the device,
except during active streams where `DATA` lines and binary frames are interspersed
asynchronously.

**Success:**
```
OK [<value>]\r\n
```

**Error:**
```
ERR <code> [<detail>]\r\n
```

The device must always respond to commands even during active streams. Commands received
during binary frame transmission must be queued and processed after the frame completes.

---

## 8. Error Codes

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
| `BAD_ARGS` | Wrong number or type of arguments for action |
| `BUSY` | Device cannot accept this command right now |
| `FAILED` | Action attempted but failed (with optional message) |
| `ALREADY_STREAMING` | `STREAM START` issued while already streaming |
| `NOT_STREAMING` | `STREAM STOP` issued while not streaming |

---

## 9. USB CDC-ACM Identification Convention

When a DECO device uses USB CDC-ACM as its transport, it should set the USB manufacturer
string to `"DECO"`. This allows host applications and watcher scripts to identify DECO
devices by scanning serial ports without opening them.

```c
#define USB_MANUFACTURER  "DECO"           // marks this as a DECO device
#define USB_PRODUCT       "Servo Tester"   // human-readable, should match NAME in CAPS
```

This convention is optional — DECO works over any transport and the protocol itself has
no dependency on USB descriptor strings. However, implementing it enables:

- Automatic device discovery by DECO Host and similar tools
- Filtering DECO ports from unrelated serial devices
- Instant display of a human-readable name before CAPS is received

The `USB_PRODUCT` string should match the `NAME` field in CAPS exactly. This allows the
host to show the correct device name immediately when the port appears, before CAPS has
been sent and parsed.

---

## 10. Device Implementation Requirements

A conforming DECO device must:

- Respond to `CAPS\r\n` within **500ms**, at any time after the connection is opened
- Return `ERR UNKNOWN_CMD\r\n` for any unrecognized command — never hang or ignore
- Never send unsolicited output except `DATA` lines for active text streams and
  `STREAM_BEGIN`/`STREAM_END` frames for active binary streams
- Queue commands received during binary frame transmission — do not interrupt a frame
- Reset internal streaming state cleanly when the connection is closed and reopened
- Accept any baud rate if using USB CDC-ACM (baud rate is irrelevant for CDC-ACM)

A conforming DECO device should:

- Respond to commands within **100ms** under normal conditions
- Declare sensible `default` values in CAPS where applicable
- Use descriptive IDs and descriptions to aid human understanding

---

## 11. Host Implementation Requirements

A conforming DECO host must:

- Send `CAPS` immediately after opening a connection
- Perform a `GET` sweep of all readable parameters after `CAPS` to obtain live values
- Correctly implement the binary stream parser state machine (Section 6.2)
- Handle `ERR` responses gracefully without crashing
- Not send a new command until the previous command's response has been received,
  except when an active text stream is running

---

## 12. Versioning

This specification follows [Semantic Versioning](https://semver.org/).

- **MAJOR**: breaking changes to the wire format or command semantics
- **MINOR**: new commands, new CAPS keywords, backward-compatible additions
- **PATCH**: clarifications, corrections, no wire format change

The `VERSION` field in a CAPS response reflects the **device firmware version**, not
the DECO protocol version. Protocol version is tracked in this document.

### Changelog

**1.0.0** — Initial release.

---

## Appendix A — Minimal CAPS Example

The smallest valid DECO CAPS response:

```
CAPS BEGIN
TYPE temperature_sensor
NAME "Temperature Sensor"
VERSION 1.0.0
GROUP readings "Readings"
PARAM temp_c float r "Temperature in Celsius"
STREAM temp float "Live temperature stream"
END GROUP
CAPS END
```

---

## Appendix B — Example Session (Servo Tester)

```
>> CAPS
<< CAPS BEGIN
<< TYPE servo_tester
<< NAME "Servo Tester"
<< VERSION 1.0.0
<< GROUP pwm "PWM Control"
<< PARAM pulse_width_us int rw 500 2500 1500 "Pulse width in microseconds"
<< PARAM frequency_hz int rw 50 400 50 "PWM frequency in Hz"
<< PARAM enabled bool rw true "Enable PWM output"
<< PARAM mode enum rw "continuous" continuous single sweep "Operating mode"
<< ACTION center "Move to center position"
<< ACTION sweep int int "Sweep between two pulse widths (us)"
<< STREAM position float "Live position feedback (us)"
<< END GROUP
<< CAPS END

>> GET pulse_width_us frequency_hz enabled mode
<< OK 1500 50 true continuous

>> SET pulse_width_us 1200
<< OK

>> DO center
<< OK

>> STREAM position START
<< OK
<< DATA position 1487.3
<< DATA position 1488.1
<< DATA position 1489.0

>> STREAM position STOP
<< OK

>> STATUS
<< OK "PWM enabled at 1200us, 50Hz, continuous mode."
```

---

## Appendix C — Example Session (Binary Stream)

```
>> CAPS
<< CAPS BEGIN
<< TYPE logic_analyzer
<< NAME "Logic Analyzer"
<< VERSION 1.0.0
<< GROUP capture "Capture"
<< PARAM sample_rate enum rw "20MHz" 1MHz 5MHz 10MHz 20MHz "Sample rate"
<< PARAM channels int rw 1 8 8 "Active channels"
<< PARAM trigger_ch int rw 0 7 0 "Trigger channel"
<< PARAM trigger_edge enum rw "rising" rising falling "Trigger edge"
<< PARAM samples int rw 1000 50000 50000 "Samples to capture"
<< ACTION arm "Arm trigger"
<< ACTION force "Force immediate capture"
<< STREAM capture binary "Raw capture data"
<< END GROUP
<< CAPS END

>> GET sample_rate channels trigger_ch trigger_edge samples
<< OK 20MHz 8 0 rising 50000

>> SET sample_rate 10MHz
<< OK

>> SET samples 10000
<< OK

>> DO arm
<< OK

>> STREAM capture START
<< OK
<< STREAM_BEGIN capture 10000
<< <10000 raw bytes>
<< STREAM_END a3f1c209
```
