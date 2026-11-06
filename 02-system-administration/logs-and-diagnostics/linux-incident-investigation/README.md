# Linux Logs and Incident Investigation

## 1. Objective

Investigate Linux system logs using systemd-journald and journalctl.

The investigation focuses on:

- Persistent journal storage and retention.
- Boot history and event timelines.
- Log volume and message sources.
- Kernel warnings and errors.
- Driver configuration problems.
- Distinguishing actual faults from harmless or unexplained warnings.
- Applying evidence-based corrections and verifying results after reboot.

This investigation was performed on an Arch Linux host.

## 2. Environment

- Operating system: Arch Linux
- Architecture: x86_64
- Init system: systemd
- Logging service: systemd-journald
- Network management: NetworkManager
- Hardware investigated: Intel graphics and Wi-Fi drivers

Sensitive hardware identifiers, MAC addresses, UUIDs, serial numbers and local network addresses are excluded.

## 3. Investigation Methodology

The investigation followed this workflow:

1. Establish the logging baseline.
2. Inspect journal storage and boot history.
3. Identify high-volume log sources.
4. Examine warnings and errors.
5. Correlate events by timestamp.
6. Form hypotheses based on evidence.
7. Apply minimal configuration corrections.
8. Reboot and verify results.
9. Document unresolved warnings and limitations.

A warning or error message was not automatically treated as a confirmed incident.

## 4. systemd-journald Baseline

Service state was checked:

```bash
systemctl is-active systemd-journald
systemctl is-enabled systemd-journald
```

Observed results:

- Service state: active.
- Enablement state: static.

A static unit is not necessarily a problem. It is not intended to be enabled directly in the usual way.

Journal disk usage:

```bash
journalctl --disk-usage
```

Observed journal usage: approximately 1.5 GiB.

The persistent journal directory occupied approximately 1.6 GiB:

```bash
sudo du -sh /var/log/journal /run/log/journal 2>/dev/null
```

The small difference between reported values is attributable to different accounting and display methods.

## 5. Journal Persistence and Configuration

The configuration was inspected:

```bash
systemd-analyze cat-config systemd/journald.conf
```

No explicit storage or size-limit overrides were found in the displayed configuration.

The `Storage` setting was commented out. The default `auto` behavior uses persistent storage when `/var/log/journal` exists.

Persistent journal files were present under `/var/log/journal`.

Important distinctions:

- A commented-out `SystemMaxUse` does not mean unlimited journal storage.
- systemd-journald has built-in defaults.
- `cat-config` displays configuration text; it does not print every resolved runtime default.

Boot history was inspected:

```bash
journalctl --list-boots
```

The journal contained 94 boot entries, including the current boot.

## 6. Journal Integrity and File Segments

Journal files were inspected:

```bash
sudo find /var/log/journal -type f -printf '%s %f\n' | sort -n | tail -n 20
```

The largest displayed journal segments were approximately 48–56 MiB each.

This indicated that journal usage was distributed across multiple files rather than being caused by one exceptionally large segment.

Journal integrity was checked:

```bash
sudo journalctl --verify
```

All journal files visible in the captured output reported `PASS`.

The displayed output was truncated, so this report does not claim that every file was independently verified.

## 7. Log Source Analysis

Journal records were grouped by systemd unit or syslog identifier.

All available history:

```bash
sudo journalctl --no-pager -o json |
jq -r '._SYSTEMD_UNIT // .SYSLOG_IDENTIFIER // "unknown"' |
sort | uniq -c | sort -nr | head -n 20
```

The largest historical sources included:

| Source | Records |
|---|---:|
| user@1000.service | 1,367,005 |
| vmware.service | 360,747 |
| kernel | 242,878 |
| init.scope | 84,900 |
| NetworkManager.service | 60,177 |
| session-1.scope | 44,179 |
| containerd.service | 38,448 |
| docker.service | 27,768 |

A separate seven-day query was performed:

```bash
sudo journalctl --since "7 days ago" --no-pager -o json |
jq -r '._SYSTEMD_UNIT // .SYSLOG_IDENTIFIER // "unknown"' |
sort | uniq -c | sort -nr | head -n 20
```

The largest recent sources included:

| Source | Records |
|---|---:|
| kernel | 11,176 |
| user@1000.service | 5,675 |
| NetworkManager.service | 4,813 |
| init.scope | 4,609 |
| vmware.service | 1,997 |
| archlinux-keyring-wkd-sync.service | 1,568 |
| session-1.scope | 1,406 |
| containerd.service | 981 |
| wpa_supplicant.service | 890 |

### Assessment

Historical VMware-related logging was substantially higher than its recent seven-day volume.

The counts alone did not establish a logging loop, service failure, or resource leak.

The kernel was the largest source in the recent seven-day sample. Further classification was necessary before treating this as abnormal.

## 8. Warning and Error Classification

Warnings and errors were inspected using:

```bash
sudo journalctl -b -p warning --no-pager -n 30
sudo journalctl -k -b -p err --no-pager
```

The investigation identified several categories:

| Event | Assessment |
|---|---|
| UFW BLOCK messages | Firewall events; not automatically evidence of an attack |
| Desktop portal ScreenSaver warning | Feature/API limitation; no confirmed system failure |
| TLP power policy warning | Configuration conflict with power-profiles-daemon |
| portfolio-failure-demo.service failure | Previously controlled test from an earlier lab |
| i915 driver warning | Hardware-specific warning; graphics remained operational |
| SMBus busy messages | Startup messages; root cause not established |
| ACPI missing symbol | Firmware/ACPI method errors; functional impact not established |
| Missing regulatory.db | Confirmed missing package/file; corrected |
| Unsupported iwlwifi parameter | Confirmed unsupported configuration; corrected |

A journal priority is a classification of message severity, not proof of root cause.

## 9. Intel Graphics Driver Investigation

The kernel journal was filtered for Intel graphics messages:

```bash
sudo journalctl -k -b --no-pager |
grep -E 'i915|intel_bios_init|drm'
```

The investigation found:

```text
i915: unknown parameter 'fastboot' ignored
```

The configuration file `/etc/modprobe.d/i915.conf` contained an unsupported `fastboot=1` parameter.

Supported parameters were checked:

```bash
modinfo -p i915
```

The unsupported option was removed while preserving the other configured options.

Resulting configuration:

```text
options i915 enable_fbc=1 enable_guc=3 enable_psr=1
```

The initramfs and Unified Kernel Image were regenerated:

```bash
sudo mkinitcpio -P
```

After reboot, the `fastboot` warning no longer appeared.

### Remaining i915 warning

An `intel_bios_init` warning remained.

However:

- The i915 driver initialized.
- The graphics device was bound to the i915 driver.
- `graphical.target` was active.
- No visible graphics malfunction was reported.

The remaining warning was documented as unresolved, with no confirmed functional impact.

No additional driver parameters were changed without evidence.

## 10. SMBus and ACPI Investigation

The SMBus messages were counted and examined:

```bash
sudo journalctl -k -b --no-pager |
grep -Fc 'SMBus is busy'

sudo journalctl -k -b --no-pager |
grep -F 'SMBus is busy'
```

Seven messages appeared within approximately 5.6 milliseconds during startup.

The controller was inspected:

```bash
lspci -nnk -s 00:1f.4
```

Observed:

- Intel SMBus Host Controller.
- Kernel driver in use: `i801_smbus`.
- Kernel module: `i2c_i801`.

The driver was bound to the controller.

The journal also contained ACPI errors involving an unresolved `WIST` symbol and aborted methods:

```text
ACPI BIOS Error: Could not resolve symbol
ACPI Error: Aborting method ... CNVW.IFUN
ACPI Error: Aborting method ... CNVW._DSM
```

### Timeline assessment

The SMBus messages and ACPI errors occurred close together during boot, but temporal proximity does not establish a causal relationship.

The Wi-Fi driver subsequently detected the hardware, loaded firmware, and connected successfully.

The exact root causes of the SMBus and ACPI messages were not established.

No BIOS changes or speculative driver modifications were made.

## 11. Wi-Fi Driver Configuration Investigation

The initial journal contained:

```text
iwlwifi: unknown parameter 'd0i3_disable' ignored
```

The module's supported parameters were inspected:

```bash
modinfo -p iwlwifi
```

The configuration file `/etc/modprobe.d/iwlwifi.conf` contained:

```text
options iwlwifi power_save=1 d0i3_disable=0 uapsd_disable=0
```

The installed driver exposed `power_save` and `uapsd_disable`, but not `d0i3_disable`.

The unsupported option was removed.

Final configuration:

```text
options iwlwifi power_save=1 uapsd_disable=0
```

The effective configuration was verified:

```bash
modprobe -c | grep '^options iwlwifi'
```

After reboot:

- The unsupported `d0i3_disable` warning disappeared.
- The Intel Wi-Fi device was detected.
- Firmware loaded successfully.
- NetworkManager reported the Wi-Fi interface as connected.

The correction was verified without changing the remaining supported parameters.

## 12. Regulatory Database Investigation

The kernel journal initially reported:

```text
Direct firmware load for regulatory.db failed with error -2
cfg80211: failed to load regulatory.db
```

The package and file were checked:

```bash
pacman -Q wireless-regdb
ls -l /usr/lib/firmware/regulatory.db
```

The package was initially absent, and the requested file did not exist.

The official Arch Linux package `wireless-regdb` was available and installed:

```bash
sudo pacman -S wireless-regdb
```

After reboot, the journal was checked again:

```bash
sudo journalctl -k -b --no-pager |
grep -Ei 'regulatory\.db|failed to load regulatory'
```

The query returned no matching messages.

This confirmed that the missing regulatory database warning was resolved in the subsequent boot.

## 13. Final Verification

The following results were confirmed:

| Item | Final result |
|---|---|
| systemd-journald | Active |
| Persistent journal | Present |
| Unsupported i915 `fastboot` option | Removed; warning disappeared |
| Unsupported iwlwifi `d0i3_disable` option | Removed; warning disappeared |
| wireless-regdb | Installed |
| regulatory.db warning | No longer present after reboot |
| Wi-Fi firmware | Loaded successfully |
| Wi-Fi connection | Connected |
| i915 driver | Initialized and bound |
| SMBus messages | Root cause unknown |
| ACPI messages | Root cause unknown |
| Remaining i915 warning | No confirmed functional impact |
| iwlwifi `Unhandled alg: 0x707` | Unresolved; impact not established |

## 14. Operational Lessons

1. A warning is an observation, not a root cause.
2. Journal volume must be analyzed by source and time range.
3. Historical record counts do not automatically indicate a current problem.
4. Kernel module parameters must be checked against the installed module.
5. Configuration changes require post-reboot verification when they affect early driver initialization.
6. Multiple messages occurring together do not necessarily share a root cause.
7. A working device does not make every warning irrelevant, but it provides important evidence about functional impact.
8. Unresolved warnings should remain documented as unresolved rather than being presented as repaired incidents.
9. Avoid speculative BIOS, kernel, and driver changes when the evidence does not justify them.

## 15. Limitations

- This was a local Arch Linux host investigation, not a production outage response.
- No confirmed production incident was reproduced.
- The exact causes of the SMBus, ACPI, and remaining i915 warnings were not established.
- The impact of `Unhandled alg: 0x707` was not established.
- Journal integrity output was truncated; only the visible `PASS` results are reported.
- No BIOS firmware update, kernel downgrade, or hardware replacement was performed.

## 16. Final Result

The investigation identified and corrected three configuration or package problems:

1. An unsupported Intel graphics module parameter.
2. An unsupported Intel Wi-Fi module parameter.
3. A missing wireless regulatory database package.

Each correction was followed by a reboot and a targeted journal verification.

Other hardware-specific warnings were investigated and documented without claiming an unsupported root cause.

The result is an evidence-based Linux logging and incident investigation, including configuration changes, verification, and explicit limitations.
