# S0ix / suspend tweaks for the Surface Pro 7+

Configuration bits that improve `s2idle` (S0ix) behaviour on the **Surface
Pro 7+** (Tiger Lake-UP3). The wakeup fix in particular is the important one:
without it the device wakes itself up repeatedly while suspended.

Everything here was tested on a Surface Pro 7+ running Fedora with kernel
`6.19.8-3.surface`. Most of it is not Pro 7+ specific and should apply to
other Tiger Lake Surface devices, but the device ids differ -- check them
before copying (see the comments in each file).

| File | Goes to |
|---|---|
| `99-no-wakeup-als.rules` | `/etc/udev/rules.d/` |
| `50-nvme-runtime-pm.rules` | `/etc/udev/rules.d/` |
| `disable-wake-sources.service` | `/etc/systemd/system/` |
| `99-log-wakeup` | `/usr/lib/systemd/system-sleep/` (must be executable) |

Treat these as examples: read the comments and adapt them rather than copying
them blindly.

## 1. Spurious wakeups from the ambient light sensor

**Symptom.** The device wakes itself a few seconds after suspending, over and
over, draining the battery in a closed bag. `/sys/power/pm_wakeup_irq` reports
IRQ 14, which on this machine is `INT34C5:00` -- the Tiger Lake pin
controller.

**Cause.** The APDS9960 ambient light / proximity sensor (`i2c-MSHW0184:00`)
has wakeup enabled by default. Its interrupt reaches the system through the
pin controller and is treated as a wake event, so proximity changes wake the
machine. Disabling adaptive brightness in the desktop environment does not
help, because the sensor is still armed at the kernel level.

**Fix.** Install `99-no-wakeup-als.rules`, then reboot or run:

```bash
sudo udevadm control --reload
sudo udevadm trigger --subsystem-match=i2c
```

Verify:

```bash
cat /sys/bus/i2c/devices/i2c-MSHW0184:00/power/wakeup   # -> disabled
```

Note that IRQ 14 is shared: the pin controller also carries legitimate wake
sources, so seeing IRQ 14 as the wake reason after this fix is normal and
expected. What goes away is the runaway wake loop.

## 2. NVMe runtime power management

The KIOXIA BG4 in this machine comes up with `power/control=on`, which keeps
the PCIe link from entering its low-power states while idle. Install
`50-nvme-runtime-pm.rules` and check with:

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/control   # -> auto
```

The vendor/device ids in the rule are specific to that SSD; the Pro 7+ shipped
with more than one model, so confirm yours with `lspci -nn | grep -i nvme`.

## 3. Unnecessary ACPI wake sources

`PEG0` (the PCIe bridge to the NVMe) and `TXHC` (the Thunderbolt 4 controller)
are wakeup-enabled by default. Neither is useful on a tablet with no external
Thunderbolt peripherals:

```bash
sudo cp disable-wake-sources.service /etc/systemd/system/
sudo systemctl enable --now disable-wake-sources.service
grep -E 'PEG0|TXHC' /proc/acpi/wakeup    # -> both *disabled
```

Writing a name to `/proc/acpi/wakeup` toggles it rather than setting it, so
the unit checks the current state first and is safe to run repeatedly. Drop
the `TXHC` line if you do wake the machine from Thunderbolt devices.

## 4. Diagnosing wakeups

`99-log-wakeup` is a `systemd-sleep` hook that appends one entry per
suspend/resume cycle to `/var/log/wake-diagnostics.log`: the wake IRQ and its
`/proc/interrupts` line, wakeup sources with a non-zero `wakeup_count`, and
how long the hardware actually stayed in S0ix.

```bash
sudo cp 99-log-wakeup /usr/lib/systemd/system-sleep/
sudo chmod +x /usr/lib/systemd/system-sleep/99-log-wakeup
```

The interesting number is `hardware sleep` versus the wall-clock time between
the `SUSPEND` and `RESUME` lines. If the machine was suspended for eight hours
but only slept for one, something is keeping the SoC out of S0ix -- that is a
different problem from being woken up, and this log is how you tell them
apart.

The log is not rotated; add a `/etc/logrotate.d/` snippet if you intend to
leave it installed long-term.

## Results

Measured over 114 suspend/resume cycles across 15 days of daily use, with all
of the above applied:

| | Result |
|---|---|
| Cycles where the machine suspended but failed to sleep | 2 of 114 |
| Hardware sleep residency, short suspends (minutes) | 99-100% |
| Hardware sleep residency, long suspends (> 2 h) | 10-48% |
| Longest single hardware sleep observed | ~1.16 h |

Before the ALS fix the device could not stay suspended for more than a few
seconds at a time; that behaviour is gone. These numbers are from the fixed
configuration only -- the original failure was not instrumented, so there is
no measured "before" column to compare against.

### Known limitation

Long suspends still show poor S0ix residency: on an overnight suspend the
hardware reached S0ix for roughly one hour out of ten. The wake loop is fixed,
but something on this platform still limits sustained hardware sleep, and the
cause is not identified yet. If you are chasing battery drain during suspend,
install the hook in section 4 first and check that number before assuming
wakeups are to blame.
