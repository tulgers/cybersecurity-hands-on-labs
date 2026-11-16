# Lab 02 — Banner Grabbing: Service & Version Detection

**Date:** 2026-11-14 (evening)
**Course:** EC-Council Certified SOC Analyst (CSA), Module 02 — Understanding Cyber Threats, IoCs, and Attack Methodology
**MITRE ATT&CK:** TA0043 Reconnaissance · **T1595 Active Scanning** (T1595.002 Vulnerability Scanning)
**Environment:** Kali Linux in Oracle VirtualBox
**Target — only this one:** `scanme.nmap.org` (45.33.32.156), Nmap's official practice host

> Status: ✅ **Complete — run 2026-11-14 evening.** Both outputs below are my real Kali terminal output from that run. Before this lab I re-ran the full `nmap scanme.nmap.org` scan as a refresher (21:23–22:11); it reached 99.99% but stalled in the retransmission tail on the filtered ports — the same behaviour that made Lab 01 take ~102 minutes — so I stopped it with Ctrl+C and worked from Lab 01's confirmed open ports, exactly as this sheet planned.

## 1. Objective

Lab 01 found **which doors are open** on scanme.nmap.org: 22/ssh, 80/http, 9929/nping-echo. This lab answers the attacker's next question: **what exactly is behind each door — which service, which version?** That is banner grabbing (banner = the name tag a service shows when you knock).

Why the version matters: a version number is a lookup key. "Apache 2.4.41" → search that version's known vulnerabilities (CVEs) → the attacker now knows whether this door is worth attacking, and how.

## 2. Attacker's view — step by step

1. **Knock politely, read the name tag.** Connect to the open port; most services introduce themselves ("SSH-2.0-OpenSSH_…", "Server: Apache/…"). The attacker does not break anything yet — just reads.
2. **Write down service + version for every open port.** 22 → SSH daemon and its version; 80 → web server and its version; 9929 → nping-echo (Nmap's own test service on scanme — expected, not a real target service).
3. **Look up the versions.** Each version is checked against public CVE databases. An old, unpatched version tells the attacker exactly which exploit to try.
4. **Pick the weakest door.** Web services (port 80) are usually the softest target: login forms, /admin and /login paths, injection points. SSH is usually hard — unless its version carries a known flaw.
5. **SOC gözüyle (defender's note):** every one of these probes is a log line — a version probe looks different from normal traffic, and repeated probing from one source IP is exactly the pattern a SOC analyst hunts.

## 3. Commands to run (in order)

### 3.1 Version detection on only the 3 known open ports

Lab 01's full scan took ~102 minutes because of the 34 filtered ports. This run touches only the 3 open ports, so it should finish in seconds:

```bash
nmap -sV -p 22,80,9929 scanme.nmap.org
```

- `-sV` = service/version detection: Nmap sends probes and matches the replies against its fingerprint database.
- `-p 22,80,9929` = only these ports — no re-scanning of the 963 closed / 34 filtered ones.

**My output (real, from my Kali terminal, 2026-11-14 22:12 EST):**

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-14 22:12 -0500
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.047s latency).
Other addresses for scanme.nmap.org (not scanned): 2600:3c01::f03c:91ff:fe18:bb2f

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http       Apache httpd 2.4.7 ((Ubuntu))
9929/tcp  open  nping-echo Nping echo
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.40 seconds
```

### 3.2 The web server's name tag, the simple way

```bash
curl -I http://scanme.nmap.org
```

- `curl -I` = fetch only the HTTP headers (the "name tag" page of the reply), not the page itself.
- Look for the `Server:` line — that is the banner, handed over freely to anyone who asks.

**My output (real, 2026-11-14 22:23 EST):**

```
HTTP/1.1 200 OK
Date: Sun, 15 Nov 2026 03:23:16 GMT
Server: Apache/2.4.7 (Ubuntu)
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Type: text/html
```

The `Server:` header matches the `-sV` result exactly (Apache 2.4.7 on Ubuntu) — two different tools, same banner, cross-verified.

### 3.3 (Optional, only if 3.1 is unclear) Aggressive version probe

```bash
nmap -sV --version-intensity 5 -p 80 scanme.nmap.org
```

Higher intensity = more probes = a more confident fingerprint — and more noise in the target's logs. On a real engagement this is a trade-off; on scanme it is safe practice.

## 4. What I learned (filled in after the run, 2026-11-14)

- Port 22 is running: **OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (protocol 2.0)**
- Port 80 is running: **Apache httpd 2.4.7 (Ubuntu)** — and yes, the `Server:` header in 3.2 matches it exactly: `Apache/2.4.7 (Ubuntu)`.
- Port 9929 is: nping-echo — Nmap's own test service, present on scanme by design; on a real host an unknown service on an unusual port would be a question to escalate, not ignore.
- One version I looked up afterwards, and whether a known CVE exists for it: both banners are roughly a decade old (OpenSSH 6.6.x and Apache 2.4.7 are 2013–2014-era; current lines are OpenSSH 10.x and Apache 2.4.6x). The professional check I learned: don't judge age from memory — compare the banner against the vendor's current release, then search that exact version in NVD (nvd.nist.gov). Version lines this old carry long public CVE lists; on a production host this finding would be raised as outdated/unsupported software. On scanme it is deliberate — a safe target with something interesting to find.

## 5. SOC view — how a defender catches this

- **IDS / WAF:** version probes carry distinctive probe strings; a burst of them from one source IP in seconds is scanning behaviour, not user traffic.
- **Web logs:** `curl -I`-style header-only requests, plus probes of /admin, /login, /.git — one IP doing this in sequence is mapping the site.
- **Sentinel / SIEM triage question:** is this source a known, authorised scanner (our own vulnerability scanner, Shodan) or an unknown host? Known + scheduled = document and close. Unknown + version-probing right after a port scan = same attacker, next kill-chain step — escalate and watch that IP for exploitation attempts.

## 6. MITRE mapping

| Concept | ID | What it means |
|---|---|---|
| Reconnaissance (tactic — the goal) | TA0043 | Gather information to plan the attack |
| Active Scanning — Vulnerability Scanning (technique) | T1595.002 | Probe services/versions to find an exploitable weakness |

## 7. Scope & ethics

Only `scanme.nmap.org` — Nmap's official practice host — is probed in this lab. Never version-scan a system you do not own or have written permission to test.

---
*Prepared 2026-11-10 (evening) as the follow-up step proposed after Lab 01: "the hacker's next step is learning the service and version behind the open doors." Numbering note: the README's planned Lab 02 was "Tor exit-node sign-ins in Sentinel" — when this lab is completed, renumber the README table (this becomes Lab 02, Tor/Sentinel becomes Lab 03, DNS/ping sweep becomes Lab 04) or file this as Lab 01 Part 2 — Sophia's choice.*
