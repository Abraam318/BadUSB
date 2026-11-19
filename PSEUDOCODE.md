# How It Works — Pseudocode Reference

> This documents the logic and design decisions behind each component. Not a build guide — you still need to understand every layer to implement it yourself.

---

## 1. The Trigger (Digispark Sketch)

```
ON startup:
    wait 2 seconds        // give the OS time to recognize the device

ON loop (runs once):
    press Win + R         // open Run dialog
    wait 150ms
    type the PowerShell command
    press Enter
    wait forever          // job done, nothing left to do
```

**Why the 2-second delay?**
The OS needs time to load the HID driver. Without it, the first few keystrokes get dropped.

**Why keep the command short?**
The device types like a human — character by character. Every extra character is extra time. The command only needs to do one thing: reach out to the server.

---

## 2. The Payload Logic (served over HTTP)

```
SET attacker_ip, ports, command to re-fetch this script

// --- PERSISTENCE (installed first, before the shell) ---

WRITE to registry Run key
    → disguised name
    → value = command to re-fetch and re-run this script on logon

CREATE scheduled task (logon trigger)
    → same command
    → runs when user logs in

CREATE scheduled task (interval trigger)
    → same command
    → repeats every 10 minutes for 1 year
    → purpose: reconnect if the session dropped

// --- REVERSE SHELL ---

TRY:
    open TCP connection to attacker machine
    send hostname as banner

    LOOP:
        read command from attacker
        execute it, capture output
        send output back + current directory prompt

    on disconnect: exit loop

CATCH any error:
    exit silently        // fail quietly, persistence will reconnect
```

**Why persistence is installed before the shell opens:**
If the shell fails or gets killed, the backdoor is already in place. The 10-minute task handles reconnection automatically.

**Why HTTP instead of embedding the payload on the device:**
Running commands or scripts directly triggers AV/EDR behavioral detection immediately.
Fetching them as a plain HTTP request looks like normal network traffic — no file written, nothing on disk, no suspicious process spawning from the device itself.

---

## 3. Cleanup Logic

```
DELETE registry Run key entry
UNREGISTER scheduled task (logon)
UNREGISTER scheduled task (interval)
PRINT confirmation
```

Run this on the target after testing to remove all persistence artifacts.

---

## Design Decisions Summary

| Decision | Reason |
|---|---|
| HTTP fetch instead of embedded payload | Avoids on-device storage, blends with normal traffic |
| Persistence installed before shell | Shell failure doesn't lose access |
| Two scheduled tasks + Run key | Redundancy — different triggers, different detection profiles |
| Fail silently on shell error | Persistence handles recovery, no noise |
| Short keystroke command | Typing speed is the bottleneck on the device |
