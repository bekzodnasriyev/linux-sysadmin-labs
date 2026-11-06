# Linux systemd Service Troubleshooting

## 1. Objective

This lab investigates how systemd manages Linux services, how service dependencies are represented, how logs are collected, and how service failures can be diagnosed using systemd tools.

The investigation combines read-only analysis of a running NetworkManager service with a controlled service failure.

## 2. Environment

- OS: Arch Linux
- Init system: systemd
- Service investigated: NetworkManager
- Diagnostic tools: `systemctl`, `systemd-analyze`, `journalctl`, `nmcli`
- Failure simulation: `systemd-run` and `/usr/bin/false`

## 3. systemd Architecture

systemd is the system and service manager. It runs as PID 1 and manages services, targets, mounts, sockets, and other units.

```text
                Linux Kernel
                     |
                     v
                systemd (PID 1)
                     |
             Service Management
                     |
          +----------+----------+
          |          |          |
          v          v          v
      NetworkManager dbus-broker other services
          |
          v
      Network Interfaces
```

## 4. Service State Investigation

Commands used:

```bash
systemctl status NetworkManager
systemctl is-active NetworkManager
systemctl is-enabled NetworkManager
systemctl show NetworkManager -p MainPID -p ActiveState -p SubState
```

Observed results:

- Unit loaded successfully.
- Service state: `active`.
- Substate: `running`.
- Autostart: `enabled`.
- Main process was present.

Important distinction:

- `enabled` describes boot-time activation configuration.
- `active` describes the current runtime state.

These properties are independent.

## 5. Unit File Investigation

Inspected the unit:

```bash
systemctl cat NetworkManager.service
```

Main unit location:

```text
/usr/lib/systemd/system/NetworkManager.service
```

Relevant directives:

| Directive            | Purpose                                                   |
|----------------------|-----------------------------------------------------------|
| `Wants=`             | Pulls in another unit as a weak dependency                |
| `Requires=`          | Defines a stronger dependency                             |
| `After=`             | Defines startup ordering                                  |
| `Before=`            | Defines ordering before another unit                      |
| `BindsTo=`           | Establishes a strong lifecycle relationship               |
| `Type=dbus`          | Service startup is associated with acquiring a D-Bus name |
| `BusName=`           | Specifies the D-Bus name                                  |
| `ExecStart=`         | Defines the main start command                            |
| `Restart=on-failure` | Requests restart after qualifying failures                |
| `KillMode=process`   | Applies stop signals to the main process                  |

`After=` and `Wants=` have different purposes: ordering does not itself pull in a dependency.

## 6. Unit Configuration and Runtime Dependencies

Inspected effective properties:

```bash
systemctl show NetworkManager.service \
  -p FragmentPath \
  -p DropInPaths \
  -p Type \
  -p ExecStart \
  -p Restart \
  -p After \
  -p Wants \
  -p Requires
```

Compared the declared unit configuration with effective runtime properties.

The effective dependency set can include dependencies generated automatically by systemd, not only directives written in the unit file.

Dependency trees:

```bash
systemctl list-dependencies --all NetworkManager.service
systemctl list-dependencies --reverse NetworkManager.service
```

Validated the unit configuration:

```bash
systemd-analyze verify /usr/lib/systemd/system/NetworkManager.service
```

No diagnostic output was reported by the verification command.

## 7. Journal Investigation

Collected service logs:

```bash
journalctl -b -u NetworkManager.service
journalctl -b -u NetworkManager.service -p warning
journalctl -u NetworkManager.service -n 50 --no-pager
```

Key findings:

- NetworkManager started successfully.
- Wi-Fi authentication completed.
- DHCP assigned an IPv4 lease.
- The interface reached the activated state.
- NetworkManager reported global connectivity.
- Warnings were observed on a Wi-Fi P2P interface.

The observed P2P warning was not established as the cause of a connectivity failure.

## 8. D-Bus Dependency Investigation

Inspected related services:

```bash
systemctl status systemd-resolved.service systemd-homed.service --no-pager
systemctl list-unit-files 'systemd-resolved*' 'systemd-homed*'
journalctl -b -u dbus-broker.service -p warning --no-pager
```

Findings:

- `systemd-resolved.service` was installed but inactive and disabled.
- `systemd-homed.service` was installed but inactive and disabled.
- D-Bus logged activation failures for `org.freedesktop.resolve1` and `org.freedesktop.home1`.

These observations explain why the corresponding D-Bus activation requests could not activate those services. They do not establish a NetworkManager failure.

NetworkManager remained operational without systemd-resolved in this environment.

## 9. Controlled Service Failure

Created a transient unit:

```bash
sudo systemd-run \
  --unit=portfolio-failure-demo.service \
  --property=Restart=no \
  /usr/bin/false
```

`/usr/bin/false` intentionally exits with status code `1`.

Inspected the result:

```bash
systemctl status portfolio-failure-demo.service --no-pager

journalctl -u portfolio-failure-demo.service -n 20 --no-pager

systemctl show portfolio-failure-demo.service \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p ExecMainCode \
  -p ExecMainStatus
```

Observed evidence:

```text
ActiveState=failed
SubState=failed
Result=exit-code
ExecMainCode=1
ExecMainStatus=1
```

Interpretation:

- `ActiveState=failed`: the unit entered a failed state.
- `Result=exit-code`: the main process exited with a non-zero status.
- `ExecMainCode=1`: the process termination type was `CLD_EXITED`.
- `ExecMainStatus=1`: the actual process exit status was `1`.

## 10. Root Cause Analysis

Controlled incident:

| Stage | Finding |
|---|---|
| Incident | Service entered failed state |
| Observation | `ActiveState=failed` |
| Evidence | Main process exited with status 1 |
| Test | Inspected unit status and journal |
| Root cause | `/usr/bin/false` intentionally returned exit status 1 |

This was a deliberately generated failure, not an incident involving a production application.

## 11. Cleanup

Reset the failed state:

```bash
sudo systemctl reset-failed portfolio-failure-demo.service
```

The transient unit was subsequently no longer found.

No permanent service configuration was created, and NetworkManager was not modified or restarted.

## 12. Operational Lessons

- A service being enabled does not mean it is currently active.
- A service being active does not guarantee that the application is healthy.
- Unit dependencies and startup ordering are different concepts.
- `systemctl status` provides a useful summary, while `journalctl` provides event history.
- D-Bus activation warnings must be investigated in context.
- A non-zero exit code identifies a process failure, not necessarily its underlying cause.
- A successful unit verification does not guarantee runtime functionality.

## 13. Scope and Limitations

This lab demonstrates service inspection, dependency analysis, journal investigation, and controlled failure diagnosis.

It does not demonstrate recovery from a real production service outage, application-level debugging, or a complete service restoration procedure.

Further work will investigate real service failure scenarios, restart behavior, dependency failures, and recovery verification.

## 14. Final Result

Completed a read-only investigation of NetworkManager and reproduced a controlled systemd service failure.

Evidence was collected using native systemd diagnostic tools, and the temporary test unit was cleaned up after the investigation.
