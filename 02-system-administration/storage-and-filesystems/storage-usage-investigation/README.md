# Linux Storage Usage Investigation and Docker Cleanup

## 1. Objective

Investigate elevated root filesystem usage, identify the largest storage consumers, determine whether Docker storage is contributing to disk consumption, perform a controlled cleanup, and verify the result.

The investigation follows a read-only diagnostic workflow before any cleanup action.

## 2. Environment

- OS: Arch Linux
- Root filesystem: ext4
- Root device: NVMe partition
- Container runtime: containerd
- Container management: Docker
- Diagnostic approach: read-only inspection followed by targeted cleanup

Machine-specific identifiers have been excluded.

## 3. Initial Observation

Initial filesystem usage:

| Metric               | Initial value |
|----------------------|---------------|
| Root filesystem size | 449 GB        |
| Used space           | 324 GB        |
| Available space      | 102 GB        |
| Usage                | 77%           |
| Inode usage          | 7%            |

The root filesystem had substantial usage, but inode consumption was low.

This indicated that the immediate issue was storage capacity consumption rather than inode exhaustion.

## 4. Diagnostic Workflow

### 4.1 Filesystem capacity

Commands:

```bash
df -hT
df -i
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS
findmnt -no SOURCE,FSTYPE,OPTIONS /
```

### 4.2 Locate major storage consumers

Commands:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
sudo du -xhd1 /var 2>/dev/null | sort -h
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
sudo du -xhd1 /home 2>/dev/null | sort -h
```

### 4.3 Investigate containerd storage

Commands:

```bash
sudo du -xhd2 /var/lib/containerd 2>/dev/null | sort -h
systemctl status containerd.service docker.service --no-pager
sudo ctr namespaces list
```

Findings:

- containerd initially occupied approximately 42 GB.
- Content blobs occupied approximately 9.8 GB.
- OverlayFS snapshots occupied approximately 32 GB.
- The `moby` namespace was present, which is commonly used by Docker.
- Both Docker and containerd services were running.

### 4.4 Investigate Docker objects

Command:

```bash
docker system df -v
```

Findings:

- Numerous untagged images were present.
- Many images had no container references.
- Several stopped containers remained.
- Two persistent volumes were present.
- Build cache usage was negligible.

Image sizes were not treated as independent values because Docker images can share layers.

### 4.5 Investigate Pacman cache

Commands:

```bash
sudo du -sh /var/cache/pacman/pkg
sudo ls -lhS /var/cache/pacman/pkg | head -15
pacman -Q linux
pacman -Q nvidia-utils
pacman -Q blender
```

The package cache occupied approximately 20 GB and contained multiple versions of installed packages.

A dry run was performed before cleanup:

```bash
paccache -d
```

The dry run identified 437 candidates and estimated approximately 8.19 GiB of reclaimable space. No files were removed by this command.

## 5. Root Cause Assessment

The investigation identified two major cleanup opportunities:

1. Docker image and container storage had accumulated over time. The containerd storage directory initially occupied approximately 42 GB.
2. The Pacman package cache occupied approximately 20 GB and contained multiple versions of package archives.

Additional storage consumers included Ollama data, Flatpak installations, and system journal logs. These were not included in the cleanup.

Inode usage was only 7%, so inode exhaustion was not the cause.

## 6. Remediation

### 6.1 Docker cleanup

Docker-managed cleanup reclaimed approximately 43.76 GB.

After cleanup:

- Docker reported zero images and zero containers after cleanup.
- Two persistent volumes remained.
- containerd storage decreased from approximately 42 GB to 22 MB.
- No manual deletion was performed inside `/var/lib/containerd`.

Persistent volumes were deliberately preserved because they may contain application data.

### 6.2 Pacman cache cleanup

The cache was cleaned using:

```bash
sudo paccache -r
```

The default policy retains the three most recent cached versions of each package.

The cache decreased from approximately 20 GB to 12 GB.

## 7. Post-Cleanup Verification

Commands:

```bash
df -hT /
docker system df
sudo du -sh /var/lib/containerd
sudo du -sh /var/cache/pacman/pkg
```

Results:

| Metric | Before | After |
|---|---:|---:|
| Root filesystem used | 324 GB | 275 GB |
| Available space | 102 GB | 152 GB |
| Root usage | 77% | 65% |
| containerd storage | 42 GB | 22 MB |
| Pacman cache | 20 GB | 12 GB |

The two Docker volumes remained present:

| Volume | Links | Size |
|---|---:|---:|
| english-platform_postgres_data | 0 | 67.36 MB |
| english-platform_redis_data | 0 | 89 B |

A link count of zero means no current container references the volume. It does not prove that its data is disposable.

## 8. Operational Lessons

- Filesystem capacity and inode availability are separate resources.
- `df` reports filesystem-level usage, while `du` estimates usage by traversing directory entries.
- Docker images can share layers, so displayed image sizes must not simply be added together.
- Docker manages containerd storage; manually deleting containerd internals can damage runtime state.
- Package cache cleanup should follow the package manager's retention policy.
- Persistent volumes must be treated as potentially valuable application data.
- Cleanup should be followed by independent verification.

## 9. Limitations

- This was a local storage investigation, not a production outage.
- No filesystem corruption was observed.
- No filesystem repair or `fsck` operation was performed.
- Docker cleanup removed unused image/container data; persistent volumes were preserved.
- Ollama data, Flatpak data, and journal logs were not cleaned.
- The investigation did not include database recovery or application-level validation.

## 10. Final Result

Root filesystem usage decreased from approximately 77% to 65%.

Available space increased from approximately 102 GB to 152 GB.

The cleanup was verified using filesystem, Docker, and directory-level measurements.
