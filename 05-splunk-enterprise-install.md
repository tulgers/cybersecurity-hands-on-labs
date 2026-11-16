# Lab 05 — Installing Splunk Enterprise on My Own Linux VM

**Date:** 2026-11-13 (evening) — install and first start; first login/search to be confirmed when run
**Course:** EC-Council Certified SOC Analyst (CSA) — Log Management / SIEM practice
**Environment:** Oracle VirtualBox — "Splunk VS" VM, Oracle Linux (RHEL family), installed by me
**Target — only this one:** my own virtual machine, Splunk on `localhost` only

> Status: 🟢 **Installed, running, and signed in (2026-11-13 evening).** Extraction to `/opt/splunk` completed, Splunk started successfully, and I signed in at `http://localhost:8000` as Administrator ("Hello, Administrator"). My first SPL search was written correctly but Splunk paused it because the VM disk had only 2.2 GB free (below its 5 GB pause threshold); I lowered the pause threshold in Server settings → General settings from 5000 to 1000 and started a restart, which was still pending when the laptop battery ran low — so the search result is to be confirmed at the Saturday lab block. The output sections below stay intentionally BLANK until I paste them from my own screen: nothing is pre-filled or invented. My Splunk admin password lives only in my physical notebook — it is **not** in this file and was never shared in chat.

## 1. Objective

Get a real Splunk Enterprise running on my own lab VM so I can practise SPL and log analysis the way a SOC analyst uses a SIEM — instead of only reading about Splunk in course slides.

## 2. What was actually wrong before this lab

The VM was named "Splunk VS", but Splunk had never been installed on it:

- The old `splunk.tgz` in my home folder was **0 bytes** — an empty file dated 20 August 2025 whose download never actually started. Extracting it failed with `unexpected end of file`.
- Lesson: a file's name proves nothing; check its size first (`ls -lh`). A real Splunk Enterprise `.tgz` is about **1.7 GB**.

## 3. Steps I ran — in order

1. **Fresh download.** In Firefox on the VM, from splunk.com's Splunk Enterprise download page, the Linux **.tgz** build: **Splunk Enterprise 10.6.1.1, 1711.85 MB**, saved to `~/Downloads`. (The free trial converts to the Free licence after 60 days — plenty for lab practice.)
2. **Extract to /opt:**
   ```bash
   cd Downloads
   sudo tar -zxvf splunk* -C /opt
   ```
   The `splunk*` wildcard let the shell complete the long filename itself. Extraction finished and the prompt returned — Splunk was on disk under `/opt/splunk`.
3. **First start** (from inside the program folder, to avoid typing the long path):
   ```bash
   cd /opt/splunk/bin
   sudo ./splunk start --accept-license --run-as-root
   ```
   - Without `--run-as-root`, Splunk 10.x refuses to start as root and explains why — on a disposable lab VM, running as root is an accepted, deliberate choice.
   - The licence text pages on screen: `q` did not exit the pager here; holding **Space** scrolled to the end, then `y` + Enter accepted.
   - At the prompts I created the administrator account: username `admin`, and a new Splunk admin password that I chose myself and wrote **only in my notebook**.
4. **Startup sequence observed:** Splunk generated its certificates, then sat at "Waiting for web server at http://127.0.0.1:8000" with progress dots while the web interface came up.

**Confirmation still to paste from my own screen (do not type from memory):**

```
(PASTE: the "web interface is available" line, and the Firefox login at http://localhost:8000)
```

**What actually happened on the first starts (2026-11-13 evening):** the first attempts kept dropping `splunkd` because the flag was typed with the British spelling `--accept-licence` (with a **c**) — Splunk does not recognise that flag. Starting **without** the licence flag brought Splunk up in one attempt:

```bash
cd /opt/splunk/bin
sudo ./splunk start --run-as-root
```

Lesson kept: when a documented flag keeps failing, check its spelling character by character — and removing the optional flag can be cleaner than retyping it.

## 4. First search — attempted 2026-11-13, result to confirm Saturday

```spl
index=_internal | stats count by sourcetype
```

- `index=_internal` = Splunk's own internal logs — a safe, built-in data set that exists on every install, so the first SPL search works before I onboard any real log source.
- **Attempt result (2026-11-13 evening):** the query was typed correctly in Search & Reporting, but Splunk paused the search for disk safety — the VM disk is 17 GB with only 2.2 GB free, below the 5 GB minimum. Fix applied: the "Pause indexing if free disk space falls below" threshold was lowered from 5000 to 1000 in Server settings → General settings and a restart was started; it had not finished when the laptop battery ran low. Permanent fix (enlarging the VM disk) is a Saturday task. Saturday step 1: confirm Splunk is up (`sudo ./splunk status` from `/opt/splunk/bin`, start it with `sudo ./splunk start --run-as-root` if not), then re-run this search.
- **My output:**

```
(PASTE REAL OUTPUT HERE)
```

## 5. Typing lessons (kept on purpose — same lesson as Lab 04)

- Every separate word takes a space; a path takes none around its slashes: `/opt/splunk/bin/splunk` is one word, `sudo` + space + `./splunk` + space + `start` + space + `--accept-license` is four.
- My slips tonight were all missing spaces or Enter pressed where Tab was meant — the shell then honestly reports that the (wrong) command does not exist. Nothing was damaged in any of them.

## 6. SOC view — why a defender cares

- **This is the SIEM deployment order the CSA exam tests:** install → first start / admin account → web interface → onboard log sources into indexes → search with SPL. I have now done the first three steps with my own hands.
- An admin password for a SIEM is a high-value credential: unique, written in my notebook only, never pasted into chat or a write-up.
- Splunk's `_internal` index is also a detection source in real SOC work: failed logins to Splunk Web and configuration changes show up there first.

---
*Install run 2026-11-13 evening, guided photo-by-photo, on the same "Splunk VS" VM recovered in Lab 04 (GRUB `rd.break` password reset, earlier the same evening). Web login confirmed the same evening (Administrator). Next (Saturday lab block): confirm the restart finished, re-run the first `_internal` search above, and screenshot the result table for this sheet.*
