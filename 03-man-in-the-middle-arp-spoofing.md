# Lab 03 — Man-in-the-Middle: ARP Spoofing in the Virtual Lab

**Date:** Started 2026-11-14 (evening, Toronto) — in progress
**Course:** EC-Council Certified SOC Analyst (CSA), Module 02 — Network Attack TTPs
**MITRE ATT&CK:** **T1557 Adversary-in-the-Middle** (core-list ID — notebook entry, 2026-11-12)
**Environment:** Oracle VirtualBox — Kali Linux (attacker) + Cybersecurity LabVM Workstation (victim), both on the same VirtualBox **NAT Network**. Home network is never touched.
**Targets — only these:** my own two lab VMs, and a fake test login typed with INVENTED credentials (never a real password).

> Status: 🔄 **In progress — started 2026-11-14 evening.** Sophia chose to run MitM first, ahead of Splunk and Lab 02. Pre-flight and addressing are DONE (see §0, filled from her real screenshots). §3.1 IP forwarding was also done: enabled on Kali (output `1`, understood, then at Sophia's request turned back OFF — output `0` — so nothing is left open on her computer). The lab paused there, before any ARP spoofing: no poisoning was ever started. Next step when the session resumes: §3.2 ARP poisoning (turn forwarding back on first). Remaining output sections are intentionally BLANK — like Labs 01–02, everything in this portfolio is real output from my own run; nothing is pre-filled or invented.

## Weekend block agenda (this lab runs second, after Lab 02)

1. **~10 min — Notebook re-read:** this week's ✍️ entries (CSA exam format/weights, banner grabbing, DNS footprinting concept, MitM, core MITRE list T1566/T1110/T1078/T1557/T1059/T1003/T1486).
2. **~15 min — Lab 02 banner grabbing** (its own sheet: `nmap -sV -p 22,80,9929 scanme.nmap.org` + `curl -I`).
3. **~45 min — This lab (Lab 03 MitM).**
4. **~20 min — GitHub write-ups** for both labs in the same session (soc-labs, tulgers).

## 0. Pre-flight — the one unknown that can sink this lab

- [x] **Workstation login check — PASSED 2026-11-13 evening:** Sophia started the **Cybersecurity LabVM Workstation** VM and reached the desktop using the SOC Analyst (CyberOps Associate) account shown on the lab machine's own login screen. The forgotten password is the **Splunk VS** VM only; Kali's login is fine. A system "update available" prompt was dismissed on purpose — lab VMs are not updated before a lab. This lab is UNBLOCKED for the Saturday 2026-11-14 weekend block.
- [x] **Both VMs on the same NAT Network — DONE 2026-11-14:** a NAT network named `NatNetwork` was created in VirtualBox Network Manager (the planned name `SocLabNet` was not used), and both Kali and the Cybersecurity LabVM Workstation were set to Attached to: NAT Network / Name: NatNetwork, then started. Home network untouched.
- [x] Addresses — filled 2026-11-14 from Sophia's real terminal screenshots:
  - Kali IP: `ip a` → **10.0.2.15**, interface **eth0** (attacker — "Ben Kali'yim")
  - Gateway IP: `ip route` → **10.0.2.1** (default via 10.0.2.1 dev eth0)
  - Victim (Workstation) IP: `ip a` on the Workstation → **10.0.2.3** (`inet 10.0.2.3/24` on enp0s3, MAC 08:00:27:55:44:07 — matches the .3 host in Kali's `nmap -sn 10.0.2.0/24` scan, which found 4 hosts up: .1, .2, .3, .15)
  - Notebook clarifications from this session: `inet` = the line that carries the IP address; `lo` = loopback (localhost, 127.0.0.1, the machine talking to itself) — not LAN; LAN = the local network itself (here 10.0.2.0/24); `nmap -sn` = ping scan only, no port scan (👀 lab-only, not an exam item). In a real attack the victim IP is found by scanning (`nmap -sn`), not by running `ip a` on the victim — the lab used `ip a` on the Workstation only to confirm .3 vs .2.

## 1. Objective

In the lesson (2026-11-12) we learned the theory: the attacker does not replace the server — he replaces **the road**. He poisons both ARP tables, splits the one TCP connection into two (victim↔attacker, attacker↔server), and **must keep the traffic flowing (IP forwarding)** or the attack collapses and exposes itself. This lab makes that visible: the victim browses normally and sees the real page, while the attacker reads a copy of everything sent over plain HTTP — and then sees nothing readable over HTTPS.

## 2. Attacker's view — step by step

1. **Poison both ARP tables.** Victim is told "the gateway's MAC is the attacker's MAC"; the gateway is told "the victim's MAC is the attacker's MAC." Nobody's MAC is copied — the attacker writes **his own MAC** into both tables under other people's IPs.
2. **Turn on IP forwarding.** One switch: "a packet that arrives for someone else — forward it, don't drop it." Off = packets die at Kali, victim's page hangs, attack fails (this is the check question Sophia answered correctly on 2026-11-12).
3. **Watch a copy go past.** Victim logs into a plain-HTTP test page with INVENTED credentials; Wireshark on Kali shows them in clear text — the lesson's "read" moment, live.
4. **Repeat over HTTPS.** Same capture, encrypted site: only TLS records — readable by nobody in the middle. That contrast IS the lesson.
5. **Clean up.** Stop the spoofing (arpspoof restores the real ARP entries), turn forwarding back off, confirm the victim browses normally.

## 3. Commands to run (in order — Kali terminal)

### 3.1 Enable IP forwarding

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### 3.2 Start ARP poisoning (two terminals, both left running)

Terminal 1 — lie to the victim ("I am the gateway"):
```bash
sudo arpspoof -i eth0 -t 10.0.2.3 10.0.2.1
```
Terminal 2 — lie to the gateway ("I am the victim"):
```bash
sudo arpspoof -i eth0 -t 10.0.2.1 10.0.2.3
```
- Filled with the real 2026-11-14 addresses from §0: victim = 10.0.2.3, gateway = 10.0.2.1, Kali interface = eth0. If the VMs are restarted and DHCP gives different addresses, re-check with `ip a` / `nmap -sn` before running these.
- If `arpspoof` is missing: `sudo apt install dsniff` (Kali usually ships it).

### 3.3 Capture

Start **Wireshark** on Kali (interface = the same one as above), display filter:
```
http
```
Then on the **Workstation (victim)**: browse to a plain-HTTP test login page (e.g. the public test site `http://testphp.vulnweb.com/login.php` — a site published for security testing) and log in with INVENTED credentials (e.g. user `labtest`, password `Ornek123!` — invented for this lab, used nowhere else).

**What I saw in Wireshark (paste/summarise real capture — look for the POST line with the invented username/password in clear text):**

```
(PASTE REAL FINDING HERE)
```

### 3.4 The HTTPS comparison

Victim now opens an HTTPS site and logs in (test account only). In Wireshark change the filter to:
```
tls
```
**What changed (in my own words):**

```
(FILL IN AFTER THE RUN — expected: connection visible, content unreadable)
```

### 3.5 Clean-up (always, before closing the VMs)

- Ctrl+C in both arpspoof terminals (arpspoof sends corrective ARP replies).
- Turn forwarding off:
```bash
echo 0 | sudo tee /proc/sys/net/ipv4/ip_forward
```
- [ ] Victim still browses normally afterwards: yes / no ______

## 4. SOC view — how a defender catches this

- **ARP anomalies:** one IP's MAC address changing repeatedly, or the gateway IP suddenly sharing a MAC with a workstation = classic spoofing signal.
- **Tools:** dynamic ARP inspection (DAI) on switches, ARPwatch-style monitoring, endpoint alerts on ARP table changes.
- **User reports:** browser certificate warnings on HTTPS sites + "the internet got slow" are real-world MitM tells.
- **Sentinel / SIEM question:** is there a log source that would even see this? Layer-2 attacks are mostly invisible to a cloud SIEM — detection lives on the network/endpoint side; a SOC analyst must know which layer each attack is visible at.

## 5. MITRE mapping

| Concept | ID | What it means |
|---|---|---|
| Adversary-in-the-Middle | T1557 | Attacker sits between two parties, relaying — and able to read/modify/insert |

## 6. Scope & ethics

Only my own two lab VMs on an isolated VirtualBox NAT Network. Credentials typed in this lab are invented for the lab. Never ARP-spoof a network you do not own — on a real network this is an attack, full stop.

---
*Theory source: 2026-11-12 lesson — TCP from the ground up, "split the TCP connection" = one call replaced by two, ARP poisoning + IP forwarding mechanics, HTTPS contrast. Lab plan agreed with Sophia 2026-11-12 (Kali = attacker, Workstation = victim, same NAT Network, GitHub write-up after). Moved to the weekend lab block the same day under the new lab-scheduling decision (weekday lessons uninterrupted; labs batched at the weekend).*
