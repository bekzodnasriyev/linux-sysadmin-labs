# Linux Boot Process and Startup Dependency Investigation

## 1. Objective

Investigate the Linux boot process on a running Arch Linux system, from UEFI firmware initialization to the systemd graphical target.

The investigation focuses on:

- UEFI firmware and boot manager.
- Unified Kernel Image (UKI).
- Kernel and initramfs structure.
- NVMe driver and root filesystem initialization.
- Initramfs hooks and root filesystem mounting.
- The `switch_root` transition.
- systemd startup timing and service dependencies.
- Network readiness and its effect on dependent services.

The goal is to understand the actual boot chain and identify startup dependencies using system evidence.

## 2. Environment

| Component | Observed configuration |
|---|---|
| Distribution | Arch Linux |
| Architecture | x86_64 |
| Kernel | 7.2.14-arch1-1 |
| Firmware | UEFI |
| Boot manager | rEFInd |
| Boot image | Unified Kernel Image |
| Initramfs generator | mkinitcpio 42.2 |
| Root device | NVMe partition |
| Root filesystem | ext4 |
| Init system | systemd |
| Default target | graphical.target |

Sensitive identifiers such as partition UUIDs, disk serial numbers, MAC addresses and network names are intentionally excluded.

## 3. Investigation Methodology

The investigation used read-only inspection of:

- Firmware boot entries.
- Boot partition contents.
- UKI structure and embedded sections.
- Kernel command line.
- Initramfs configuration and contents.
- Kernel configuration.
- Loaded kernel modules.
- sysfs device relationships.
- Initramfs initialization scripts.
- systemd service properties and dependencies.
- Journal logs and boot timing measurements.

No bootloader configuration, kernel parameters, service configuration or filesystem contents were modified during the investigation.

## 4. Firmware and Boot Manager

UEFI firmware was confirmed through:

`/sys/firmware/efi`

Firmware boot entries identified rEFInd as the active boot manager.

The boot configuration uses a Unified Kernel Image located at:

`/boot/EFI/Linux/arch-linux.efi`

The UKI was identified as a PE32+ EFI executable.

Its sections include:

- `.linux` — Linux kernel image.
- `.initrd` — initial RAM filesystem.
- `.cmdline` — kernel command line.
- `.osrel` — operating system release information.
- `.uname` — kernel version metadata.
- `.splash` — splash image section.

This confirms that the kernel and initramfs are packaged together in a UKI.

## 5. Kernel Command Line and Root Filesystem

The running kernel command line contains:

- `root=PARTUUID=...` — identifies the root partition.
- `zswap.enabled=0` — disables zswap.
- `rw` — requests read/write root mounting.
- `rootfstype=ext4` — specifies the root filesystem type.

Sensitive partition identifiers are omitted.

The active root filesystem was verified independently:

`findmnt -no SOURCE,FSTYPE /`

Observed result:

`/dev/nvme0n1p2 ext4`

The root filesystem is therefore an ext4 filesystem on an NVMe partition.

## 6. Initramfs Configuration

The mkinitcpio configuration uses:

`HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)`

The hooks provide early boot functionality:

| Hook | Purpose |
|---|---|
| base | Basic initramfs environment |
| udev | Device discovery and management |
| autodetect | Select modules relevant to the current hardware |
| microcode | Early CPU microcode loading |
| modconf | Include module configuration |
| kms | Early kernel mode setting |
| keyboard | Keyboard support |
| keymap | Console keyboard layout |
| consolefont | Console font configuration |
| block | Block device support |
| filesystems | Filesystem support |
| fsck | Filesystem checking tools |

An important distinction: empty `MODULES`, `BINARIES` and `FILES` arrays do not mean the initramfs is empty. Hooks and autodetection populate the image.

## 7. NVMe and ext4 Kernel Support

The kernel configuration showed:

- `CONFIG_BLK_DEV_NVME=m`
- `CONFIG_EXT4_FS=y`

This means:

- NVMe support is compiled as a loadable kernel module.
- ext4 support is built directly into the kernel.

The initramfs inspection confirmed the presence of NVMe modules, including `nvme.ko.zst` and `nvme-core.ko.zst`.

The running system also showed the NVMe modules loaded through `lsmod`.

The PCI controller driver relationship was verified through sysfs:

`/sys/bus/pci/devices/.../driver`

The resolved driver path was:

`/sys/bus/pci/drivers/nvme`

The NVMe namespace exists beneath the corresponding PCI controller in sysfs.

An attempted lookup of `device/driver` directly under the namespace did not exist. This was not a driver failure: the driver binding is associated with the PCI controller.

## 8. Initramfs Initialization and Root Transition

The actual initramfs entry script was inspected:

`/usr/lib/initcpio/init`

Its initialization sequence includes:

1. Set up the early userspace environment through `mount_setup`.
2. Source initramfs helper functions.
3. Prepare virtual filesystems and required device interfaces.
4. Parse the kernel command line.
5. Initialize early logging.
6. Load initramfs configuration.
7. Execute early hooks.
8. Load required kernel modules.
9. Execute regular hooks.
10. Resolve the root device.
11. Run the configured root filesystem check.
12. Mount the real root filesystem at `/sysroot`.
13. Execute late and cleanup hooks.
14. Verify the root mount and the presence of the init executable.
15. Execute `switch_root` to transfer control to the real root filesystem.

The final transition is performed using:

`/usr/bin/switch_root /sysroot "$init" "$@"`

The default init executable is `/sbin/init`, which resolves to systemd on this system.

Simplified transition:

```text
Linux kernel
    |
    v
/init (initramfs, PID 1)
    |
    +--> Prepare /dev, /proc, /sys, /run
    |
    +--> Discover and resolve root device
    |
    +--> Check and mount root at /sysroot
    |
    v
switch_root
    |
    v
/sysroot/sbin/init
    |
    v
systemd (PID 1)
    |
    v
System services
    |
    v
graphical.target
```

The initramfs is an independent early userspace environment. `switch_root` changes the root filesystem and starts the real init process. It is not simply a normal directory change.

## 9. Boot Timing Analysis

The command `systemd-analyze time` reported:

| Boot phase | Duration |
|---|---:|
| Firmware | 4.292 s |
| Boot loader | 4.051 s |
| Kernel | 2.097 s |
| Userspace | 8.082 s |
| Total | 18.522 s |

The graphical target was reached approximately 8.081 seconds into userspace startup.

These values describe the measured boot and are not universal performance guarantees.

## 10. Service Startup Analysis

The `systemd-analyze blame` output identified the following service startup durations:

| Service | Duration |
|---|---:|
| NetworkManager-wait-online.service | 4.423 s |
| systemd-binfmt.service | 950 ms |
| thermald.service | 720 ms |
| vmware.service | 663 ms |
| docker.service | 633 ms |
| NetworkManager.service | 491 ms |
| upower.service | 428 ms |
| proton.VPN.service | 376 ms |
| containerd.service | 172 ms |

These values represent service activation durations. They do not independently establish how much each service contributes to total boot time.

The critical-chain analysis provides dependency and ordering information.

## 11. NetworkManager Wait-Online Investigation

`NetworkManager-wait-online.service` was identified as an important part of the boot dependency chain.

Its unit executes:

`/usr/bin/nm-online -s -q`

The unit is ordered after NetworkManager and before `network-online.target`.

Its configured timeout is 60 seconds. The observed execution completed successfully in approximately 4.423 seconds.

Journal evidence showed:

- NetworkManager started.
- The Wi-Fi device became available.
- Automatic connection activation began later.
- Authentication and association completed.
- DHCPv4 configuration succeeded.
- NetworkManager reached startup complete.
- The wait-online service completed successfully.

The wireless driver firmware loaded earlier in the boot process. The available evidence does not establish that firmware loading caused the wait-online delay.

The exact reason for the entire 4.423-second wait was not conclusively identified.

## 12. network-online.target Dependencies

The critical chain showed:

```text
graphical.target
  |
  v
multi-user.target
  |
  v
docker.service
  |
  v
network-online.target
  |
  v
NetworkManager-wait-online.service
  |
  v
NetworkManager.service
```

Docker declares:

- `Wants=network-online.target`
- `After=network-online.target`

Ollama also declares:

- `Wants=network-online.target`
- `After=network-online.target`

Both are ordered after the target and therefore wait for its completion before starting.

These are weak dependencies, not `Requires=network-online.target`.

The target is a synchronization point. It does not guarantee that every external website or internet service is reachable.

The critical-chain evidence indicates that wait-online participates in the startup ordering of Docker and Ollama.

## 13. Separate Wi-Fi Firmware Observation

Kernel journal inspection also revealed a separate Intel Wi-Fi firmware error approximately 19 minutes after boot.

The journal recorded a firmware error, device reset and subsequent restart.

This occurred well after the initial boot sequence and was not established as the cause of the measured startup delay.

It is documented as a separate observation rather than a boot-time root cause.

## 14. Diagnostic Lessons

The investigation demonstrates several important operational principles:

- A bootloader entry does not by itself prove that the entire boot chain is healthy.
- A UKI can package the kernel, initramfs and command line into one EFI executable.
- A kernel driver may be built in or loaded as a module; these require different diagnostic interpretations.
- Device-driver binding must be inspected at the correct level of the kernel device hierarchy.
- Initramfs is an independent early userspace environment.
- Discovering a block device does not guarantee that its filesystem can be mounted.
- Successfully mounting root does not guarantee that the expected init executable exists.
- `switch_root` is the transition from initramfs to the real root filesystem.
- `systemd-analyze blame` and `critical-chain` answer different questions.
- Service ordering and service requirements are not interchangeable.
- `network-online.target` does not guarantee universal internet connectivity.
- A warning or error occurring after boot must not automatically be blamed for startup delays.

## 15. Scope and Limitations

This was a read-only investigation of an operating system that booted successfully.

The work did not include:

- Simulating a failed boot.
- Modifying bootloader entries or kernel parameters.
- Deliberately breaking initramfs.
- Testing filesystem corruption or recovery.
- Changing service dependencies to optimize boot time.
- Proving the exact internal reason for the wait-online duration.
- Establishing the cause or user impact of the later Wi-Fi firmware error.

No production incident or root cause is claimed.

## 16. Final Result

The investigation reconstructed the observed boot path from UEFI firmware through rEFInd, UKI, Linux kernel, initramfs, root filesystem mounting and `switch_root` to systemd.

It also identified a measurable startup dependency involving NetworkManager wait-online and services ordered after `network-online.target`.
