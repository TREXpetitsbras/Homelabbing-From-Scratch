# 🔥 Firewall — OPNsense on a ThinkCentre M710q Tiny

> **Technical runbook.** This file is the condensed, reproducible *how-to*.
> The *why* (choices, trade-offs, lessons learned) lives in the journal: [Journal.md → Firewall](../Journal.md#-firewall--opnsense) (in French).

| | |
|---|---|
| **Tested on** | OPNsense 26.7.6 (FreeBSD 15.1) |
| **Hardware** | Lenovo ThinkCentre M710q Tiny — Celeron G3900T, 8 GB RAM, 250 GB NVMe |
| **WAN NIC** | UGREEN USB-A → RJ45 1 Gbps, ASIX AX88179A chip (`ue0`) |
| **LAN NIC** | Onboard Intel port (`em0`) |
| **Upstream** | Freebox Pop (router mode during preparation, bridge mode after cut-over) |

**Legend:** ✅ Done · 🕓 Planned

| # | Procedure | Status |
|---|---|---|
| 1 | [Prerequisites](#1-prerequisites) | ✅ |
| 2 | [Installation media](#2-installation-media) | ✅ |
| 3 | [BIOS settings](#3-bios-settings) | ✅ |
| 4 | [Installation](#4-installation) | ✅ |
| 5 | [Console setup](#5-console-setup) | ✅ |
| 6 | [WAN fix: AX88179A → `cdce` driver](#6-wan-fix-ax88179a--cdce-driver) | ✅ |
| 7 | [First web login](#7-first-web-login) | ✅ |
| 8 | [Snapshots & updates](#8-snapshots--updates) | ✅ |
| 9 | [Setup wizard](#9-setup-wizard) | ✅ |
| 10 | [Hardening & backup](#10-hardening--backup) | 🕓 |
| 11 | [VLANs & DHCP](#11-vlans--dhcp) | 🕓 |
| 12 | [Firewall rules](#12-firewall-rules) | 🕓 |
| 13 | [Cut-over (Freebox bridge mode)](#13-cut-over-freebox-bridge-mode) | 🕓 |
| 14 | [Validation tests](#14-validation-tests) | 🕓 |

---

## 1. Prerequisites

| Item | Note |
|---|---|
| Monitor + **DisplayPort** cable | The M710q has no HDMI by default (DP → HDMI adapter works) |
| USB keyboard | Only needed for install and console |
| USB stick ≥ 4 GB | Will be wiped |
| Admin PC with Ethernet | Plugged directly into the LAN port during setup |

**Cabling during preparation (double NAT):**

```
Freebox (router mode) ──► USB adapter [WAN · ue0] ── Tiny ── [LAN · em0] ──► Admin PC
      192.168.1.0/24                                           192.168.10.0/24
```

<!-- 📸 Image idea: photo of the back of the Tiny with both cables labelled (WAN / LAN) -->

---

## 2. Installation media

1. Download the **`vga`** image (USB installer) from the official mirror list: `OPNsense-<version>-vga-amd64.img.bz2`.
2. Download the matching `OPNsense-<version>-checksums-amd64.sha256` file from the same mirror.
3. Verify integrity (Windows PowerShell):

```powershell
(Get-FileHash .\OPNsense-26.7-vga-amd64.img.bz2 -Algorithm SHA256).Hash -eq "<official hash>"
# Expected: True
```

4. Write the image to the USB stick with **balenaEtcher** (accepts `.bz2` directly), or decompress with 7-Zip and use **Rufus**.

> A hash only proves **integrity**, not **authenticity**. For authenticity, verify the OpenSSL signature against the OPNsense public key.

---

## 3. BIOS settings

`F1` = BIOS setup · `F12` = one-time boot menu

| Setting | Value | Reason |
|---|---|---|
| Secure Boot | **Disabled** | The installer won't boot otherwise |
| After Power Loss | **Power On** | Auto-restart after a power outage |
| Administrator / Supervisor password | **Set** | Prevents BIOS tampering |
| Power-On password | **Not set** | Would block unattended reboots |
| Boot order *(after install)* | **NVMe only**, USB boot disabled | 🕓 Prevents booting another OS |

<!-- 📸 Image idea: BIOS screen showing "After Power Loss: Power On" -->

---

## 4. Installation

1. Boot from the USB stick (`F12`). Wait for the live system login prompt.
2. Log in as **`installer`** / **`opnsense`**.

> ⚠️ The keyboard is **QWERTY** at this stage. On an AZERTY keyboard, type the `a` of `installer` with the **Q** key.

| Installer screen | Choice |
|---|---|
| Keymap | **French** (AZERTY) |
| Install mode | **Install (ZFS)** |
| ZFS pool type | **stripe** (single disk) |
| Target disk | **NVMe** (`nda0` / `nvd0`, ~250 GB). ⚠️ Not `da0` (USB stick) |
| Root password | Strong password, stored in a password manager |
| End | **Complete install**, then **remove the USB stick** before reboot |

<!-- 📸 Image idea: installer screen "Install (ZFS)" + disk selection -->

---

## 5. Console setup

Log in as `root`. Console menu:

**Option 1 — Assign interfaces**

| Prompt | Answer |
|---|---|
| Configure LAGGs? | No |
| Configure VLANs? | No (done later in the web UI) |
| WAN interface | **`ue0`** (USB adapter — FreeBSD names every USB NIC `ueX`) |
| LAN interface | **`em0`** (onboard Intel) |
| Optional interfaces | None |

**Option 2 — Set interface IP address → LAN**

| Prompt | Answer |
|---|---|
| Configure via DHCP? | No |
| IPv4 address | **`192.168.10.1`** |
| Subnet bit count | **`24`** |
| Upstream gateway | None |
| IPv6 | No (for now) |
| Enable DHCP server on LAN? | **Yes**, range `192.168.10.100` → `192.168.10.200` |
| Revert web GUI to HTTP? | **No** (keep HTTPS) |

> ⚠️ Do **not** keep the default `192.168.1.1/24`: it collides with the Freebox network on the WAN side during the double-NAT phase. `192.168.10.0/24` is temporary until the VLAN plan is applied.

**Expected console header:**

```
LAN (em0)  -> v4: 192.168.10.1/24
WAN (ue0)  -> v4/DHCP4: 192.168.1.x/24
```
<img width="1600" height="733" alt="Console_Header" src="https://github.com/user-attachments/assets/2990b5cc-ddb7-455d-886d-ea4e44cfdb9b" />



---

## 6. WAN fix: AX88179A → `cdce` driver

**Symptom:** `ue0: link state changed to DOWN` / `UP` several times per second, no or intermittent Internet, failed firmware updates.

**Cause:** known bug in the FreeBSD `axge(4)` driver with the AX88179A chip ([FreeBSD bug 267514](https://www.mail-archive.com/usb@freebsd.org/msg01348.html)). Still present on FreeBSD 15.1.

**Fix:** switch the adapter to USB configuration 1 (CDC Ethernet), so it attaches to the generic, stable `cdce(4)` driver. No third-party driver is installed.

### 6.1 Identify the adapter

Console → option **8 (Shell)**:

```sh
usbconfig list
# e.g. ugen0.3: <ASIX ... AX88179A> at usbus0, cfg=0 ...

usbconfig -d ugen0.3 dump_device_desc | grep -E "idVendor|idProduct"
# Expected: idVendor = 0x0b95 / idProduct = 0x1790
```

### 6.2 Manual test (not persistent)

```sh
usbconfig -d ugen0.3 set_config 1
dmesg | tail -n 15        # look for "cdce0" attached to ue0
```

Then from the admin PC: `ping -t 1.1.1.1` for 5 minutes + a speed test.

### 6.3 Make it persistent with devd

**Script** — `/usr/local/sbin/axge-to-cdce.sh`

```sh
#!/bin/sh
# Switch the ASIX AX88179A adapter to the cdce driver
# Workaround for FreeBSD bug 267514 (axge: link flapping)
DEV="$1"
usbconfig -d "$DEV" | grep -q "cfg=1" && exit 0
usbconfig -d "$DEV" set_config 1
logger -t axge-to-cdce "$DEV switched to configuration 1 (cdce driver)"
```

```sh
chmod +x /usr/local/sbin/axge-to-cdce.sh
```

**devd rule** — `/usr/local/etc/devd/axge-cdce.conf`

```
notify 100 {
    match "system"    "USB";
    match "subsystem" "DEVICE";
    match "type"      "ATTACH";
    match "vendor"    "0x0b95";
    match "product"   "0x1790";
    action "/usr/local/sbin/axge-to-cdce.sh $cdev";
};
```

```sh
service devd restart
```

> The `cfg=1` check in the script makes it idempotent and prevents a switch loop.

### 6.4 Verify (after reboot and after **every major update**)

```sh
dmesg | grep -E "cdce|axge"                         # cdce0 must be present
grep axge-to-cdce /var/log/system/latest.log        # script log entry
```

### 6.5 Rollback

```sh
rm /usr/local/etc/devd/axge-cdce.conf /usr/local/sbin/axge-to-cdce.sh
service devd restart
# then reboot
```

**Measured result** (double NAT): 743 Mbps down / 871 Mbps up / 14 ms, zero link drops.

<!-- 📸 Image idea: speed test result + dmesg output showing cdce0 -->

---

## 7. First web login

1. Admin PC on the LAN port, **Wi-Fi disabled** (forces traffic through OPNsense).
2. Browse to **`https://192.168.10.1`**.
3. Before accepting the self-signed certificate warning, compare its **SHA-256 fingerprint** with the one displayed on the console.
4. Log in as `root`.

---

## 8. Snapshots & updates

**Snapshots** — *System → Snapshots* (ZFS boot environments)

| Action | How |
|---|---|
| Create | **+** → name it (e.g. `post-install`, `wan-cdce-ok`) |
| Restore | **✓ Activate** on the snapshot → reboot |
| Read the `Active` column | `N` = running now · `R` = used at next reboot |

> ⚠️ Never click **Activate** unless you want to roll back. Snapshots live on the same disk: they are **not** a backup.

**Updates** — *System → Firmware → Status → Check for updates*

1. Create a snapshot first.
2. Update, wait for the reboot.
3. Check the version on the dashboard.
4. Re-run the checks from [6.4](#64-verify-after-reboot-and-after-every-major-update).

| Snapshot | Taken |
|---|---|
| `post-install` | Right after installation |
| `wan-cdce-ok` | After validating the `cdce` fix |
| Post-update snapshot | After upgrading to 26.7.6 |

---

## 9. Setup wizard

*System → Wizard*

| Screen | Setting | Value |
|---|---|---|
| General | Domain | `internal` (ICANN private-use TLD — avoid `.local`, which conflicts with mDNS) |
| General | Primary / secondary DNS | Empty |
| General | Override DNS | **Unchecked** |
| General | Enable Resolver (Unbound) | **Checked** (recursive resolution) |
| General | DNSSEC / Harden DNSSEC data | **Checked** |
| Time | Timezone | `Europe/Paris` |
| WAN | Type | DHCP |
| WAN | Block RFC1918 private networks | **Unchecked — temporary** (WAN is in 192.168.1.0/24 during double NAT). ⚠️ Re-enable after cut-over |
| WAN | Block bogon networks | **Checked** |
| LAN | IP / mask | `192.168.10.1` / `24` |
| Root password | — | Left empty (unchanged) |

<!-- 📸 Image idea: wizard WAN screen showing both "Block" checkboxes -->

---

## 10. Hardening & backup

🕓 *To be completed: web GUI on LAN only (HTTPS), SSH disabled, console menu password, named admin account with TOTP, encrypted `config.xml` export stored outside the repository.*

## 11. VLANs & DHCP

🕓 *To be completed: VLAN sub-interfaces on `em0` (10, 20, 30, 40, 50, 60), gateways, one DHCP scope per VLAN.*

## 12. Firewall rules

🕓 *To be completed: default deny, inter-VLAN flow matrix.*

## 13. Cut-over (Freebox bridge mode)

🕓 *To be completed. Checklist:*

- [ ] IPv4 fullstack active on the Free line
- [ ] Freebox Connect app working over 4G (rollback path)
- [ ] Freebox → bridge mode, WAN cable on the Freebox port carrying the bridge
- [ ] OPNsense WAN gets the public IPv4 via DHCP
- [ ] Re-enable **Block RFC1918 private networks** on WAN
- [ ] IPv4 only at first (IPv6 prefix delegation later)

## 14. Validation tests

🕓 *To be completed: connectivity, inter-VLAN isolation, throughput after cut-over, simulated power outage (Freebox and Tiny rebooting together).*

---

## Appendix A — Diagnostic commands

| Where | Command | Use |
|---|---|---|
| Windows | `ipconfig` | Check the address received (192.168.10.1xx = OK, 169.254.x.x = no DHCP) |
| Windows | `ping -t 1.1.1.1` | Continuous connectivity test, no DNS involved |
| Windows | `nslookup wikipedia.org` | DNS resolution through OPNsense |
| OPNsense shell | `dmesg \| grep -E "cdce\|axge"` | Which driver the USB NIC is using |
| OPNsense shell | `usbconfig list` | USB devices and their current configuration (`cfg=`) |
| OPNsense GUI | *Interfaces → Overview* | Link status and addresses of each interface |
| OPNsense GUI | *System → Log Files → General* (filter `ue0`) | Link state changes |
| OPNsense GUI | *System → Gateways → Configuration* | Gateway status, loss, RTT |
| OPNsense GUI | *Interfaces → Diagnostics → Ping* | Ping from the firewall itself |

## Appendix B — Troubleshooting

| Symptom | Check | Fix |
|---|---|---|
| PC gets `169.254.x.x` | Cable in the right port? Tiny booted? | PC cable → onboard Intel port (`em0`) |
| PC gets `192.168.1.x` | Cables swapped | Freebox → USB adapter, PC → Intel port |
| WAN has no IPv4, link up | Logs: `ue0: link state changed` repeating? | Apply [section 6](#6-wan-fix-ax88179a--cdce-driver) |
| Intermittent / very slow Internet | `ping -t 1.1.1.1` drops + link state changes in logs | Apply [section 6](#6-wan-fix-ax88179a--cdce-driver) |
| `ping 1.1.1.1` OK but no website loads | `nslookup` fails | Check *Services → Unbound DNS* is enabled |
| Firmware update shows nothing / stays on old version | WAN stable? | Fix the WAN link first, then retry |
| Lost access after a bad change | — | *System → Snapshots* → Activate last good snapshot → reboot |

## Appendix C — References

- [OPNsense documentation](https://docs.opnsense.org/)
- [FreeBSD `axge(4)` manual](https://man.freebsd.org/cgi/man.cgi?query=axge&sektion=4)
- [FreeBSD `cdce(4)` manual](https://man.freebsd.org/cgi/man.cgi?query=cdce&sektion=4)
- [FreeBSD `devd.conf(5)` manual](https://man.freebsd.org/cgi/man.cgi?query=devd.conf&sektion=5)
- [FreeBSD bug 267514 — AX88179A link flapping](https://www.mail-archive.com/usb@freebsd.org/msg01348.html)
- [OPNsense forum — USB Ethernet ASIX AX88179A](https://forum.opnsense.org/index.php?topic=33098.0)
