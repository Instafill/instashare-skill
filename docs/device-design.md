# Design: a physical session display for Claude Code

A small desk device — a screen and a couple of buttons — that shows the **model** and **reasoning effort** of your running Claude Code session(s), and lets you flip between multiple open sessions (e.g. several PowerShell windows) with the buttons.

**Feasibility verdict: yes, buildable with off-the-shelf hobby parts, and multi-window support works natively.** Claude Code's statusline feature delivers exactly the data needed, per session, in near-realtime. This document is the design and research write-up; no code yet.

## 1. Where the data comes from

Claude Code can run a user-configured **statusline command** and passes it a JSON payload on stdin
(docs: <https://code.claude.com/docs/en/statusline>). The payload includes everything the device needs:

| Field | Example | Notes |
| --- | --- | --- |
| `session_id` | `"abc123…"` | Unique per session — the key that makes multi-window support work |
| `model.display_name` | `"Opus"` | Bare name; **no** `(1M context)` marker (that can be inferred from `context_window.context_window_size`) |
| `model.id` | `"claude-opus-5"` | |
| `effort.level` | `"high"` | `low / medium / high / xhigh / max`; **live**, reflects mid-session `/effort` changes; absent for models without effort support |
| `cwd`, `workspace.project_dir` | `"C:/projects/foo"` | Useful to label which window is which |
| `context_window.used_percentage` | `8` | Nice-to-have extra display page |

Invocation behavior that shapes the design:

- The statusline command fires on session start/resume, each assistant message, mode changes, and an optional `refreshInterval` timer, debounced at 300 ms.
- **Each open session invokes it independently with its own `session_id`.** Three PowerShell windows → three independent streams of updates. This is why "will it support multiple PowerShell windows" is a yes by construction.
- On Windows the command can be a PowerShell script: `"statusLine": { "type": "command", "command": "powershell -NoProfile -File C:/Users/<you>/.claude/statusline.ps1" }` (forward slashes).
- There is **no session-ended event** — a closed window simply stops sending. Stale sessions must age out on the device (see §6).

## 2. Architecture

```
PowerShell window 1 ─ statusline script ─┐
PowerShell window 2 ─ statusline script ─┼─ HTTP POST /session ──> device (WiFi, LAN-only)
PowerShell window 3 ─ statusline script ─┘        ~1 KB JSON            │
                                                              session table keyed by session_id
                                                              screen shows one session; buttons cycle
```

Each statusline render does a **fire-and-forget HTTP POST** of `{session_id, model, effort, cwd, ts}` to the device, with a short timeout (~250 ms) so a powered-off device never slows the statusline down. The device keeps a table keyed by `session_id`, last-write-wins, and expires entries that haven't updated recently. The buttons move a cursor through the table; the screen shows the selected session with a `2/3`-style position marker.

## 3. How the device connects to the computer

| Option | How it works | Multi-window | Host-side software | Verdict |
| --- | --- | --- | --- | --- |
| **WiFi** (recommended) | Device joins your LAN, runs a tiny HTTP server | Natural — every window POSTs independently | None beyond the one-line POST in the statusline script | **Recommended** |
| **USB serial** | Device on a COM port | Needs an always-running host daemon: Windows COM ports are single-owner, so the windows can't all write to it directly | Daemon (COM discovery, lifetime management) | Fallback if no radio is wanted; **most secure** |
| **Bluetooth** | Classic SPP appears as a virtual COM port; BLE uses WinRT APIs | SPP inherits the same single-owner COM problem as USB (daemon needed); BLE needs WinRT code instead of a one-line HTTP call | Daemon or WinRT client | Not recommended — wireless, but with the worst of both worlds |

## 4. Security: is WiFi safe here?

The key fact: **only metadata ever leaves the PC** — model name, effort level, session id, a directory path, a timestamp. No transcript content, no code, no keys. The realistic worst case on a hostile LAN is someone reading "Opus / high / C:\projects\foo" or spoofing a fake session tile onto the screen.

With that threat model:

- **WiFi** is acceptable on a trusted home network with three cheap mitigations, all part of the firmware spec:
  1. **LAN-only, listen-only.** The device accepts inbound POSTs and makes *no outbound connections at all*. Never port-forward it or expose it to the internet.
  2. **Shared-secret header.** The statusline script sends `X-Device-Key: <token>`; firmware rejects requests without it. Stops casual LAN spoofing/reading.
  3. **Minimal payload.** Optionally send only the tail of `cwd` (project folder name) rather than the full path.
- **Bluetooth** is not meaningfully safer for this data. It is short-range and pairing-gated rather than IP-addressable, but BT/BLE stacks carry their own CVE history, and you pay the daemon/WinRT complexity from §3 for it.
- **USB serial is the genuinely safest option** — no radio, physically attached. Choose it if security outweighs the convenience cost of running a host daemon.

Recommendation: WiFi with mitigations 1–2 for a home/trusted network; USB serial if the device will live somewhere untrusted.

## 5. Hardware options

| Option | Parts cost | Radio | Firmware stack | Difficulty | Fit |
| --- | --- | --- | --- | --- | --- |
| **ESP32 dev board** (recommended) | ~$5–8 | WiFi + BT | Arduino/PlatformIO or MicroPython | Low — huge community, HTTP server examples everywhere | Best all-round |
| **Raspberry Pi Pico W** | ~$7 | WiFi | MicroPython-first | Low | Equivalent; pick if you prefer Python |
| **Arduino Uno/Nano or Pico (non-W)** | ~$4–10 | none (USB) | Arduino | Low, but requires the host daemon | The USB-serial/security option |
| **Desktop simulator** | $0 | n/a | Any local HTTP listener + window | Trivial | Validate the protocol before buying anything |

### Parts list for the recommended build

| Part | Indicative price |
| --- | --- |
| ESP32 dev board (e.g. ESP32-DevKitC / WROOM-32) | ~$6 |
| SSD1306 128×64 I²C OLED display | ~$4 |
| 2× tactile push buttons | <$1 |
| Breadboard + jumper wires (or solder direct) | ~$5 |
| USB cable for power/flashing | ~$2 |
| **Total** | **~$15–20** |

### Wiring sketch

- OLED via I²C: `SDA → GPIO21`, `SCL → GPIO22`, plus 3.3 V and GND — two signal pins total.
- Buttons: each between a GPIO (e.g. 32, 33) and GND, using the ESP32's internal pull-ups; pressed = LOW.
- Power over the USB cable (from the PC or any USB charger).

## 6. Firmware behavior spec (prose, no code yet)

- **Endpoint**: `POST /session` accepting JSON `{session_id, model, effort, cwd, ts}`; requires the `X-Device-Key` header; responds `204`. Optional `GET /healthz` for setup debugging.
- **Session table**: keyed by `session_id`, last-write-wins, capacity ~8. Entries older than **15 s** without an update are dropped (statusline fires at least every few seconds during activity; set `refreshInterval` for idle sessions).
- **Screen layout** (128×64): line 1 model name large; line 2 effort badge (`HIGH`), line 3 tail of `cwd`; bottom-right `2/3` session position; a subtle "stale" glyph if the selected session hasn't updated in >5 s.
- **Buttons**: next/prev session, debounced ~50 ms; long-press on one button toggles an extra page (context-window %, uptime). If the selected session expires, fall back to the most recently updated one.
- **Addressing**: mDNS (`http://claude-display.local`) with a fixed-IP fallback; WiFi credentials + device key compiled in or set via a one-time serial setup prompt.
- **No outbound connections. Ever.**

## 7. Host setup spec (Windows)

- A PowerShell statusline script that (a) renders whatever status text you want in the terminal and (b) fires the non-blocking POST with a ~250 ms timeout, swallowing all errors — a dead/absent device must never affect Claude Code.
- Wired in `settings.json` as shown in §1. Only **one** statusLine command is configurable, so if you already have a statusline, the device push becomes a few extra lines inside it rather than a separate script.
- The device key lives in the script (or an env var), matching the firmware.

## 8. Limitations and open questions

- `model.display_name` carries no `(1M context)` marker; if wanted, infer from `context_window.context_window_size` ≥ 1M.
- `effort` is absent for models that don't support effort — the screen should just omit the badge.
- Session end is detected only by expiry (no goodbye event), so a closed window lingers on screen for up to the expiry window.
- WiFi provisioning UX (hardcoded vs. serial setup vs. captive portal) — decide when building.
- Statusline update cadence during long idle periods depends on `refreshInterval`; without it, an idle session may look "stale" even though it's open.

## 9. Next steps (out of scope for this document)

1. Pick hardware (ESP32 + OLED recommended) — or start with the $0 desktop simulator to validate the protocol.
2. Firmware implementing §6; statusline script implementing §7.
3. Optional: a `device/` folder in this repo with firmware, the PowerShell script, and the simulator.
