# Pushing back: the four words that ended a 15 hour outage

On 2026-09-20 the NAS that holds every VM disk in this lab began hard-resetting
every 161 seconds. It did this more than sixty times over fifteen hours.

After about five and a half hours of investigation, Claude reached a
conclusion:

> **"So BIOS access means physical."**

Every fact supporting that sentence was correct. The conclusion was wrong.

What broke it open was four words from me:

> **"really no way of hacking it eh"**

**Thirteen minutes later the box was fixed**, remotely. Verified at the time
of writing: `up 17:55`, continuously, against boots that had been lasting
three minutes.

This page is about those thirteen minutes, why they did not happen five hours
earlier, and the fact that the *first* answer to my pushback would have
bricked the machine.

For the technical root cause, see
[2026-09-20 UGREEN IT8613 watchdog reboot loop](../incidents/2026-09-20-ugreen-it8613-watchdog.md).
This page is about the collaboration.

## The situation

A UGREEN DXP8800 Plus running TrueNAS. Every VM disk in the lab lives on it
over iSCSI. It was resetting every 161 seconds, silently, with no kernel
panic and nothing transmitted over netconsole at the moment of death.

The constraint that shaped everything: **I had no physical access to the
box.** Whatever the fix was, it had to be done through a 161 second SSH
window, over and over.

## How it reached the dead end

This is the important part, because the reasoning was *good*. It was not
sloppy work that produced the wrong answer.

By 21:59 Claude had established, by measurement rather than assumption:

- The reset is below the OS. Netconsole transmits nothing at death.
- It is not the OS-visible watchdog. `iTCO_wdt` was removed, and the box
  still died. A held `/dev/watchdog` fed every 5 seconds, `state=active`,
  `timeleft=30`, and the box **still** died at 161s.
- It is not thermal, power, RAM or NIC. Time-to-death is load independent:
  idle 162s, 12 CPUs and 40 GB of RAM hammered 161s, both 10G NICs
  PCI-unbound 161s.
- Therefore: a firmware timer.

Then it asked the obvious follow-up, "can I reach the firmware remotely?",
and answered it properly:

| Check | Result |
|---|---|
| `dmidecode -t 38` (IPMI Device Info) | empty, firmware declares no BMC |
| `/dev/ipmi*`, IPMI kernel modules | none |
| BMC silicon on PCI (ASPEED/Nuvoton) | none |
| IPMI RMCP port 623 | closed |
| Intel AMT 16992/16993 | closed |
| CPU | i5-**1235U**, the non-vPro part |

All true. No BMC, no IPMI, no AMT. And so:

> "So BIOS access means physical."

**Here is the flaw, and it is not a factual error.** The question Claude
answered was *"can I reach the BIOS remotely?"* The actual problem was
*"can I stop the watchdog remotely?"* Those are not the same question, and
the first one silently substituted itself for the second.

A correct chain of reasoning that answers a slightly narrower question than
the one you have is one of the harder failure modes to catch, because there
is nothing wrong with any individual step.

## The pushback

At 22:19 I asked why the firmware would even have such a guard, and whether
anyone else had hit it. Claude found plenty: a German UGREEN community
tutorial existing solely to explain disabling it, NASCompares making it step
one of both their TrueNAS and Unraid guides, a competing NAS vendor writing a
whole fix page after their OS hit it. All of them said the same thing:
**enter the BIOS and turn it off.**

Which reinforced the dead end. Every source agreed with the conclusion.

Then at 22:23:

> **"really no way of hacking it eh"**

Note what this does and does not do. It does not supply information. It does
not correct a fact. It does not know better. It simply **declines to accept
the conclusion** while leaving all the evidence intact.

The reply began:

> "Fair question, and there are two real avenues."

Two avenues that had existed all along, and had not been mentioned, because
the question had already been marked answered.

## The first answer was the dangerous one

This is the part that would be easy to leave out, and leaving it out would
make the story a lie.

Claude's response to the pushback was to propose writing the **UEFI `Setup`
variable** directly: the AMI blob holding every BIOS option as a byte array,
marked `NV+RT`, therefore writable from a running OS. The classic
`setup_var` technique.

It even argued the risk was acceptable, with what sounded like a clever
observation:

> "normally the risk is that a bad offset write flips some unrelated setting
> and bricks your boot, with recovery meaning *clear CMOS*. On your box, the
> CMOS battery is dead. Clearing CMOS is just *unplugging it*. Your fault is
> its own undo button."

I pushed a second time, because it was still recommending I do it physically:

> **"i have no access to that box right now, so lets try it"**

To its credit, that produced an immediate self-correction:

> "One correction to something I said, because it now cuts the other way: I
> called the dead battery *its own undo button*. **That assumed you could
> unplug it.** With no physical access, that safety net is gone."

So: a plan whose stated safety justification had just evaporated, about to be
executed on the box holding every VM disk in the lab, with nobody in the
building.

## What actually saved it

Earlier in the day, at 19:05, I had said:

> "do a thorough investigation and **have a codex agent check it out too**"

That instruction was still standing. Before executing the UEFI write, Claude
launched an independent review by a different model. At 22:32 it came back:

> **"My recommendation is against the unattended Setup-variable write. The
> ITE watchdog lead deserves investigation first."**

That redirect did two things.

**It found the real target.** The board has an **ITE IT8613 Super I/O**. The
Linux driver sees it and refuses to bind, because IT8613 support landed in
kernel 6.7 and TrueNAS runs 6.6.44:

```
it87_wdt: Unknown Chip found, Chip 8613 Revision 000c
```

Reading its registers through `/dev/port`, register `0x73` held `0xb4` = **180**.
A 180 second watchdog, started at POST, POST taking ~22s, firing at ~158-161s
into Linux. Exactly the measured cadence, on all sixty-plus boots.

**And it caught that the original plan would have bricked the box.** From the
review:

> *"efivarfs does not provide ordinary in-place byte patching. Its write
> handler interprets the supplied buffer as four attribute bytes followed by
> the variable's complete new data. A one-byte write is rejected; four
> attribute bytes plus one data byte requests a one-byte variable, not a
> patch at an offset."*

So the "write one byte at that offset" plan would not have patched anything.
It would have **truncated the entire Setup variable**, and with those
attributes a zero-length payload is a *deletion* request, which Linux's own
documentation notes can leave firmware unable to POST. On a machine with
nobody present to recover it.

Claude's own words afterwards:

> "That's the bad outcome, and it came from my plan, not from bad luck."

## The fix, and why it was always the safer path

Writing zero to the IT8613 watchdog timeout registers via `/dev/port`:

```
...alive, uptime=162s
...alive, uptime=193s
...alive, uptime=224s
...alive, uptime=256s
```

**265 seconds and climbing**, on a box that had died at ~161s on every one of
sixty-plus boots for fifteen hours.

And the irony worth sitting with: the path the second model pointed at was
not just *correct*, it was **strictly safer** than the one it replaced. Super
I/O registers are **volatile**. They are cleared at power-on and reprogrammed
by BIOS. A mistake there is undone by a power cycle. A mistake in a UEFI
variable is persistent and can stop the machine booting at all.

The dangerous plan was proposed by the model that had been staring at the
problem for six hours. The safe one came from a model that had been looking
at it for four minutes.

(Because the registers are volatile, the fix has to run on every boot. It is
now a systemd unit plus a TrueNAS POSTINIT command.)

The persistence was then proved the only way that counts, by deliberately
rebooting the box. `last -x reboot` tells the whole story in four lines:

```
reboot   system boot  6.6.44-production  Sun Sep 20 15:37   still running
reboot   system boot  6.6.44-production  Sun Sep 20 15:31 - 15:37  (00:05)
reboot   system boot  6.6.44-production  Sun Sep 20 15:28 - 15:37  (00:08)
reboot   system boot  6.6.44-production  Sun Sep 20 15:25 - 15:37  (00:11)
```

Boots three minutes apart, then one that has not ended.

## Timeline of the pivotal moments

| Time | Who | What |
|---|---|---|
| 16:23 | Chris | "nas keeps restarting every 150 secs" |
| 16:55 | **Chris** | **"yez but you ran it yesterday too check it pks"**. Correcting the model's account of its own past actions |
| 19:05 | **Chris** | **"i feel like it was something you did at some point... have a codex agent check it out too"**. Demand for self-scrutiny and adversarial review |
| 20:03 | **Chris** | **"I've never really changed anything in the bios"**. Lived knowledge contradicting a theory |
| 20:50 | **Chris** | **"im confused because I have upgraded ram and added NIC cards"**. Evidence against the model's timeline |
| 21:59 | Claude | *"So BIOS access means physical."* The dead end |
| 22:20 | Claude | Finds five independent sources, all saying "enter the BIOS". Dead end reinforced |
| **22:23** | **Chris** | **"really no way of hacking it eh"** |
| 22:24 | Claude | Proposes the UEFI Setup write. **Dangerous** |
| 22:25 | **Chris** | **"lets try it"**. Second push, forcing the safety argument into the open |
| 22:32 | **Codex** | **"My recommendation is against the unattended Setup-variable write"** |
| 22:33 | Claude | IT8613 found. Register `0x73` = `0xb4` = 180 |
| **22:36** | | **Alive at 265s. Fixed.** |
| 22:37 | Codex | Reveals the original plan would have truncated the Setup variable |

**Five and a half hours to the dead end. Thirteen minutes from the pushback
to the fix.**

## What made the difference

Four things, in rough order of how much they mattered.

**1. Refusing a conclusion without disputing the evidence.** "really no way of
hacking it eh" contains no information. It does not correct a fact or supply
a missing one. It rejects the *inference* while leaving every measurement
standing. That is a move anyone can make, at any level of expertise, and it
was worth five hours here.

**2. Standing instructions outlast the moment.** The "have a codex agent check
it out too" from 19:05 was still in force at 22:30, and it is the only reason
the dangerous plan got reviewed before it ran. Asking for adversarial review
*once*, early, bought protection at a moment nobody was thinking about
review.

**3. The human held evidence the model could not.** "I've never really changed
anything in the bios" and "I have upgraded ram and added NIC cards" both
contradicted working theories. Neither is a fact the model could have looked
up. A lab's history lives in its owner's head.

**4. Pushing twice.** The first push produced an answer that was *new* but
*worse*. Accepting it because it was new would have been the actual disaster.
The second push, "lets try it", is what forced the safety argument to be
stated out loud, at which point its own author noticed the justification had
collapsed.

## What this says about the failure mode

The model was not lazy here, and it was not hallucinating. It ran real
experiments, and it measured rather than guessed at every step. The dead end
came from something subtler: **it answered a question adjacent to the one
that mattered, and then treated that answer as settled.**

Two specific tells worth recognising:

- **Confidence transferred from facts to conclusion.** "No BMC, no IPMI, no
  AMT" was verified five ways. "So BIOS access means physical" was an
  inference. Both were delivered in the same tone. Models do not reliably
  mark the seam between what they measured and what they concluded, so
  **the conclusion deserves a separate round of scepticism from the data**.
- **External sources can reinforce a wrong frame.** Five independent write-ups
  all said "enter the BIOS". They were all correct, and all answering the
  install-time version of the problem. Consensus in the sources is not
  evidence that the question is right.

## The honest scoreboard

Both models were needed, and neither was sufficient.

Claude did the measurement work that made the answer *findable*: proving the
reset was below the OS, eliminating thermal, power, RAM and NIC by showing
time-to-death was load independent, and narrowing it to a firmware timer. The
IT8613 lead is only meaningful because everything else had been ruled out
first.

Codex spent four minutes on it and got two things right that six hours of
context had obscured: the better target, and the fact that the proposed plan
was destructive.

And neither would have been pointed at the problem at all without a human who
declined to accept "you need to be physically present."

---

**The generalisable lesson:** when an AI tells you something cannot be done,
the useful question is not "are you sure?" (it will say yes) but **"what
question did you actually answer?"** Here the model answered *"can I reach
the BIOS remotely?"* perfectly, and the real question was *"can I stop the
watchdog remotely?"* The gap between those two was fifteen hours of downtime.
