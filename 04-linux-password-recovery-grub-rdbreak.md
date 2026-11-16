# Lab 04 — Linux Password Recovery: GRUB `rd.break` on Oracle Linux

**Date:** 2026-11-13 (evening)
**Course:** EC-Council Certified SOC Analyst (CSA) — hands-on lab practice / Linux recovery
**Environment:** Oracle VirtualBox — "Splunk VS" VM, Oracle Linux (RHEL family), installed by me
**Target — only this one:** my own virtual machine. Username on the system: `sophiaozok` (shown on the login screen as "sophia ozok")

> Status: ✅ **Complete — run by me on 2026-11-13.** My Splunk VS VM had been unusable for days because I had forgotten its login password. This lab recovered it without reinstalling and without losing any data. The new password is written only in my physical notebook — it is **not** in this file and was never shared in chat.

## 1. Objective

Regain access to my own Linux VM whose password I had forgotten, using the standard RHEL-family recovery path: interrupt the boot in GRUB, boot into the emergency `rd.break` shell, remount the real system writable, `chroot` into it, and set a new password with `passwd`.

## 2. Attacker's / recovery view — step by step

1. **Interrupt the boot.** Restart the VM and press `e` the moment the GRUB menu appears (it only stays ~5 seconds) to edit the boot entry.
2. **Edit the kernel line.** Go to the line starting with `linux` (the `vmlinuz` line), jump to its end with Ctrl+e, delete `rhgb quiet`, and append `rd.break`. Boot with Ctrl+x.
3. **Remount the real system writable.** At the `switch_root:/#` prompt:
   ```
   mount -o remount,rw /sysroot
   ```
4. **Step inside the real system and confirm the username:**
   ```
   chroot /sysroot
   ls /home
   ```
   Output: `sophiaozok` — the real system username (the login screen only showed the display name).
5. **Set the new password:**
   ```
   passwd sophiaozok
   ```
   Result: `passwd: all authentication tokens updated successfully.` (Typed characters are invisible at the password prompt — that is normal.)
6. **Ask SELinux to relabel on next boot, then exit twice:**
   ```
   touch /.autorelabel
   exit
   exit
   ```
7. **Wait out the relabel.** The VM rebooted and ran SELinux relabeling for several minutes — do not power off during this. Afterwards the normal login screen appeared and the new password worked: desktop reached, VM rescued.

## 3. What went wrong along the way (kept on purpose — this is the real lesson)

- The GRUB menu disappears in ~5 seconds; I missed it on the first reset. Fix: click into the VM window first, finger ready on `e`.
- My only real slip was **missing spaces**: I typed `mount-o` instead of `mount -o …` and later ran `chroot`/`ls` together without a space. Linux commands do not forgive missing spaces — the shell simply says the command does not exist. Nothing was damaged either time.
- Random kernel messages (`random: crng init done`) interleaved with my typing at the emergency prompt. The keystrokes still landed correctly; the screen just looked messy.

## 4. SOC view — why a defender cares about this "recovery" trick

- **Physical / console access = game over for an unencrypted disk.** Anyone who can reach the boot menu of an unprotected Linux machine can reset the root or user password in minutes, exactly as above. This is a recovery feature *and* an attack path — the difference is only whose machine it is.
- **Defences:** GRUB/bootloader password, BIOS/UEFI password, full-disk encryption (LUKS), and physical/console access control on servers and VMs.
- **Detection angle:** unexpected reboots into emergency mode, changed password timestamps in `/etc/shadow`, and SELinux relabel events after a boot are all traces this procedure leaves behind.

## 5. What I learned

- GRUB editing, `rd.break`, `chroot` and `passwd` are one connected chain: interrupt → emergency shell → writable mount → step inside → change password → relabel → normal boot.
- The display name on a login screen is not the system username; `ls /home` inside the chroot settles it.
- A forgotten VM password is recoverable on my own machine — no reinstall, no data loss — as long as the disk is not encrypted.

---
*Run and documented 2026-11-13 evening, guided photo-by-photo. This VM is the same "Splunk VS" machine whose forgotten password had been the last blocker for the weekend lab block; with this recovery, plus the Cybersecurity LabVM Workstation login test passed the same evening, both lab machines are ready. Next on this VM: extract the downloaded `splunk.tgz` to `/opt` and start Splunk for the first time.*
