<div align="center">

# Backdoor HTTPS Attack

### *I built a Windows backdoor so I could learn to catch one.*

A red-team intrusion run end to end in an isolated lab — then turned around and
read as the detection story a defender would see.

[**Live write-up →**](#) · [GitHub](https://github.com/Nikhith-ni7) · [LinkedIn](https://www.linkedin.com/in/sai-nikhith-atmakuri-204b501b1)

</div>

---

## The idea

Here's the thing nobody tells you when you're studying for a SOC role: you spend all your
time learning what an alert is *supposed* to look like, and almost none of it watching the
thing that sets the alert off. You memorise the shape of the shadow without ever meeting the
object casting it.

So I went and met the object.

I built a working Windows backdoor — generated it, slipped it past antivirus, ran it, caught
the callback, and climbed all the way to `SYSTEM`. Not to break anything. To stand on the
attacker's side of a single event, and then walk around to the other side and ask: *what did
that just look like to whoever's watching?*

That walk around to the other side is the whole project. Everything red I did, I held up and
read back as blue.

---

## Setting the stage

Two virtual machines on my own hardware: a **Kali** box playing attacker, a **Windows 11**
box playing victim, both sealed on an isolated VMware network with no road back to my real
home LAN. Snapshot taken before anything detonated, so the whole thing could be wound back to
clean when I was done.

I chose the payload on purpose. `windows/meterpreter/reverse_https` isn't the loud one — it's
the *quiet* one. It phones home over TLS, so the traffic wears the costume of ordinary
encrypted web browsing, and it beacons back in little pulses instead of holding an obvious
door open. Which means a defender can't just read what's inside it. They have to catch it by
how it *behaves*. I wanted to know that behaviour from the inside.

---

## How it went down

**The payload.** I built the executable with `msfvenom`. (First time, I forgot to give it a
format or an output file and it sprayed 575 bytes of raw shellcode across my terminal as
gibberish — which, weirdly, was the most useful thing that could've happened. It made the
difference between *the payload* and *its container* finally click. The `.exe` is just those
same bytes in a Windows wrapper that makes a double-click run them.)

**The trap.** A payload that nobody catches is useless, so I set up a listener with the exact
same payload baked into the exe — mismatch it and the callback never lands — and served the
file from a quiet Apache server for the target to grab.

**The wall.** This is where it stopped being a script and started being a fight. An unencoded
Meterpreter payload is *trivially* caught by signature antivirus, so Windows Defender flagged
and deleted my exe the instant it tried to touch disk. Rather than wrestle the protection
switches, I went into **Virus & threat protection settings**, scrolled to the option near the
bottom — **Add or remove exclusions** — and told Defender to look the other way for one
specific folder. Then I dropped the exe into that blind spot, ran it, and watched it sail
straight through. The lesson landed harder than the win: real intruders almost never "turn off
the antivirus." They carve out one narrow exception and live inside it.

**The callback.** The session opened. The target reached back across the wire to my Kali box
over "HTTPS," and `sysinfo` confirmed the machine. That's the moment the loop closes — the
backdoor calling home. (It got flaky right after, commands timing out, because the 32-bit
payload was running inside a 32-bit process on a 64-bit machine — fragile by nature. That's
exactly *why* operators migrate to a sturdier process the second they land. I learned it by
feeling the wobble myself.)

**The climb.** One command — `getsystem` — and Windows handed me the keys. It succeeded via
**EfsPotato**, which coerces the system into authenticating to itself over a named pipe, steals
that `SYSTEM` token, and wears it. No password. Initial user to highest-privilege account on
the box. The fact that it *worked* isn't a trophy — it's a finding. Microsoft has patched most
of the Potato family, so a successful EfsPotato is really a sentence about how far behind the
target's updates are.

---

## And then, the other side

This is the part that matters for the work I actually want to do. Everything above, read back
from a SOC console:

| What I did (red) | What it looks like (blue) |
|---|---|
| Dropped an unsigned exe and ran it | A ~72 KB unsigned binary writes to disk and *immediately* starts talking to the network — a process that has no business online suddenly is (Security 4688 / Sysmon 1) |
| Added a Defender exclusion | A new exclusion / protected-folder carve-out appears — a loud, high-fidelity state change defenders hunt for directly |
| Beaconed home over TLS | Regular, low-jitter callbacks to a non-standard HTTPS port (`8080`) — you can't read the payload, but the *rhythm* gives it away |
| Climbed with EfsPotato | Named-pipe artifacts (Sysmon 17/18) and an EFSRPC footprint — a process suddenly holding a `SYSTEM` token it was never given |

That's the other side of it. I can't un-know it now. When an alert fires on an unsigned binary
beaconing to an odd TLS port, I'm not looking at an abstraction anymore — I know what's on the
other end, why the port is strange, why the content is sealed. That's the entire point of
having built the thing.

---

## The technical record

<details>
<summary><b>Lab environment</b></summary>

| | |
|---|---|
| Attacker | Kali Linux (VMware Fusion, Apple Silicon) — `192.168.81.128` |
| Target | Windows 11 VM — `192.168.81.129` |
| Network | Isolated VMware segment, **no route to my real LAN** |
| Tooling | Metasploit / msfvenom, Apache (staging) |
| C2 | HTTPS reverse handler on `8080` |

</details>

**MITRE ATT&CK techniques exercised**

| Tactic | Technique | ID |
|---|---|---|
| Execution | Command & Scripting Interpreter | T1059 |
| Command & Control | Ingress Tool Transfer | T1105 |
| Command & Control | Application Layer Protocol: Web Protocols | T1071.001 |
| Command & Control | Encrypted Channel: Asymmetric Cryptography | T1573.002 |
| Command & Control | Non-Standard Port | T1571 |
| Privilege Escalation | Access Token Manipulation | T1134 |
| Privilege Escalation | Exploitation for Privilege Escalation (EfsPotato) | T1068 |
| Defense Evasion | Process Injection (migrate) | T1055 |

---

## Where it goes next

The natural sequel writes itself: run the same payload again, this time with an **encoder**
applied (obfuscation — T1027), and capture both versions side by side in **Wazuh**. A clean
before-and-after on exactly what signature-based detection catches and what it sleeps through.
That's the next thing to run, capture, and read from both sides.

---

## Ethics & scope

Every action here happened against **virtual machines I own**, on an **isolated lab network**
with no route to any production or third-party system, for **education and detection
research**. Nothing in this repository is a deployable tool — it's a write-up of authorized,
self-contained lab work, and the environment was reverted to a clean snapshot at the end.

<div align="center">

—

**Sai Nikhith Atmakuri** · M.S. Computer Science · aspiring SOC / Security Analyst

</div>
