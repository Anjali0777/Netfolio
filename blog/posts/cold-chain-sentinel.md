---
title: Cold Chain Sentinel
---

## What happened

Overnight on Sunday 11 October 2026, someone took control of the refrigeration in supermarkets across France. According to [IT-Connect](https://www.it-connect.fr/cyberattaque-supermarches-froid-danfoss/) (12 Oct), the attackers forced the controllers' defrost cycle on and kept it running with a 50 °C setpoint. Defrost is normally a short heating phase that clears ice from the coils. Left on, it turns a fridge into an oven.

Intermarché reported 150–200 affected stores to AFP, and Coopérative U reported fewer than 100. Leclerc, Carrefour Market and Monoprix stores were also mentioned. The reported target was older versions of Danfoss refrigeration software. [Tom's Hardware](https://tomshardware.com/tech-industry/cyber-security/unprecedented-cyberattack-cripples-supermarket-fridges-across-europe-as-hackers-activate-automatic-defrosting-delivery-and-store-prices-also-impacted-by-incident) cited early loss estimates near €300,000, potentially rising to several million euros nationwide. Two stores in Haguenau alone estimated about €270,000 between them. On the same night, Ahold Delhaize shut down systems at Albert Heijn in the Netherlands after a security incident ([NL Times](https://nltimes.nl/2026/10/12/cybersecurity-incident-hack-may-behind-outage-albert-heijn)). Nobody has said whether the two incidents are linked.

## The problem

What stood out to me was the timing, more than the hack itself. It happened at night, while the stores were closed. The system under attack was also the one reporting temperatures, so nothing told anyone that anything was wrong. Staff found out when they opened up and found warm display cases and a butchery counter that had to be emptied.

Food safety gives a hard deadline here. Refrigerated ready-to-eat food has to stay at or below 5 °C (41 °F). The usual retail guidance, including the [Conference for Food Protection's emergency plan](https://ehs.dph.ncdhhs.gov/faf/docs/enfmanstrat/EmergencyGuidanceforColdHolding.pdf) and the FDA Food Code's 4-hour time-as-a-control rule, says food that has been warmer than that for more than about four hours must be cooked or thrown away.

So the question that decides the losses is when someone finds out:

- **About 02:15:** there are four hours to switch off the defrost, move stock to working cases, or call a technician.
- **07:00:** nothing can be saved.

## What I built

[Cold Chain Sentinel](https://github.com/Anjali0777/cold-chain-sentinel) is a small Python tool that watches refrigeration from the outside. It uses two inputs:

1. **Independent probe temperatures**: cheap wireless sensors that don't depend on the refrigeration controller.
2. **The controller's event log**: when each defrost starts and stops, and every setpoint change.

From these it raises five kinds of alert:

- a setpoint changed out of range
- a defrost running too long
- too many defrosts in a day
- a temperature excursion
- the controller's reading disagreeing with the probe's

It then gives every case a decision: **OK**, **Act now** (with a deadline), or **Discard**. It totals how much stock is still saveable and how much is already lost, and it reports how many minutes before opening the first critical alert fired.

It comes as a CLI that writes a JSON and an HTML report, plus a FastAPI endpoint. A `--as-of` flag replays the data as it stood at any moment, which is how I tested it against the attack timeline.

To test it without real store data, I wrote a simulator for a 12-case store. Normal defrosts run four times a day. At 02:10 on 12 October, the butchery, fish counter and frozen-meat cases get a forced 50 °C defrost. I also made the fish counter's controller keep reporting normal temperatures during the attack. That tests the one alert that only works if your sensors are independent.

## How it works

Most of the detectors are built on one small function. It walks a case's readings in time order and returns each stretch where a condition held, such as "probe above 5 °C". It splits stretches where data is missing, so a dead sensor doesn't look like a four-hour excursion:

```python
def runs(ts, flag, step):
    out, start, last = [], None, None
    for t, f in zip(ts, flag):
        if f:
            if start is None:
                start = t
            elif t - last > 2 * step:   # a gap in the data closes the run
                out.append((start, last + step)); start = t
            last = t
        elif start is not None:
            out.append((start, last + step)); start = None
    if start is not None:
        out.append((start, last + step))
    return out
```

The rest is choosing thresholds that ignore normal operation. Normal defrosts push the air temperature above 5 °C for about 25 minutes several times a day. My first version marked every case as "act now" because of them. So the food-safety clock now ignores short bumps and stretches that already recovered. The excursion alert also waits 45 minutes before firing. On the simulated store, the eight cases with only normal defrosts raise no alerts at all. A separate stuck defrost timer the day before shows up as one medium-severity alert, which is the right level for it.

The freezer case taught me something too. A freezer warming from −22 °C takes a while to reach 5 °C, but its stock is thawing long before that. So freezers also trigger "act now" when they pass their own alert limit of −15 °C.

## Business impact

Here is the simulated store replayed at two moments:

| | At 03:15 | At 08:00 |
|---|---|---|
| Critical alerts | 6 | 9 |
| Stock still saveable | €25,900 | €0 |
| Stock to discard | €0 | €25,900 |

The first critical alert, the 50 °C setpoint, fires at 02:10. That is 290 minutes before a 07:00 opening, and every discard deadline falls between 06:16 and 06:46. Nothing changes between the two runs except when someone looks.

Scaling that to the real incident, with rough figures:

- The Haguenau stores averaged about €135,000 each.
- If an alert at 02:10 lets an on-call manager save even a third of that, it is about €45,000 per store.
- Across 150 Intermarché stores, that is roughly €6.75M.
- Independent wireless probes cost tens of euros per case.

The honest caveats:

- This only works if someone is on call and can actually do something at 2 a.m.
- The per-store losses are early press estimates.
- My test data is synthetic: the attack follows the reported pattern, but the temperature curves are my model, not measurements.

## What I'd do next

- **SMS/email alerting with escalation**, since a report nobody reads at 2 a.m. saves nothing.
- **An MQTT collector** for common wireless probe gateways, so it runs on live data.
- **Learned defrost profiles per case**, to cut the 45-minute grace period down to a few minutes.
- **A store-wide signal:** three cases getting the same setpoint in the same minute is much stronger evidence than any single reading.

The broader lesson I took from this: the system you're monitoring shouldn't be the only thing reporting on itself.

Built with AI pair-programming (Claude); I reviewed the design, tests and results.
