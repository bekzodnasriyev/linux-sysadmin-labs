# Linux Device Model and Kernel Interfaces

## 1. Objective

Investigate how Linux represents hardware and virtual devices, exposes kernel information to user space, and connects device nodes with kernel device objects and drivers.

This lab examines three important kernel interfaces:

- `/dev` — device nodes used by user-space applications.
- `/proc` — process and kernel information interface.
- `/sys` — sysfs representation of kernel objects, devices, drivers, and subsystems.

The investigation uses read-only inspection on a running Arch Linux system.

## 2. Environment

| Component            | Observed value                |
|----------------------|-------------------------------|
| Operating System     | Arch Linux                    |
| Kernel               | 7.2.14-arch1-1                |
| Architecture         | x86_64                        |
| CPU                  | Intel Core i5-13420H          |
| Logical CPUs         | 12                            |
| Main storage         | NVMe, approximately 476.9 GiB |
| Root filesystem      | ext4                          |
| Boot filesystem      | vfat                          |
| Virtual block device | zram0, 4 GiB                  |

## 3. Investigation Methodology

The investigation followed this workflow:

1. Identify the three kernel interfaces.
2. Inspect their mounted filesystem types.
3. Identify block devices and device nodes.
4. Trace device nodes to their sysfs representations.
5. Inspect major and minor device numbers.
6. Examine kernel-exported device attributes.
7. Inspect block-device information in `/proc`.
8. Identify the PCI controller and its bound kernel driver.

All observations were collected using read-only commands. No device configuration, partition table, filesystem, or driver binding was modified.

## 4. Kernel Interfaces

### 4.1 `/dev` — Device Nodes

The `/dev` directory is backed by `devtmpfs` on the investigated system.

It provides device nodes through which user-space applications can access kernel-managed devices.

Observed examples:

| Device node      | Type  | Major | Minor |
|------------------|-------|-------|-------|
| `/dev/nvme0n1`   | Block | 259   | 0     |
| `/dev/nvme0n1p1` | Block | 259   | 1     |
| `/dev/zram0`     | Block | 253   | 0     |

The leading `b` in the long-format `ls -l` output identifies a block device.

Major and minor numbers are used by the kernel to identify the device associated with a device node.

A device node is not the physical device itself. It is an interface through which user space can request operations handled by the kernel.

### 4.2 `/proc` — Process and Kernel Information

The `/proc` directory is backed by the `proc` filesystem.

It exposes information about running processes and kernel state.

The investigation examined:

- `/proc/partitions`
- `/proc/devices`

Observed block-device entries in `/proc/partitions`:

| Major | Minor | Device    | Reported blocks |
|-------|-------|-----------|-----------------|
| 259   | 0     | nvme0n1   | 500107608       |
| 259   | 1     | nvme0n1p1 | 1048576         |
| 259   | 2     | nvme0n1p2 | 479134720       |
| 259   | 3     | nvme0n1p5 | 1048576         |
| 259   | 4     | nvme0n1p6 | 18874368        |
| 253   | 0     | zram0     | 4194304         |

The `#blocks` values in `/proc/partitions` are expressed in 1 KiB blocks.

The `/proc/devices` output included:

- Major 253: `zram`
- Major 259: `blkext`

These entries describe registered block-device major-number associations. They are not a list of individual physical devices.

### 4.3 `/sys` — Sysfs and the Linux Device Model

The `/sys` directory is backed by `sysfs`.

Unlike `/dev`, which exposes device nodes, sysfs exposes structured information about kernel objects and their relationships.

The investigation found these block-device paths:

**NVMe disk:**

`/sys/devices/pci0000:00/.../nvme/nvme0/nvme0n1`

**NVMe partition:**

`/sys/devices/pci0000:00/.../nvme/nvme0/nvme0n1/nvme0n1p1`

**ZRAM:**

`/sys/devices/virtual/block/zram0`

The paths demonstrate two different device representations:

- The NVMe disk belongs to a hardware-related device hierarchy.
- ZRAM is represented as a virtual block device.

The `/sys/class/block` directory provides a class-oriented view of block devices, with links to their underlying kernel objects.

## 5. Device Attributes

The following attributes were read from `/sys/class/block`.

| Attribute                  | Observed value | Meaning                             |
|----------------------------|----------------|-------------------------------------|
| `size`                     | 1000215216     | Number of 512-byte sectors          |
| `queue/logical_block_size` | 512            | Logical block size in bytes         |
| `ro`                       | 0              | Device is not marked read-only      |
| `removable`                | 0              | Device is not reported as removable |
| `device/model`             | 321JN0512GB    | Device model                        |
| Partition `partition`      | 1              | First partition                     |

### Size calculation

The `size` attribute reports the number of 512-byte sectors.

Calculation:

`1000215216 × 512 = 512110190592 bytes`

This is approximately:

- 512.11 GB (decimal)
- 476.94 GiB (binary)

This agrees with the capacity reported by `lsblk`.

**Important:** `ro=0` does not guarantee that every user can write to the device. Actual access also depends on permissions, device access controls, filesystem mount options, and other kernel mechanisms.

## 6. Device Identity and udev

The `udevadm info` investigation returned properties including:

- `DEVPATH`
- `DEVNAME`
- `DEVTYPE`
- `MAJOR`
- `MINOR`
- `SUBSYSTEM`

For the NVMe disk, the observed values included:

| Property    | Value          |
|-------------|----------------|
| `DEVNAME`   | `/dev/nvme0n1` |
| `DEVTYPE`   | `disk`         |
| `MAJOR`     | 259            |
| `MINOR`     | 0              |
| `SUBSYSTEM` | `block`        |

The sysfs `uevent` attribute exposed the following device information:

```text
MAJOR=259
MINOR=0
DEVNAME=nvme0n1
DEVTYPE=disk
DISKSEQ=1
```

udev operates in user space. It receives kernel device events and processes device information, applying rules and managing device-related properties and links.

The kernel device model and udev are related but distinct:

- The kernel manages device objects, drivers, and device events.
- udev processes those events in user space.
- devtmpfs provides device nodes; udev can apply additional permissions, ownership, naming, and symlink rules.

## 7. PCI Controller and Kernel Driver

The investigation identified the NVMe controller's PCI address as `0000:01:00.0`.

The PCI device's driver link resolved to:

```text
/sys/bus/pci/drivers/nvme
```

This confirms that the PCI device is bound to the Linux `nvme` PCI driver.

The earlier attempt to find a driver under:

`/sys/class/block/nvme0n1/device/driver`

returned no such file.

This was not evidence of a missing NVMe driver. The block device's `device` link pointed to the NVMe controller object, while the driver binding was found at the PCI controller level.

This distinction is important when troubleshooting Linux device hierarchies: a driver link is associated with a particular kernel device object, not necessarily every child or related object.

## 8. Observed Device Relationship

The investigation established the following relationship:

```text
PCI DEVICE
0000:01:00.0
      |
      v
PCI DRIVER
/sys/bus/pci/drivers/nvme
      |
      v
NVMe CONTROLLER
nvme0
      |
      v
BLOCK DEVICE
nvme0n1
      |
      +------ nvme0n1p1
      +------ nvme0n1p2
      +------ nvme0n1p5
      +------ nvme0n1p6
      |
      v
DEVICE NODE
/dev/nvme0n1
```

This is a simplified conceptual view. It does not represent every intermediate kernel object or every event in device discovery.

## 9. Diagnostic Commands

The following commands were used during the investigation.

|                       Command                          |              Diagnostic purpose                 |
|--------------------------------------------------------|-------------------------------------------------|
| `uname -a`                                             | Identify kernel and architecture                |
| `lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS`           | Inspect block devices and partitions            |
| `findmnt /dev /proc /sys`                              | Identify mounted filesystem types               |
| `ls -l /dev/nvme0n1`                                   | Inspect device node and major/minor numbers     |
| `readlink -f /sys/class/block/nvme0n1`                 | Resolve the kernel device path                  |
| `cat /sys/class/block/nvme0n1/size`                    | Read device capacity in sectors                 |
| `cat /proc/partitions`                                 | Inspect registered partitions and block devices |
| `cat /proc/devices`                                    | Inspect registered device major numbers         |
| `udevadm info --query=property --name=/dev/nvme0n1`    | Inspect udev device properties                  |
| `readlink -f /sys/bus/pci/devices/0000:01:00.0/driver` | Identify the PCI device's bound driver          |

These commands provide complementary views of the same system. No single interface provides the complete device model.

## 10. Production and Security Considerations

### Device identity exposure

udev properties can contain hardware identifiers, serial numbers, WWNs, and partition-table UUIDs.

These values should be reviewed before publishing diagnostic output.

### Permissions

Device nodes have ownership and permission settings. Access to a device can expose sensitive data or allow destructive operations.

Do not assume that a device is safe to access simply because it appears under `/dev`.

### Kernel interfaces

`/proc` and `/sys` expose information about the running system. Some attributes are writable and can change kernel or device behavior.

This lab used read-only inspection. Do not write to sysfs attributes without understanding their exact effect.

### Driver troubleshooting

A missing `driver` link at one sysfs path does not automatically mean that the device has no driver. First identify the exact kernel object and subsystem to which the driver should be bound.

## 11. Limitations

This investigation focused on block devices and the NVMe PCI controller.

It did not attempt to:

- Modify device attributes.
- Unbind or reload kernel drivers.
- Trigger device removal or rediscovery.
- Investigate all udev rules.
- Perform destructive storage tests.
- Examine every kernel subsystem.

The findings describe the observed Arch Linux system and should not be treated as universal output for every Linux distribution or hardware configuration.

## 12. Key Takeaways

- `/dev`, `/proc`, and `/sys` expose different views of kernel-managed resources.
- A device node is not the same thing as the kernel device object.
- Major/minor numbers associate device nodes with kernel device handling.
- sysfs exposes the Linux device hierarchy and device attributes.
- `/proc` provides runtime information about processes and kernel subsystems.
- udev processes kernel device events in user space.
- Driver binding must be investigated at the correct device and subsystem level.
- Combining these interfaces provides a more complete diagnostic picture than relying on any single command.

## 13. Final Result

Successfully traced an NVMe block device from its PCI controller and bound kernel driver through the Linux device hierarchy to its user-space device node.

The investigation also identified the distinction between physical and virtual block devices, inspected kernel-exported attributes, and correlated device information across `/dev`, `/proc`, `/sys`, and udev.

**Status:** Investigation completed. Documentation based on observed system evidence.
