# Safety Logic

[Main Guide](README_EN.md) · [Deutsch](SAFETY.md)

This document explains which conditions merely block a start, which isolate a
single irrigation channel, and which latch the entire system. It does not
replace mechanical overflow protection, leak detection, or certified
electrical protection.

## Core rules

- A fault of one plant valve should lock only that plant. P1 through P4 use
  identical logic.
- Faults of shared actuators block all new starts.
- If an actuator that may be on cannot be proven off, the complete system is
  latched.
- A global latch can only be acknowledged after the critical condition is
  fixed and `All Actuators Confirmed Safely Off` is on.
- Warnings and critical events are written to the Home Assistant logbook and
  to the persistent `plantinator_bewasserung_sicherheit` notification.

## Confirmation of Wi-Fi actuators

**Switch Feedback Max. (not runtime)** is a timeout for the state report of a
Wi-Fi switch. It is not an irrigation duration. The following `turn_on`
procedure applies only when opening the shared main valve or a plant valve.
With the defaults of eight seconds and a two-second retry delay, it works as
follows:

1. Send `turn_on` and wait no longer than eight seconds from that command for
   a fresh `on` report.
2. If it is missing, send `turn_off` and wait no longer than eight seconds from
   that command for `off`. If the valve is already reliably `off`, this step
   completes immediately.
3. Wait two seconds.
4. Send a second `turn_on` and start a new confirmation window of up to eight
   seconds.
5. If that also fails, switch off again and verify safe `off` with its own
   confirmation window of up to eight seconds.

The eight seconds therefore do **not** begin at the first switch-off attempt
and are not one shared total timeout. Every checked valve command gets an
independent window.

The pump deliberately works differently: its calculated runtime starts
immediately with the pump `turn_on`. At the same time, the system waits for a
**fresh** `on` report, but only until the earlier end of the feedback timeout
or calculated runtime. A three-second shot therefore does not receive eight
additional seconds. If `on` is confirmed after one second, two seconds of
requested runtime remain. If confirmation is still missing at shot end, the
pump is switched off immediately, **0 ml** are recorded, and the shared pump
is temporarily locked. The pump is not started a second time because missing
feedback could otherwise cause an unknown extra volume.

At the calculated shot end, `turn_off` is sent immediately. Only **after that
off command** may the system wait up to eight seconds for the confirmed safe
`off` report. This safety window does not extend the shot or the pump's
requested on-time.

## Fault classes and response

| Condition | Response | Recovery | iPhone |
|---|---|---|---|
| Invalid mapping, profile, sensor age, or water level before a run | Start blocked | automatic after correction | no critical alert |
| Plant valve unavailable | affected plant locked only | automatic after at least 60 s once reachable and `off` | time-sensitive warning |
| Both plant-valve opening attempts fail, but `off` is confirmed | affected plant locked only | automatic after at least 60 s with reliable `off` | time-sensitive warning |
| Main valve unavailable or fails both opening attempts, but `off` is confirmed | all new starts blocked without global alarm latch | automatic after at least 60 s with reliable `off` | time-sensitive warning |
| Pump start has no fresh `on` confirmation, but safe `off` is confirmed afterwards | 0 ml recorded and all new starts blocked by the pump lock | automatic after at least 60 s once the pump is reachable and reliably `off` | time-sensitive warning |
| Wetback target too low, invalid wetback sensor, or noncritical top-up failure | sequence ends or top-up is skipped | automatic on a later valid cycle | time-sensitive warning |
| Pump, plant valve, or main valve cannot be confirmed `off` after a run | safe stop and global latch | manual only after correction and confirmed safe off | critical alert |
| Pump unexpectedly on or pump on without an open plant valve | safe stop and global latch | manual only | critical alert |
| Plant valve unexpectedly open or multiple plant valves open | safe stop and global latch | manual only | critical alert |
| Required actuator becomes `unknown` or `unavailable` during a run | safe stop and global latch | manual only | critical alert |
| Water level becomes unsafe during a water process | safe stop and global latch | manual only | critical alert |
| Wetback peak exceeds the hard upper limit | immediate safe stop and global latch | manual only | critical alert |
| Safe stop or startup check cannot prove that all mapped actuators are off | global latch | manual only after physical inspection | critical alert |

## Plant locks

Under **Monitoring → Actuator Locks and Run State**, the dashboard shows a
lock and reason for every plant. A locked channel is skipped by automatic
plant selection. Other plants remain available while the shared pump and main
valve are safe.

A shared main valve and the shared pump cannot be assigned to one plant.
Their soft actuator locks therefore block all new starts but do not globally
latch the system while `off` is confirmed.

## Acknowledging a global latch

1. Turn the main switch off or enable the manual system lock.
2. Read the alarm reason, latest safety event, and actuator states.
3. Physically inspect the pump, main valve, every plant valve, and hoses.
4. Repair mapping or connectivity and wait until every actuator is available.
5. Run **Safe Stop**. `All Actuators Confirmed Safely Off` must then be on and
   Critical System Status must be `ok`.
6. Press **Acknowledge Alarm**.
7. Retest with a small supervised volume.

Acknowledgement is rejected while a critical condition remains or safe off
has not been confirmed.

## Event log

Every safety event has a level (`info`, `warnung`, or `kritisch`), code,
details, and timestamp. These values appear on the Monitoring dashboard. The
Home Assistant logbook provides the timeline, while the persistent
notification shows the latest entry. In addition, **Debug → Last 20 Errors**
persistently keeps the newest warnings and critical events independently of
the optional debug file logger. Informational events are intentionally omitted
from that list.

The notify service configured under **Monitoring Settings** receives:

- warnings with the iOS `time-sensitive` interruption level;
- global critical latches with `critical`, sound, and full volume.
