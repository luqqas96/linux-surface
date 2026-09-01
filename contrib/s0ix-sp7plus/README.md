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

## 4. Diagnosing wakeups and S0ix residency

`99-log-wakeup` is a `systemd-sleep` hook that appends one entry per
suspend/resume cycle to `/var/log/wake-diagnostics.log`.

```bash
sudo install -m 755 99-log-wakeup /usr/lib/systemd/system-sleep/
```

A cycle looks like this:

```
=== 2026-09-01 22:20:05 SUSPEND (suspend) ===
  --- LTR: IP blocks advertising a latency requirement ---
    XHCI               non-snoop=0         snoop=499712    (ns)
    AGGREGATED_SYSTEM  non-snoop=0         snoop=1047552   (ns)
=== 2026-09-01 22:20:15 RESUME (suspend) ===
  wake IRQ: 14
      14:  10  ...  IR-IO-APIC  14-fasteoi  INT34C5:00
  suspended for : 10 s
  hardware sleep: 9 s
  S0ix residency: 90%
  slp_s0 delta  : 9 s
  --- substate residency delta ---
    S0i3.0     9 s
```

Two different failures show up here, and they are easy to confuse:

**Something is waking you up.** The `wake IRQ` line and the wakeup-source
table name the culprit. Note that `wakeup_count` is cumulative since boot, so
compare it across resumes rather than reading it as a per-cycle value.

**Something is stopping you from staying asleep.** Compare `S0ix residency`
against the wall-clock time. A machine suspended for eight hours that only
reached hardware sleep for one is not being woken up -- it is failing to stay
in S0ix, and no amount of chasing wake sources will fix that. `slp_s0 delta`
corroborates the number from an independent PMC counter, and the substate
delta shows whether the time went to the deep state (`S0i3.x`) or a shallow
one.

The `LTR` block is the first place to look for the second failure. Each IP
block advertises how long it tolerates being unattended; a block asking for
tens of microseconds keeps the SoC out of substates whose exit latency is
measured in milliseconds. These values are sampled while the system is still
awake, so treat them as a lead rather than proof.

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
hardware reached S0ix for roughly one hour out of ten. Short suspends are
fine, and the substate delta shows the time does go to `S0i3.0`, so the SoC
reaches the deep state and then keeps leaving it -- without generating a
system wakeup, which is why the wake log looks clean while the battery drains.

The wake loop is fixed; this is a separate, still unexplained problem. The
`LTR` block logged at suspend time is the most promising lead, but I have not
confirmed a culprit, so this README does not name one. If you are chasing
battery drain during suspend, install the hook in section 4 first and check
the residency number before assuming wakeups are to blame.
