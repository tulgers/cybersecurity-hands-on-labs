# Lab 01 — Reconnaissance: Nmap Port Scanning & Ping Sweep

**Date:** 2026-11-10
**Course:** EC-Council Certified SOC Analyst (CSA), Module 02 — Understanding Cyber Threats, IoCs, and Attack Methodology
**MITRE ATT&CK:** TA0043 Reconnaissance · **T1595 Active Scanning** (T1595.001 Scanning IP Blocks, T1595.002 Vulnerability Scanning — see notes)
**Environment:** Kali Linux in Oracle VirtualBox

> Status: ✅ **Complete.** All terminal output in this write-up is real, taken from my Kali runs on 2026-11-10 — nothing is invented.

## 1. Objective

Learn reconnaissance (keşif) hands-on: before any exploitation, an attacker maps the target. I wanted to feel the difference between **passive** recon (no touch — e.g. looking up an IP on Shodan, which already scanned it) and **active** recon (touching the target — port scanning and pinging it), and to see what active recon looks like from the defender's side.

The slide that started this lab says it plainly: **"Exploitation begins with reconnaissance."** Catch the recon, and you can stop the attack before it starts.

## 2. Attacker's view — step by step

1. **Passive recon first — no trace.** Read the company website, find employees on LinkedIn, look up the domain, search Shodan. The attacker never touches the target, so nothing appears in the target's logs.
2. **Active recon — leaves traces.** Scan ports to find open doors, ping an IP range to find live hosts. Every probe is a log line somewhere.
3. **Build the map.** From the answers: domain name, IP blocks, running services, VPN points, authentication methods, system information. Only then does the attacker pick an entry point.

The four techniques on the Module 02 slide:

- **Social Engineering** — trick a person into giving information ("I'm from IT, can you confirm your password?")
- **Port Scanning** — knock on the doors. With Nmap: is 3389 (RDP) open? Is 22 (SSH) open? An open port is a possible entry point.
- **DNS Footprinting** — follow the domain's trail: which IPs, which subdomains (e.g. `mail.example.com`), which records exist?
- **Ping Sweeping** — ping every address in a range, one by one, to find which machines are alive.

## 3. Commands & output

### 3.1 Check the tools

```bash
nmap --version
```

```
Nmap version 7.99 ( https://nmap.org )
Platform: x86_64-pc-linux-gnu
Compiled with: liblua-5.4.8 openssl-3.6.3 libssh2-1.11.1 libz-1.3.1 libpcre2-10.46 libpcap-1.10.6 nmap-libdnet-1.18.0 ipv6
Compiled without:
Available nsock engines: epoll poll select
```

✅ Nmap 7.99 is preinstalled on Kali — no install step needed. (Run 2026-11-10, Kali terminal.)

```bash
ping -c 4 127.0.0.1
```

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.034 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.033 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.032 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.029 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3068ms
rtt min/avg/max/mdev = 0.029/0.032/0.034/0.002 ms
```

✅ 4 packets sent, 4 received, **0% packet loss** — localhost is alive. `time=0.03 ms` is near-instant because `127.0.0.1` is the machine itself. `ttl=64` is the typical Linux value. (This ping was repeated deliberately — I had tried it once before in this same Kali terminal.)

### 3.2 First active scan — safe target only

Target: **`scanme.nmap.org`** — Nmap's official practice host, provided for exactly this purpose. (My own lab VMs / localhost are the only other targets in this repo.)

```bash
nmap scanme.nmap.org
```

Full output (scan started 13:55 EST, finished ~15:37 EST):

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-10 13:55 -0500
Warning: 45.33.32.156 giving up on port because retransmission cap hit (10).
```

**What happened:** `scanme.nmap.org` resolved to IP `45.33.32.156`. The warning means Nmap knocked on a port 10 times and got no answer at all, so it gave up on that port. A port that never answers is usually **filtered** — a firewall silently drops the packets instead of replying. The scan is not broken, just slow: with 1,000 ports to check and many of them silent, each one costs up to 10 retries. Faster alternative: `nmap -F scanme.nmap.org` (fast mode — top 100 ports only).

```
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.00049s latency).
Other addresses for scanme.nmap.org (not scanned): 2600:3c01::f03c:91ff:fe18:bb2f
Not shown: 963 closed tcp ports (reset), 34 filtered tcp ports (no-response)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9929/tcp open  nping-echo

Nmap done: 1 IP address (1 host up) scanned in 6130.48 seconds
```

**Reading it:** 3 open doors — **22/ssh** (remote administration), **80/http** (web server — exactly the port 80 from the lesson), **9929/nping-echo** (Nmap's own test service, run deliberately on scanme). 963 ports were **closed** (they answered RST — "nobody here"), 34 were **filtered** (no answer at all — these caused the retransmission warnings). The scan took **6130.48 seconds (~102 minutes)** because every silent port cost up to 10 retries — this is why the fast mode (`-F`, top 100 ports) exists.

Suggested follow-up, not yet run:

```bash
nmap -sV scanme.nmap.org   # -sV: ask what service/version is behind each open port
```

## 4. What I learned

- **Open ports on scanme.nmap.org: 22/ssh, 80/http, 9929/nping-echo.** 963 ports closed (RST), 34 filtered (no response). The full 1,000-port scan took 6130.48 seconds (~102 minutes) because the filtered ports each cost up to 10 retries — the retransmission-cap warnings were the clue.
- Shodan was **passive**: I never touched the target — Shodan had already scanned it. Nmap is **active**: I touch the target, and the target's logs can see me.
- A ping sweep answers one question only — "is anyone home?" — but that answer is the first line of the attacker's map.

## 5. SOC view — how a defender catches this

Active recon is noisy, and noise is detectable:

- **IDS / firewall:** one source IP, many rejected connections in a short time = a port scan.
- **Web logs:** one IP, many `404` responses, probing sensitive paths in a short window = someone mapping the site. (This is the 404 pattern I already knew from triage practice.)
- **Sentinel / SIEM:** correlate the source IP across firewall, IDS and web logs; a single host scanning many ports is one incident, not many alerts.

**Triage question I would ask on shift:** one IP scanning — is it a known scanner (e.g. Shodan, a vulnerability scanner we run ourselves) or an unknown host with no business touching us? Known + expected = document and close; unknown + sweeping = escalate, block at the firewall if policy allows, and hunt for follow-up exploitation attempts from the same IP.

## 6. MITRE mapping

| Concept | ID | What it means |
|---|---|---|
| Reconnaissance (tactic — the attacker's goal) | TA0043 | Gather information to plan future operations |
| Active Scanning (technique — how) | T1595 | Probe the target directly: port scans, ping sweeps, vulnerability scans |

## 7. Scope & ethics

Only my own lab VMs, localhost, and `scanme.nmap.org` were scanned. Never scan a system you do not own or have written permission to test — active scanning without permission is unauthorised access in most jurisdictions, and in a SOC it is exactly the behaviour we are paid to detect.
