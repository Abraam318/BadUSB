# BadUSB / HID Reverse Shell

## What This Is

A proof-of-concept demonstrating how a **USB HID device** (Digispark ATtiny85) can act as a fake keyboard to execute a payload on a Windows machine — bypassing USB storage restrictions entirely.

This repo contains the **structure and documentation**. It does not hand you a ready-made attack. You need to understand every component to build it yourself — that's intentional.

---

## Why HID Bypasses USB Policies

| Device Type | OS sees it as | Needs user permission? | Blocked by USB storage policy? |
|---|---|---|---|
| Flash drive | Storage | Yes (open file, run, allow) | Yes |
| HID device | Keyboard | No | No |

The OS trusts keyboards unconditionally. No policy blocks keyboard input by default — that's the entire attack surface.

---

## Execution Flow (High Level)

```
[Digispark plugged in]
        ↓
[Win+R → PowerShell one-liner typed as keystrokes]
        ↓
[PowerShell makes outbound HTTP request to attacker's server]
        ↓
[Script returned as a plain string → executed in memory]
        ↓
[Reverse shell opens + persistence installed]
```

### Why HTTP instead of a direct shell?

The payload is **not embedded in the device** and **not written to disk**.

Instead, PowerShell makes a normal outbound HTTP request (`Invoke-RestMethod`) to fetch the script as a string, then executes it in memory (`Invoke-Expression`). To the network:

- It looks like a normal HTTP GET request
- No suspicious binary, no dropped file, no direct socket from the device itself

This is the "fileless" approach — the script lives in memory only.

---

## Persistence Mechanisms

Three layers are installed before the shell connects:

1. **Registry Run key** — triggers on user logon
2. **Scheduled Task (logon)** — secondary logon trigger
3. **Scheduled Task (interval)** — reconnects every 10 minutes if the session drops

Persistence is the **most detectable layer**. Some AV/EDR catches it, some doesn't — depends entirely on the endpoint and its configuration.

The reverse shell traffic itself is significantly harder to flag because it blends with normal outbound HTTP.

---

## Detection Surface (Blue Team View)

This is the educational core. Here's what a defender actually sees:

| Layer | Observable |
|---|---|
| Process creation (Event 4688 / Sysmon 1) | `powershell.exe` with `-nop -ep Bypass` flags |
| AMSI | Full script content scanned at runtime before execution |
| Script Block Logging (Event 4104) | Deobfuscated script, verbatim |
| Network (Sysmon 3) | Outbound HTTP to attacker's server |
| Registry (Sysmon 12/13) | Write to `CurrentVersion\Run` |
| Scheduled Tasks | New tasks created at runtime |

**Key point:** obfuscating the script file gains you nothing. AMSI and Script Block Logging see the deobfuscated content at execution time regardless. The real detection points are **behavior** — the command-line flags, the Run key write, and the new scheduled tasks.

---

## Lab Setup Requirements

- Windows VM (take a snapshot before testing)
- Attacker machine on the same network
- HTTP server to serve the script (e.g. `python3 -m http.server`)
- Reverse shell listener (e.g. `nc -lvnp 4444`)
- Digispark flashed with the sketch
- Arduino CLI + digistump:avr core for compiling

Configure your attacker IP and ports in both the `.ino` and the `.ps1` before compiling. Changing them requires re-flashing the board.

---

## File Structure

```
├── ReverseShell-Option1.ino   # Digispark sketch — keystrokes only, no data read-back
├── x.ps1                      # Payload: persistence + shell (served over HTTP)
├── remove.ps1                 # Removes all persistence artifacts
└── build/                     # Compiler output — .hex is what gets flashed
```

---

## Removal

Run `remove.ps1` on the target to delete the Run key and both scheduled tasks. After removal, no auto-reconnect occurs.

---

## Limitations

- Requires **physical access** (~3–4 seconds)
- Fails if PowerShell execution is locked down at the enterprise level
- Persistence detection varies — enterprise EDR is a different story from consumer AV
- Not a full PTY — basic command loop only

---

## References

- [USB HID Specification](https://www.usb.org/hid)
- [MITRE ATT&CK T1091 — Replication Through Removable Media](https://attack.mitre.org/techniques/T1091/)
- [MITRE ATT&CK T1059.001 — PowerShell](https://attack.mitre.org/techniques/T1059/001/)
- [MITRE ATT&CK T1053.005 — Scheduled Task](https://attack.mitre.org/techniques/T1053/005/)
- [AMSI Overview — Microsoft Docs](https://learn.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal)
- [Script Block Logging (Event 4104)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging_windows)

---

*For authorized security research and education only.*
