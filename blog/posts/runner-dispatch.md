---
title: Runner Dispatch
---

## What happened

On 5 October 2026, GitHub Actions spent about four hours struggling to assign hosted runners to jobs. According to [GitHub's incident report](https://www.githubstatus.com/incidents/3q1yb5m7ltvb), during the roughly 90-minute peak 71.9% of hosted jobs didn't start within five minutes. The cause was a partial network failure between one of GitHub's datacenter regions and cloud database and storage services. Recovery meant shifting traffic, adding capacity, and letting the queue drain. [UptimeRobot's timeline](https://uptimerobot.com/is-it-down/github/2026-10-05/) shows the same arc, ending with queued jobs clearing just before 22:00 UTC.

I don't know how GitHub's scheduler works internally, and this isn't a guess at it. What caught my attention was the last phase of the incident. When the root cause is fixed, the scheduler still has to work through a huge backlog while thousands of runners ask for work at the same moment. If picking a job gets more expensive as the queue grows, recovery slows down in proportion to how long the outage lasted. I wanted to build a scheduler that doesn't have that property and measure the difference.

## The engineering problem

A runner goes idle. Pick a job for it, subject to these rules:

- The runner must have **every label the job asks for**. A job with `runs-on: [ubuntu-latest, gpu]` can't go to a plain Ubuntu runner.
- The job's org must be **under its concurrency cap**.
- **No job runs twice and none are lost.**
- If any job could run there, the runner **must get one**. An idle runner next to an eligible job is wasted capacity during an incident.
- One org's burst **shouldn't push everyone else to the back**.

The hard part is that the queue is full of jobs that *can't* run right now. Some belong to an org already at its cap; others need a label set no idle runner has. A good scheduler shouldn't pay for those jobs on every call.

## First attempt

The obvious design is one FIFO list. When a runner goes idle, walk from the front and take the first job it can run.

It's correct. A property test with 300 random seeds checks it against a brute-force model, and it passes. It's also fast while the queue is short: about 14 µs per assignment at 1,000 queued jobs.

Then I benchmarked it on traffic shaped like a hosted CI service: 2,000 orgs with a long-tailed share of jobs, a realistic mix of runner labels, and a queue held at a fixed depth.

- **100,000 queued:** mean assignment time rises to **765 µs**.
- **One org bursting while at its cap:** **935 µs**, because every runner walks past that org's blocked jobs again.
- **Recovery:** with a 50,000-job backlog and 2,000 runners, the baseline hadn't finished after 60 seconds of pure scheduler time. It had assigned 53% of the jobs.

The worst case is a runner that finds nothing eligible. That happens often during recovery, and it always scans the entire queue. So the scheduler does the most work exactly when it is furthest behind.

## The better design

The fix is to never look at a job you can't hand out.

1. **Group jobs by the exact label set they need.** I call each group a pool. There are only a few dozen distinct label sets, even with millions of jobs.
2. **Inside each pool, keep one FIFO per org.**
3. **Each pool keeps a ring (a circular linked list) of only the org queues that are eligible right now:** non-empty, and the org under its cap.
4. **When an org hits its cap, unlink it from every ring it's in. I call this "parking".** When one of its jobs finishes, link it back in.
5. **A runner looks up which pools it can serve.** That list is cached per runner label set. It takes the oldest head among those pools, then moves that pool's ring on, so orgs inside a pool are served round-robin.

Labels are interned into a 64-bit mask, so "does this runner have every label?" is one AND:

```go
func (m Mask) Covers(need Mask) bool { return need&^m == 0 }
```

Here is how the cost of each operation changes:

| Operation | Naive | Indexed |
|---|---|---|
| Assign | O(Q) scan, plus an O(Q) slice delete | O(k), where k is the pools a runner covers (1–4 in practice) |
| Enqueue | O(1) | O(1) |
| Complete | O(1) | O(1), plus O(p) to unpark an org that uses p pools |

Neither k nor p grows with the backlog.

The main risk with a design like this is subtle bugs. For example, an org gets parked in one pool but not another, or a ring cursor ends up pointing at a removed node. I ran the same 300-seed property test against both implementations. It checks after every operation that:

- every assigned job fits its runner, and its org was under cap
- nothing is assigned twice or lost
- "nothing eligible" is only returned when that's true

As a sanity check that the test has teeth, I loosened the cap comparison by one character, and three tests failed. A concurrency stress test runs 12 goroutines through the mutex wrapper under `-race`.

## Results

These numbers come from a 2-vCPU cloud container with Go 1.24, on the same synthetic traffic for both designs. Mean `Assign` time:

| Queue depth | Normal mix: naive | Normal mix: indexed | Capped burst: naive | Capped burst: indexed |
|---:|---:|---:|---:|---:|
| 1,000 | 14.3 µs | 86 ns | 16.2 µs | 53 ns |
| 10,000 | 116.7 µs | 178 ns | 157.4 µs | 113 ns |
| 100,000 | 765.1 µs | 495 ns | 934.6 µs | 329 ns |

Naive grows linearly with the queue. Indexed stays under a microsecond, with p99 under 1 µs at every depth.

The recovery scenario measures scheduler time to drain a backlog with 2,000 runners, where 40% of the backlog belongs to an org at its cap:

| Backlog | Naive | Indexed |
|---:|---:|---:|
| 10,000 | 12.3 s | 24 ms |
| 50,000 | stopped at 60 s, 53% done | 132 ms |
| 200,000 | stopped at 60 s, 20% done | 556 ms |

In the most targeted case, 100,000 blocked jobs sit ahead of one small org's job. Naive takes 2.04 ms per assignment; indexed takes 161 ns, more than 12,000× less.

## Tradeoffs and what I'd do at scale

Round-robin across orgs isn't free. With a burst in the queue, the median wait for small, long-tail orgs falls by 11–51%, which was the goal. But the busiest ten orgs see their p99 wait rise by 4–28% at realistic depths, and nearly 2× at a short queue. Equal shares are a policy choice. In production I'd use weighted fair queueing, with weights by plan tier, and add aging so no job waits past a bound. I put this in the README rather than reporting only the speed numbers, because a design that hides its costs is hard to trust.

Other choices I made deliberately:

- **One mutex.** Critical sections are 50–500 ns, so the algorithm was the bottleneck, not lock contention. At 10× scale, I'd shard by label-set pool, since pools rarely overlap.
- **A 64-label limit per interner.** It keeps subset checks to a single instruction. Beyond 64, a bitset array costs a constant factor, not a different complexity.
- **In-memory state.** A real scheduler would rebuild from a durable job store. Replaying 200k jobs into this structure takes milliseconds.

At 100× scale, this becomes a distributed service:

- Each shard owns some pools, behind a replicated log.
- Org caps become leased tokens, so shards don't coordinate on every assignment.
- Runners long-poll the shard for their label set.

## What I'd do next

- Replace equal round-robin with deficit round-robin and per-tier weights.
- Add an HTTP long-poll front end, with runners as goroutines, to measure real time-to-start during a simulated outage instead of scheduler time alone.
- Benchmark one global lock against per-pool locks under contention.

The code, the benchmark harness and a design doc covering the alternatives I rejected are in the [repo](https://github.com/Anjali0777/runner-dispatch).

Built with AI pair-programming (Claude); I reviewed the design, tests and results.
