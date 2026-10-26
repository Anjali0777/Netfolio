---
title: Hotspot Ring
---

## What happened

On 8 October 2026, Google Cloud Storage in us-central1 had about 12 hours of slow and failing uploads and writes, plus some failed reads. Google's preliminary report, summarised by [IncidentHub](https://blog.incidenthub.cloud/2026-10-12-google-cloud-storage-outage-oct-8-2026), says a traffic-sorting optimization meant to ease local capacity constraints caused severe resource exhaustion on *specific backend storage nodes*. Downstream services felt it too; [Temporal's status page](https://status.temporal.io/) was one of several that cited the incident.

I don't know how Google routes storage traffic, and this isn't a guess at it. The phrase "specific backend nodes" made me want to work on the general question: when traffic is skewed, how do you route requests so it can't pile up on a few machines?

## The engineering problem

A router sends each request for an object to one of N backend nodes. It needs three properties, and they pull against each other:

1. **Balance.** No node carries much more than its share of live requests.
2. **Affinity.** Requests for the same object keep hitting the same node, so its cache stays warm.
3. **Stability.** When a node leaves, only about 1/N of the traffic moves, instead of every cache going cold at once.

Real traffic isn't uniform. Object popularity is roughly Zipf-shaped, and every so often one object takes a big share of all requests: a viral file, a hot index, a client stuck in a retry loop. The router has to stay balanced through that, not just on average.

## First attempt

I built the two textbook answers.

**hash(key) mod N** gives perfect affinity and is as cheap as routing gets. But when one node of 50 leaves, it moves **98.8%** of traffic, so every cache goes cold together.

**A consistent-hash ring** with 160 virtual nodes per server fixes that: removing a node moves **1.4%** of traffic, and a property test confirms only that node's keys move.

Then I benchmarked both on Zipf 1.1 traffic. The setup was 50 nodes, 10,000 requests in flight, a million steps, and the busiest node's load checked after every step. On the ring, the busiest node ran at **8.3× its fair share**; mod-N was 7.8×. Both routers send every request for a given key to one node, regardless of how busy that node is. The skew in the traffic goes straight to whichever machine owns the hot keys.

One thing surprised me. Even with *uniform* keys, the ring's busiest node ran at 1.43× the mean. With 160 virtual nodes, each server's share of the key space varies by around ±1/√160. That's well known in theory, but it was different to see it in my own numbers.

## The better design

The fix comes from a 2018 paper by Mirrokni, Thorup and Zadimoghaddam, *Consistent Hashing with Bounded Loads*. Keep the ring, and add one rule: no node may carry more than

```
cap = ceil((1 + ε) · (in-flight + 1) / N)
```

live requests. A request still starts at its key's home position on the ring. If that node is at the cap, the request walks clockwise to the next distinct node that isn't. Here is how that solves each of the three problems:

- **Most keys never notice.** Their home node isn't full, so they behave exactly like the plain ring. A test confirms they get identical results when nothing is near capacity.
- **Hot keys overflow onto their ring neighbours,** which are the same neighbours every time. Their spillover lands on a small, stable set of nodes instead of being sprayed everywhere, so even the overflow keeps some cache warmth.
- **Peak load is bounded by construction,** at ⌈(1+ε)·mean⌉, whatever the key distribution.

The risk with a load-dependent router is a bookkeeping bug. A missed release or an off-by-one would mean the bound quietly stops holding. So the main test is a fast-check property test. It generates random sequences of acquires (with skewed keys), releases, and nodes joining and leaving, and checks after every single step that no node is over the cap. To check that the test has teeth, I changed `<` to `<=` in the cap comparison, and it fails.

I also built **power of two choices** as a comparison: hash each key to two candidate nodes and use the less busy one. It's a well-known way to balance, but for a hot key it only halves the problem, because the key is still tied to the same two nodes.

## Results

Here are the Zipf 1.1 results, with 50 nodes and 200 requests in flight per node on average:

| Router | Busiest node vs mean | Affinity | Traffic moved when 1 node leaves |
|---|---:|---:|---:|
| mod-N | 7.81× | 100% | 98.8% |
| Ring | 8.30× | 100% | 1.4% |
| Two choices | 3.66× | 72% | 98.4% |
| Bounded ring ε = 1.0 | 2.00× | 82% | 1.4% |
| **Bounded ring ε = 0.25** | **1.25×** | 65% | 1.4% |
| Bounded ring ε = 0.1 | 1.10× | 57% | 1.4% |

With one extra object taking 10% of all traffic on top of the Zipf mix, the ring's busiest node was at 7.54×. The bounded ring stayed at exactly 1.25×.

A few things stand out:

- **The bound is hit exactly.** Under skew, the busiest node sits at the cap, which is what the theory promises.
- **The walk is short.** At ε = 0.25, a request skips about one ring position on average, even under heavy skew.
- **Cost is about the same.** In my harness, routing costs 1.3–1.4 µs per acquire and release for both the ring and the bounded ring. Most of that is the hash itself: turning a JavaScript string into bytes.
- **Uniform keys improve too.** The bound fixes the ring's virtual-node imbalance, from 1.43× down to 1.25×, while keeping 99.8% affinity.

## Tradeoffs and what I'd do at scale

The cost is affinity under skew. At ε = 0.25, about 35% of requests land on a different node than the key's previous request did. Nearly all of those are the hot keys, which is the point: they have to spread to stay under the cap. ε is the dial. At 1.0 you keep 82% affinity and allow 2× peak load. At 0.1 you get 1.1× peak with colder caches. I'd start at 0.25 and tune it against real cache hit rates, not my proxy metric.

Other honest limits:

- **The bound counts requests, not cost.** A 5 GB read counts the same as a 1 KB one. The fix is to charge each request a weight; the algorithm doesn't change.
- **It needs to see completions.** A stateless client-side hash can't do this. With several router instances, each enforces the bound on its own view of load, so the true per-node load can drift above the cap. You'd either give each router a tighter local ε or gossip approximate loads.
- **At much larger scale,** I'd route in two levels, zone then node, so a hotspot spills within a zone first. I'd also pair the bound with hot-key replication: the bound protects the machines, but a truly viral object should also be cached closer to users.

## What I'd do next

- **Weighted bounded loads,** so each request is charged by its expected cost.
- **Multiple routers with gossiped load,** measuring how far peak load drifts as gossip gets staler.
- **A count-min sketch to detect hot keys,** so they can be replicated before they ever hit the cap.

The code, benchmark harness, demo and a design doc covering the rejected alternatives are in the [repo](https://github.com/Anjali0777/hotspot-ring).

Built with AI pair-programming (Claude); I reviewed the design, tests and results.
