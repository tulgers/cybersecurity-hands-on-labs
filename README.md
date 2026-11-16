# SOC Labs — Hands-on Security Operations Portfolio

**Sureyya (Sophia) Ozok** — SOC Analyst (2023–2026) · EC-Council Certified SOC Analyst (CSA) student

Every lab in this repo is something I ran myself, in my own lab, and wrote up the way I would triage it on shift: what the attacker does, what it looks like in the logs, and what a SOC analyst does about it.

## How I learn

Simple first, then technical depth — and always from the **attacker's perspective**, step by step, so the detection makes sense. Key terms are given in English, with the Turkish term alongside where it helps me learn.

## Lab environment

- **Kali Linux** in Oracle VirtualBox (login works — no password issue here)
- **Targets — only these, ever:** my own lab VMs / localhost (`127.0.0.1`), my own home network, and Nmap's official practice host **`scanme.nmap.org`**
- I never scan anyone else's system without written permission. Active scanning leaves traces in logs — that is the point of the lab, and the reason the scope stays inside my own lab.

## Labs

| # | Lab | MITRE ATT&CK | Status |
|---|-----|--------------|--------|
| 01 | [Reconnaissance — Nmap Port Scanning & Ping Sweep](labs/01-reconnaissance-nmap-port-scan.md) | TA0043 Reconnaissance · T1595 Active Scanning | ✅ Complete (2026-11-10) — scanme.nmap.org: 3 open ports (22/ssh, 80/http, 9929/nping-echo), 963 closed, 34 filtered |
| 02 | [Banner Grabbing — Service & Version Detection](labs/02-banner-grabbing-service-detection.md) | TA0043 Reconnaissance · T1595 Active Scanning | ✅ Complete (2026-11-14) — `nmap -sV` found OpenSSH 6.6.1p1 (22) and Apache 2.4.7 (80); `curl -I` confirmed the same Apache banner |
| 03 | [Man-in-the-Middle — ARP Spoofing in the Virtual Lab](labs/03-man-in-the-middle-arp-spoofing.md) | T1557 Adversary-in-the-Middle | 🔄 In progress — setup complete 2026-11-14 (NatNetwork created, IP map confirmed: gateway 10.0.2.1, victim 10.0.2.3, Kali 10.0.2.15; IP forwarding tested on, then turned back off). ARP spoofing itself not yet run |
| 04 | [Linux Password Recovery — GRUB `rd.break`](labs/04-linux-password-recovery-grub-rdbreak.md) | — (recovery on my own VM) | ✅ Complete (2026-11-13) — forgotten Splunk VS password reset by me via GRUB `rd.break`, desktop login confirmed |
| 05 | [Installing Splunk Enterprise on My Own Linux VM](labs/05-splunk-enterprise-install.md) | — (SIEM deployment / Log Management) | 🔄 Mostly complete — Splunk Enterprise 10.6.1.1 installed, running and signed in (2026-11-13); first SPL search (`index=_internal \| stats count by sourcetype`) result still to confirm and paste from my own screen |

## Structure of each lab

1. **Objective** — what I wanted to learn, in one sentence
2. **Attacker's view** — what the attacker does, concretely, step by step
3. **Commands & output** — exactly what I ran, and what came back
4. **What I learned** — the finding, in plain language
5. **SOC view** — how this shows up in logs, and how I would triage it
6. **MITRE mapping** — tactic and technique, with IDs

## Tools I use

Microsoft Sentinel · Splunk · IBM QRadar · KQL / SPL · Nmap · Shodan (passive checks only) · Microsoft Entra ID / Purview DLP
