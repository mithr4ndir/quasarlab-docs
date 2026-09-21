# 2026-09-20 The NAS rebooted every 161 seconds for 15 hours, and it was a watchdog I had already ruled out

The NAS hard-reset roughly sixty times between 07:19 and 15:30 PDT, taking every
VM in the lab with it. The cause was a vendor watchdog in the Super I/O chip that
UGREEN's own OS feeds and TrueNAS does not. I spent most of the evening proving it
was not a watchdog, because I tested the wrong one.

!!! warning "Read this one for the method, not the fix"
    The fix is four lines of register writes. Getting there took six wrong
    hypotheses, three of which I stated with more confidence than the evidence
    supported. The controlled test that settled it took ten minutes and I ran it
    last. That ordering is the lesson.

## Symptom

`journalctl --list-boots` on the NAS showed 54 boots. The box answered ping, then
stopped, then answered again, on a metronome:

```
boot start   22:26:01   22:29:05   22:32:09   22:35:13
delta               184s      184s      184s
```

Each boot the journal ended **mid-line**. No `Stopping` units, no `Reached target
Shutdown`, no reboot record. Not a reboot. A hard reset.

## Impact

Every VM disk in the lab is on NAS iSCSI, so this was total:

| | |
|---|---|
| VMs down | all of them except `command-center1` (boot disk on pve2-local) |
| Kubernetes | API dead, no etcd quorum |
| Duration | ~15 hours, 07:19 to 15:30 PDT |
| Data loss | none, pool `tank` ONLINE and clean throughout |

## What I ruled out, and how

Absence of evidence kept looking like evidence, so each of these needed a test
rather than a log grep.

**Kernel panic.** The on-disk journal had none, but journald buffers, so a panic
may simply never reach disk. I set up netconsole (kernel log over UDP to another
host) and re-armed it on every boot with a reconnect loop. Netconsole transmits
from panic context, which is its entire purpose.

Result: **nothing**. Not one packet in the final seconds of any boot, and no
kernel output at all between 38s and death. That excludes panic, oops, OOM and
thermal shutdown in one shot, and it is why I started looking below the OS.

**UPS and mains.** `ups.status OL`, battery 100%, input 120.0V, load 39% of 900W.

**Temperature.** CPU package 49C at 130s in, crit 100C. NVMe 37-45C, SATA 34-36C.

**Storage.** `zpool status -x` clean, zero ATA errors, and the box still reset with
`iscsitarget` and `nfs` both stopped.

**EDAC memory errors.** `EDAC igen6 MC0/MC1: HANDLING IBECC MEMORY ERROR` looked
like a smoking gun for about ninety seconds. It is a false positive: it fires on
**both** controllers at the identical timestamp at driver probe with
`ADDR 0x7fffffffe0` (an all-ones default), and every DIMM reports
`ce_count=0 ue_count=0`. Check the counters before believing the message.

## The wrong turn worth describing

The watchdog looked innocent and then looked guilty and then was innocent, twice,
for different reasons.

`wdctl` reported an armed `iTCO_wdt` with `timeout=30`, `timeleft=29`. That is
wrong in a specific and instructive way: **`wdctl` itself opens `/dev/watchdog`**,
and `MAGICCLOSE` is `0` on this board, so merely querying the device can arm a
30-second reset. The sysfs view told the truth, `state=inactive`. My own
diagnostic had manufactured the evidence I was reading.

So I unloaded it: `modprobe -r iTCO_wdt`. Box still reset at ~161s. **Watchdog
excluded**, I said, and went back to hunting a hardware fault.

That conclusion was wrong, and the error is worth naming precisely: unloading
`iTCO_wdt` excludes *the watchdog Linux can see*. It says nothing about any other
timer on the board. I had tested the decoy.

Then I found UGREEN's documented ~3 minute watchdog, matched it against my
measured 182-184s, and promptly over-claimed again. Chris pushed back with the
observation that settled it: he had never changed a BIOS setting, so a CMOS wipe
restoring defaults should restore a *working* config, not a broken one. Fair, and
my write-up had already recorded the theory as confirmed. It was not.

## The test that should have been first

Stop reasoning about mechanism. Measure whether the deadline moves.

| Condition | Time to death |
|---|---|
| Idle | 162s |
| 12 CPUs pegged + 40GB RAM walker | 161s |
| Both Aquantia 10G NICs PCI-unbound | 161s |

One second of spread across full CPU load, a memory walk, and removing the
aftermarket NICs entirely. A marginal PSU, a failing VRM, bad RAM or a thermal
problem would all shorten that sharply under load. **Hardware faults are
load-dependent. Timers are not.**

That single table eliminated power, thermal, VRM, RAM and the NICs, and it
eliminated them *together*, in about ten minutes. It also exonerated the
aftermarket RAM and 10G cards, which had been prime suspects purely because they
were the non-stock parts.

Then, feeding the TCO watchdog properly (holding `/dev/watchdog` open,
`state=active`, `timeleft=30`, refreshed every 5s) and watching it die anyway at
161s proved the killer was a *different* timer. Not absent. Different.

## Root cause

The board has an **ITE IT8613 Super I/O**, and its watchdog is armed by BIOS to
180 seconds. UGOS feeds it. TrueNAS has no idea it exists.

```
# modprobe it87_wdt
it87_wdt: Unknown Chip found, Chip 8613 Revision 000c
```

The driver *sees* the chip and refuses to bind, because IT8613 support landed in
kernel 6.7 and TrueNAS SCALE 24.10.2.4 runs 6.6.44. So the one driver that could
have managed this watchdog declines to, on this exact kernel.

Reading Super I/O LDN `0x07` through `/dev/port`:

```
Chip ID      : 0x8613
0x71 WDT ctrl: 0x00
0x72 WDT cfg : 0x80     (bit7 -> seconds)
0x73 TMO     : 0xb4     = 180
```

`0xb4` is 180. POST takes ~22s, so a 180s timer armed at POST fires ~158-161s into
Linux. That is the number I had been measuring all night.

### Why it started that day

BIOS had the watchdog disabled, almost certainly set during the TrueNAS install
(every TrueNAS-on-UGREEN guide makes "disable the Watchdog Timer" step one, since
you cannot stay in the BIOS menu for three minutes with it on).

The CR2032 is dead. Warm reboots and soft shutdowns do **not** exercise the
battery, because the PSU 5V standby rail holds CMOS while the unit stays plugged
in. Only full AC removal does. A cold power-off that morning was the first real
test the battery had faced in months, it failed, and BIOS reverted to defaults:

```
rtc_cmos: setting system clock to 2023-01-01T00:01:03 UTC
```

`2023-01-01T00:01:03` is the "no valid time in RTC" default. Not drift. Total loss.

## Fix

Super I/O registers are volatile and BIOS re-arms 180 on every boot, so the
workaround has to run every boot.

```python
BASE, DATA, LDN_WDT = 0x2e, 0x2f, 0x07
# unlock: 0x87,0x01,0x55,0x55 -> 0x2e ; select LDN 7 ; zero 0x73/0x74 ; exit 0x02,0x02
```

Deployed in two layers that deliberately cover each other's gap:

| Layer | Runs | Survives |
|---|---|---|
| `ugreen-wdt-disable.service` (`WantedBy=sysinit.target`) | early, every boot | reboots. Lost on a TrueNAS OS upgrade |
| TrueNAS POSTINIT command | later, every boot | OS upgrades, stored in the config DB and captured in the config-git backup |

The script refuses to write unless the chip reports `0x8613`.

Verified the only way that counts: rebooted the NAS and watched a **fresh boot**
clear 405 seconds unattended, where every previous boot died at 161.

## Blast radius that looked like separate incidents

**The Kubernetes cluster came back with no quorum.** Not a k8s problem. The
QDevice VM on the NAS gets a **new random MAC on every boot**. Sixty reboots meant
sixty DHCP requests under sixty MACs, which chewed through the pool and took
`192.168.1.90` and `.91`. Those belong to k8cluster1 and k8cluster3. The nodes
came back on pool leftovers while etcd stayed pinned to the originals.

pfSense recorded it plainly:

```
arp: 192.168.1.16 moved from 00:a0:98:5e:dc:12 to 00:a0:98:50:30:53 on igb1
arp: 192.168.1.16 moved from 00:a0:98:50:30:53 to 00:a0:98:3e:96:79 on igb1
```

Six other VMs drifted the same way: npm `.150`→`.192` (which silently killed all
external access), nginx1, nginx2, timescaleDB, musicbot, uptime-kuma.

The four VMs that did **not** drift were exactly the four whose Terraform stacks
set `ipconfig0` explicitly. Everything else inherited the module default
`ip=dhcp`.

## What I did not change, and why

- **BIOS watchdog left enabled.** The proper fix needs physical access. The
  software workaround is verified and the trip is already required for the battery.
- **`blockpriv`/`blockbogons` left on WAN.** They were enabled that morning and
  correlated suspiciously with a VPN complaint, but the laptop's public IP is
  ordinary `24.x.x.x`, so neither rule can match it. Correlation, not cause. I do
  not undo security hardening on a theory I have already falsified.
- **`terraform fmt` not run.** Those files did not satisfy `fmt` beforehand, so
  formatting would bury a 6-line change in ~100 lines of churn.

## Follow-ups

- [ ] Replace the CR2032 (ansible-quasarlab#196). Until then every power cut
      re-rolls BIOS state.
- [ ] Pin the QDevice VM's MAC (ansible-quasarlab#197). This is the actual cause
      of the IP theft and it recurs on any NAS reboot.
- [ ] Codify the watchdog disable in Ansible (ansible-quasarlab#198). Runtime must
      stay Ansible-independent.
- [ ] Move the QDevice off the NAS and the cloud-init drive off iSCSI
      (ansible-quasarlab#199).
- [ ] PBS hourly backups for command-center1 on pve-local (ansible-quasarlab#201).
- [x] Pin static IPs in Terraform (terraform-quasarlab#25), otherwise the next
      `apply` reverts them to DHCP.

## Lessons

**"I disabled the watchdog and it still reset" only excludes the watchdog you can
see.** Appliance vendors ship firmware watchdogs on separate silicon. Check the
vendor documentation for the specific hardware before concluding "hardware fault,
replace the PSU".

**Load-dependence separates hardware faults from timers in ten minutes.** Vary the
load, measure the deadline. If it does not move, stop looking at power and thermals.

**A number that matches is not a confirmed mechanism.** UGREEN's documented
"3 minutes" matched my measured 182-184s so neatly that I wrote it into the record
as root cause before testing it. The match was real; the mechanism I attached to
it was wrong.

**`wdctl` arms the watchdog it reports on.** On hardware without `MAGICCLOSE`,
reading the state changes the state.

**Journal boot timestamps are not uptimes.** `journalctl --list-boots` first/last
entries gave me "67-149s survival", which I over-read as a tight interval. The
honest measurement is boot-to-boot wall clock, which was a metronomic 182-185s.
