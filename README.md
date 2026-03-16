# NuControl

A browser-based configuration editor for the [Berglund Instruments](https://berglundinstruments.com/) [NuEVI](https://berglundinstruments.com/nuevi-trumpet-horn-tuba-fingering-instrument/) and [NuRAD](https://berglundinstruments.com/nurad-ewi-sax-etc-fingering-instrument/) wind controllers. Works with firmware version 1.4.4 and later.

Use this live at [config.berglundinstruments.com](http://config.berglundinstruments.com)

## Usage

Serve the directory with a local HTTP server (required so the browser can fetch `config-items.json`):

```bash
python3 -m http.server
```

Then open http://localhost:8000/ in a browser.

## Features

- **Live MIDI** — connect to the instrument over USB and send/receive config directly via sysex, using the browser's Web MIDI API (Chrome, Firefox 99+, Edge, Opera; not Safari)
- **Load / Save** configuration files in three formats:
  - `.json` — human-readable config with name, comment, and timestamp
  - `.mid` — standard type-0 MIDI file containing a NuEVI sysex dump
  - `.syx` — raw sysex file with F0/F7 framing, compatible with general sysex utilities
- **Version gating** — items introduced in a later firmware version than the loaded config are hidden automatically; loading a file with mismatched items shows a yellow warning
- **Checksum validation** — CRC32 is verified on load; mismatches warn rather than block
- **Four input styles** driven by `uiModel` in `config-items.json`:
  - Default numeric text input with ▲/▼ stepper buttons
  - `slider` — range slider for wide-range values (e.g. hardware calibration)
  - `select` — styled dropdown for enumerated options
  - `bitfield` — a column of checkboxes for packed bit flags
- **Offset display** via `dataModel: "offset:N"` — stores the raw value but displays and accepts signed/offset values (e.g. transpose shown as −12 to +12)

## MIDI commands

All sysex messages share the same outer framing: `F0 [inner bytes] F7`.

The inner bytes follow this layout:

| Bytes | Content |
|-------|---------|
| 0–2 | Vendor ID `00 3E 7F` |
| 3–10 | Command string (ASCII, 8 bytes) |
| 11–12 | Payload size N (7-bit MIDI encoded uint16) — omitted for commands with no payload |
| 13–(13+N−1) | Payload |
| (13+N)–(16+N) | CRC32 (IEEE 802.3, 4 bytes, MSB of each byte cleared) — omitted for commands with no payload |

### Command reference

| Command | Direction | Payload | Description |
|---------|-----------|---------|-------------|
| `NuEVIc00` | Host → Instrument | None | Request a config dump. The instrument responds with `NuEVIc01`. Message is exactly 13 bytes (F0 + vendor + command + F7). |
| `NuEVIc01` | Instrument → Host | Config values | Config dump response. Contains the full config payload with CRC. |
| `NuEVIc02` | Host → Instrument | Config values | Write config to instrument. Same payload format as `NuEVIc01`. |
| `NuEVIc03` | Host → Instrument | None | Query device info. The instrument responds with `NuEVIc04`. Message is exactly 13 bytes. |
| `NuEVIc04` | Instrument → Host | See below | Device info response. |

`NuEVIc04` inner payload (after the command string, no size/CRC fields):

| Bytes | Content |
|-------|---------|
| 11–12 | Config version (7-bit MIDI encoded uint16) |
| 13+ | Firmware version string (variable-length ASCII, e.g. `1.4.6`) |

## File format

NuEVI config sysex structure:

| Bytes | Content |
|-------|---------|
| 0–2 | Vendor ID `00 3E 7F` |
| 3–10 | Command `NuEVIc01` (ASCII) |
| 11–12 | Payload size N (7-bit MIDI encoded uint16), dependent on firmware version |
| 13–(13+N−1) | N/2 × uint16 config values (7-bit MIDI encoded, 2 bytes each) |
| (13+N)–(16+N) | CRC32 (IEEE 802.3, 4 bytes with MSB of each byte set to 0) |

MIDI encoding: Each piece of data in the payload is an unsigned integer up to 14 bits in size. It is sent over MIDI as two bytes, using 7 bits per byte (MSB set to 0)

The payload size equals the address of the last config item defined in that firmware version + 2, so it grows as new items are added. When saving, the tool writes the minimum payload needed to cover all visible items.

When wrapped in a `.mid` file the payload sits in a type-0 MIDI file with a single track-0 sysex event.

## Config items

Defined in `config-items.json` as an array of sections. Each item has:

```json
"ID": {
  "address":        0,
  "friendlyName":   "Human readable name",
  "min":            0,
  "max":            127,
  "factoryDefault": 0,
  "minVersion":     24,
  "uiModel":        "slider | select:v;label:... | bitfield:bit;label:...",
  "dataModel":      "offset:N"
}
```

`address` is the byte offset within the EEPROM payload (always even; each value occupies 2 bytes, this should always be an even number).

## Tech stack

Single-file web app — `index.html` + `style.css`. Uses Vue 3 via CDN (no build step). All sysex encoding/decoding, MIDI file parsing, and Web MIDI communication is implemented in plain JavaScript in `index.html`.
