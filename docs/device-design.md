# Design: a physical session display for Claude Code

A small desk device — a screen and buttons — that shows the **model**, **reasoning effort**, and **activity state** of your running Claude Code sessions, follows whichever window you're focused on, and lets you switch between sessions from the device itself.

**Scope: commercial product, not a hobby build.** That constraint drives most decisions below — zero-install hardware, no WiFi provisioning, works on corporate-managed machines, cheap certification path.

**Feasibility verdict: buildable.** Claude Code's statusline feature emits exactly the required data, per session, in near-realtime.

## 1. Decisions locked

| Decision | Choice | Why |
| --- | --- | --- |
| Transport | **USB-C, CDC-ACM** | Driverless on Win10+/macOS/Linux. No WiFi password, no radio certification, works on locked-down corporate networks. |
| Setup | **Composite CDC + mass storage** | Device also enumerates a small read-only drive holding the one-time setup file. No download, no admin rights. |
| Host software | **Background daemon + statusline script** | Required for focus-following; also unlocks activity state and bidirectional buttons. |
| Session selection | **Follow focused window, buttons override** | Auto-follow with a most-recently-active fallback. |
| WiFi | **Deferred, not rejected** | Daemon abstracts transport; WiFi becomes a later SKU without redesign. |

## 2. Where the data comes from

Claude Code runs a user-configured **statusline command**, passing a JSON payload on stdin
(docs: <https://code.claude.com/docs/en/statusline>):

| Field | Example | Notes |
| --- | --- | --- |
| `session_id` | `"abc123…"` | Unique per session — the key for multi-session tracking |
| `model.display_name` | `"Opus"` | Bare name; **no** `(1M context)` marker (infer from `context_window.context_window_size`) |
| `effort.level` | `"high"` | `low / medium / high / xhigh / max`; live, reflects mid-session `/effort`; absent on models without effort support |
| `cwd`, `workspace.project_dir` | `"C:/projects/foo"` | Labels the session; also used for tab-title matching (§5) |
| `context_window.used_percentage` | `8` | Secondary display page |

Invocation behavior that shapes the design:

- Fires on session start/resume, each assistant message, mode changes, and an optional `refreshInterval`, debounced 300 ms.
- **Each session invokes it independently with its own `session_id`.** Multi-window support is inherent.
- On Windows the command can be a PowerShell script; configure with forward slashes.
- **No session-ended event.** The daemon detects termination by watching the process instead of relying on timeouts.

## 3. Architecture

```
Claude Code session 1 ─ statusline script ─┐
Claude Code session 2 ─ statusline script ─┼─> named pipe / localhost ─> HOST DAEMON
Claude Code session 3 ─ statusline script ─┘                                 │
                                                                             │ owns USB port (by VID/PID)
   foreground-window hook ──────────────────────────────────────────────────>│ USB CDC-ACM, bidirectional
   process watcher (session death) ─────────────────────────────────────────>│         │
                                                                        button events  v
                                                                             <──── DEVICE
```

The statusline script is deliberately trivial: serialize the payload, write it to a local pipe, exit. All logic lives in the daemon, which is the only process that touches the USB port — so there is no port contention between sessions.

## 4. Host daemon responsibilities

- **Ingest** statusline pushes from every session over a named pipe (Windows) or Unix socket.
- **Track focus** via an event-driven foreground-window hook (`SetWinEventHook`/`EVENT_SYSTEM_FOREGROUND` on Windows; `NSWorkspace.frontmostApplication` on macOS — note this needs **no** Accessibility permission at app level).
- **Map window → session**: walk the statusline script's parent process chain at render time to record which terminal process owns each session, then resolve the foreground window's PID against that map.
- **Detect session death** by watching the Claude Code process, removing it from the display immediately.
- **Enrich state** beyond the statusline payload — most valuably **"waiting for approval"**, the highest-value signal for a peripheral display.
- **Own the USB link**: discover the device by VID/PID, serialize state, receive button events.
- **Act on button events**: switch displayed session, and bring a session's terminal window to the foreground.

## 5. Known limitation: tabs vs. windows

Foreground-window detection resolves to a **window**, not a tab. Windows Terminal hosts all tabs in a single process under one window handle; macOS Terminal and iTerm2 behave the same. Three sessions in three tabs of one window are indistinguishable via the foreground-window API.

**Mitigation**: read the foreground window's *title*, which reflects the active tab, and match it against the session's `cwd` (Claude Code sets the terminal title). This is a heuristic, not a guaranteed API.

**Fallback**: when the title is ambiguous, select the most recently updated session among candidates. This must be built regardless — it is also the correct behavior when no Claude Code window is focused at all.

> Test this with tabs early. It works cleanly in development with separate windows and breaks on a real user's tabbed setup.

## 6. Device firmware spec

- **Protocol**: newline-delimited JSON over CDC-ACM. Host → device: session table updates. Device → host: button events.
- **Session table**: keyed by `session_id`, capacity ~8, authoritative from the daemon (no device-side expiry needed, since the daemon reports death explicitly).
- **Screen** (128×64 OLED): model name large; effort badge; project name; `2/3` position marker; distinct state glyph for *working* / *waiting for approval* / *idle*.
- **Buttons**: next/prev session; long-press pins a session (disabling auto-follow) or brings that terminal to the front. Debounce ~50 ms.
- **Mass storage partition**: read-only, contains the setup file and a README. Never written by the device at runtime.

## 7. Hardware and BOM

**ESP32-S3** or **RP2040** — both support USB composite (CDC + MSC) natively. RP2040 is cheaper and has excellent USB tooling; ESP32-S3 leaves the door open for the WiFi SKU on the same firmware base.

| Part | Indicative unit cost (prototype) |
| --- | --- |
| RP2040 or ESP32-S3 module | ~$4–6 |
| SSD1306 128×64 I²C OLED | ~$4 |
| 2× tactile buttons | <$1 |
| USB-C connector, PCB, enclosure | ~$5–10 |
| **Total** | **~$15–20** at prototype volumes |

Wiring: OLED over I²C (two signal pins, e.g. GPIO21/22), buttons to GPIO with internal pull-ups, power from the USB cable.

## 8. Commercial checklist

- **USB VID/PID**: own VID from USB-IF (~$6k) or a sublicensed PID from pid.codes.
- **EV code-signing certificate** (~$300–500/yr): mandatory. A daemon that hooks window events and inspects process trees *will* trip SmartScreen and EDR heuristics without it. Keep the binary small, single-purpose, and unobfuscated; consider submitting to AV vendors for whitelisting.
- **Certification**: no intentional radiator on the USB SKU → substantially cheaper FCC/CE path than any wireless option.
- **Installer must wrap, not clobber, an existing statusline.** Claude Code allows only **one** statusline command. Many developers already have one. Silently overwriting it is a far more likely product failure than any hardware decision.

## 9. Open questions

- `model.display_name` omits the `(1M context)` marker; infer from `context_window.context_window_size`.
- `effort` is absent on models without effort support — omit the badge rather than showing a default.
- Whether "waiting for approval" can be derived reliably from available signals, or needs process/pty inspection.
- Linux/Wayland cannot support focus-following; that platform degrades to most-recently-active.
- Whether buttons should be able to *change* model/effort by driving the session, not just display it.

## 10. Next steps

1. Desktop simulator: daemon + fake screen window, validating the protocol and the tab-matching heuristic before hardware exists.
2. Firmware on a dev board (§6), then host daemon (§4).
3. Enclosure, VID/PID, signing, certification.
