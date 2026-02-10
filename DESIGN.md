# LitmusChaos Observability: My Thoughts on Adding Prometheus Metrics

**Author:** Akhil Mukkara  
**Date:** February 8, 2026

---

## Why I Care About This

I've been using LitmusChaos locally and trying to understand how everything works. One thing that's been frustrating is that I can't really see what's happening. Like, when I run experiments:

- Are they actually running right now?
- Did that last one succeed or fail?
- Why is this taking so long?
- Is something broken in the control plane?

I have to dig through logs or check the UI constantly, and even then it's not always clear.

So I decided to learn about observability. I built a demo with Prometheus and Grafana (https://github.com/akhilmukkara/prometheus-grafana-observability-demo) to understand how this stuff works. It was really helpful - now I can actually SEE what's happening in my demo app.

I think LitmusChaos needs something similar. This doc is my attempt at thinking through what that would look like.

---

## What I Learned from Building My Demo

I'll be honest - before building the demo, I didn't really understand Prometheus. But after working through it, here's what I picked up:

**Counters** - These just count things. Every time something happens, add 1. In my demo, I used this to count total experiments. You can't decrease a counter, it only goes up.

**Gauges** - These show current state. They can go up or down. I used this for "how many experiments are running RIGHT NOW." So it goes 0 → 2 → 3 → 1 → 0 as experiments start and stop.

**Histograms** - These track how long things take. This was the trickiest one to understand. Basically Prometheus puts durations into "buckets" and then you can ask questions like "how many experiments took between 1-5 seconds?" or "what's the 95th percentile?"

**The basic flow is:**
1. Your app exposes a `/metrics` page with plain text
2. Prometheus visits that page every 15 seconds and saves the numbers
3. Grafana queries Prometheus and makes graphs
4. You can see what's happening in real-time

Pretty straightforward once you understand it, but it took me a while to wrap my head around the histogram buckets.

---

## What's Already Being Done

I looked at PR #791 which is adding metrics for experiments. From what I can tell, they're focused on the experiment execution side - tracking when experiments run and whether they succeed or fail.

That's good and definitely needed, but I think there's more we could add:

**What they're covering:**
- Experiment execution tracking

**What seems missing:**
- The control plane (GraphQL server, auth server) - no visibility into whether the API is working well
- The chaos operator - is it processing experiments quickly? Are there errors?
- Dashboards - the metrics exist but how do users actually see them?

My idea is to extend this to cover the whole stack, not just experiments.

---

## My Proposal: What I Think We Should Add

Based on what I learned from my demo and reading through the LitmusChaos code, here's what I'm thinking:

### 1. Control Plane Metrics

The GraphQL server and auth server are critical - if they're down or slow, nothing works. We should track:

- How many API requests are happening
- How many are failing
- How long they take
- Which operations are being used most
- Authentication success/failure

I'm not 100% sure on the exact metrics yet, but something like:
- Request counts (counter)
- Error counts (counter)
- Request duration (histogram)
- Active users (gauge)

### 2. Chaos Operator Metrics

The operator is constantly reconciling experiments. We should know:

- How often it's running
- How long reconciliation takes
- If there are errors
- If there's a backlog building up

Again, I'd need to look at the operator code more carefully to figure out exactly where to put the instrumentation, but the idea is to make the operator observable.

### 3. Better Experiment Metrics

Building on PR #791, I think we could add:

- More detailed lifecycle tracking (not just start/end, but all the states)
- Why experiments fail (what was the error?)
- How long it takes for targets to recover after chaos
- Resource usage during experiments (does a CPU hog experiment actually use a lot of CPU?)

### 4. Dashboards

This is important - metrics are useless if people can't see them. I'm thinking two dashboards:

**Dashboard 1: LitmusChaos Health**
For people running the platform. Shows:
- Is the API working?
- Are there errors?
- Is everything healthy?

**Dashboard 2: Chaos Experiments**
For people using LitmusChaos. Shows:
- My experiment success rate
- Which experiments are slowest
- Current activity
- Recent failures

I'd base these on what I built in my demo, but adapted for LitmusChaos.

---

## How I'd Approach Building This

I've never done something at this scale before, so I'm thinking I'd need to break it down:

### Start Simple

I'd probably start with the GraphQL server since I've looked at that code a bit. Add basic metrics:
- Count requests
- Count errors  
- Track duration

Get that working, make sure Prometheus can scrape it, verify the data looks right.

### Then Expand

Once I understand the pattern, apply it to:
- Auth server
- Chaos operator
- Experiments (working with whatever PR #791 does)

### Finally Polish

- Build the Grafana dashboards
- Write documentation
- Test everything
- Get feedback from the community

I'd definitely need guidance on things like:
- Where exactly to put the metric calls in the code
- What labels to use (I know too many labels is bad, but how do you decide?)
- What histogram buckets make sense (my demo buckets are just guesses)
- How to avoid breaking anything

---

## Questions I Still Have

Being honest about what I don't know:

**Technical stuff:**
- How do you instrument a GraphQL server properly?
- What if the operator reconciles super frequently - will that create too much metric data?
- How do we track experiments that fail before they even start running?
- Should each experiment pod expose metrics, or should the operator aggregate them?

**Design decisions:**
- What metrics do users actually care about? (I should probably ask in the community)
- Do we need separate Prometheus instances for different components?
- How long should we keep the metrics? (storage costs vs. being able to debug old issues)

**Practical stuff:**
- How do we test this? Do I need a full Kubernetes cluster?
- What if my changes slow things down?
- How do I make sure the metrics are accurate?

These are all things I'd need to figure out during the mentorship.

---

## How We'd Know It's Working

**From a technical perspective:**
- All the components expose `/metrics` endpoints
- Prometheus scrapes them without errors
- The dashboards actually load and show data
- It doesn't slow down LitmusChaos

**From a user perspective:**
- People can answer their questions by looking at dashboards instead of digging through logs
- The community starts using the metrics
- We get feedback that it's helpful

**From a community perspective:**
- The dashboards are included in the Helm charts
- Documentation explains how to use the metrics
- Other contributors can add new metrics easily

---

## Why I Want to Do This

**What I bring:**
- I've actually built a Prometheus/Grafana demo, so I understand the basics
- I'm already contributing to LitmusChaos (I have a merged PRs and a few active ones)
- I've set up LitmusChaos locally and poked around the codebase
- I'm active in the community and responsive

**What I need to learn:**
- How to do this at production scale
- Best practices for instrumentation
- How to make good architecture decisions
- How to work with a mentor and take feedback

I'm not going to pretend I'm an expert. I'm a first-year student who's interested in observability and wants to learn by building something real. That's what mentorships are for, right?

---

## Links

- **My Demo:** https://github.com/akhilmukkara/prometheus-grafana-observability-demo
- **PR #791 (existing metrics work):** https://github.com/litmuschaos/litmus-go/pull/791
- **LFX Mentorship Issue:** https://github.com/litmuschaos/litmus/issues/5338
- **My Application:** https://github.com/litmuschaos/litmus/issues/5338#issuecomment-3721702000
- **My PRs:**
  - Merged: litmus-docs #425 (ARM64 MongoDB docs)
  - Active: litmus #5383, #5365, #4217

---

*I'm planning to update this based on feedback from the community.
If you have thoughts, let me know!*
