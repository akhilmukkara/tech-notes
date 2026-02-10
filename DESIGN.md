# LitmusChaos Observability: Adding Prometheus Metrics

**Author:** Akhil Mukkara  
**Date:** February 8, 2026

---

## Why This Matters

I've been running LitmusChaos locally and contributing to the project, and one thing that's been bugging me is that it's hard to know what's actually happening under the hood. When I run chaos experiments, I can't easily answer basic questions like:

- How many experiments are running right now?
- What's my success rate?
- Why did that experiment take so long?
- Is the control plane healthy?

I built a Prometheus + Grafana demo (https://github.com/akhilmukkara/prometheus-grafana-observability-demo) to learn how observability works, and I think LitmusChaos needs something similar. This doc outlines what I'd add and how I'd approach it.

---

## What I Learned Building My Demo

Before diving into LitmusChaos specifics, here's what I learned from building my observability demo:

**The Three Metric Types:**

**Counters** only go up - perfect for counting total events. In my demo, I used `chaos_experiments_total` to count all experiments. Every time an experiment runs, the counter increments. You can then use `rate()` in PromQL to see experiments per second.

**Gauges** go up and down - perfect for current state. I used `chaos_experiments_active` to show how many experiments are running right now. It goes: 0 → 3 → 5 → 2 → 0 as experiments start and stop.

**Histograms** track distributions - perfect for durations. I used `chaos_experiment_duration_seconds` to track how long experiments take. Prometheus automatically creates buckets, and you can calculate percentiles like "95% of experiments complete in under 10 seconds."

**How the whole thing works:**

Your Go app exposes a `/metrics` endpoint that returns plain text. Prometheus scrapes it every 15 seconds and stores the data. Grafana queries Prometheus and draws graphs. Simple, but powerful.

---

## Current State

### Existing Metrics Work

There is currently an open PR (#791) that adds Prometheus metrics for experiment execution. My proposal builds upon this foundation by extending observability to the entire LitmusChaos stack - control plane, operator, and adding user-facing dashboards.

**What exists/is in progress:**
- Experiment execution metrics (PR #791)

**What I'm proposing to add:**
- Control plane metrics (GraphQL API, Authentication)
- Chaos operator metrics (reconciliation, queue depth)
- Grafana dashboards with real-time visualization
- Comprehensive documentation and examples

The goal is complete observability across all LitmusChaos components, not just experiment execution.

---

## Components That Need Metrics

Based on my understanding of the LitmusChaos architecture, here are all the components that should be instrumented:

### 1. Control Plane (litmus-portal)

**GraphQL Server** - This is the main API that the frontend talks to. We need to know:
- Request rate (queries per second)
- Error rate (how many queries fail)
- Latency (how long queries take)
- Most-used operations (which queries are called most)

**Authentication Server** - Handles user login and auth. We need:
- Login attempts (total and by success/failure)
- Active sessions
- Token validation time
- Auth errors by type

**Frontend Server** - Serves the React app. Less critical, but useful to track:
- Page load times
- API call errors (from browser perspective)
- Active users

### 2. Chaos Infrastructure

**Chaos Operator** - The Kubernetes operator that reconciles experiment CRDs. We need:
- Reconciliation loops (how often it runs)
- Reconciliation duration (time to process an experiment)
- Reconciliation errors
- Experiment queue depth

**Chaos Experiments (litmus-go)** - The actual chaos injection code. Building on the existing PR #791 work, I'd add:
- Experiment lifecycle states (pending → running → completed)
- Injection success/failure tracking
- Target pod metrics (which pods are being chaos'd)
- Recovery metrics (did targets recover properly)
- Resource impact during chaos

**Chaos Runner** - Manages experiment execution. We need:
- Runner pod health
- Experiment assignments
- Execution errors

### 3. Database Layer

**MongoDB** - The backend database. We should track:
- Query duration
- Query types (read vs write)
- Connection pool usage
- Errors

We could use the MongoDB exporter for this rather than instrumenting it ourselves.

---

## Metrics Strategy

### Control Plane

The GraphQL and authentication servers would expose metrics for:
- Request volume and error rates
- API latency and performance
- Authentication success/failure patterns
- Active user sessions
- Most frequently used operations

### Chaos Operator

The operator would track:
- Reconciliation activity and frequency
- Processing time per reconciliation
- Queue depth and backlog
- Error rates by type
- Experiment processing throughput

### Chaos Experiments

Building on PR #791, we'd add metrics for:
- Complete experiment lifecycle tracking (all state transitions)
- Success/failure rates by experiment type
- Execution duration and performance
- Target recovery time and success
- Resource impact (CPU, memory) during chaos
- Injection failure reasons

The specific metric names, types, labels, and histogram buckets would be determined during implementation based on Prometheus best practices and community feedback.

---

## Architecture & Implementation

### Where Metrics Are Exposed

Each component exposes metrics on its own `/metrics` endpoint:
```
┌─────────────────┐
│  Prometheus     │  (Scraper - polls every 15s)
└────────┬────────┘
         │
    ┌────┴────────────────────┬─────────────────┐
    │                         │                 │
┌───▼─────────┐      ┌────────▼────┐    ┌──────▼──────┐
│ GraphQL     │      │ Chaos       │    │ Experiment  │
│ Server      │      │ Operator    │    │ Pods        │
│ :8080       │      │ :8080       │    │ :8080       │
│ /metrics    │      │ /metrics    │    │ /metrics    │
└─────────────┘      └─────────────┘    └─────────────┘
```

### Prometheus Scraping Config
```yaml
scrape_configs:
  # Control Plane
  - job_name: 'litmus-graphql'
    static_configs:
      - targets: ['graphql-server:8080']
    
  - job_name: 'litmus-auth'
    static_configs:
      - targets: ['auth-server:8080']
  
  # Operator
  - job_name: 'chaos-operator'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['litmus']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: chaos-operator
        action: keep
  
  # Experiment Pods (dynamic discovery)
  - job_name: 'chaos-experiments'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_chaosUID]
        action: keep
```

The experiment pods will be discovered automatically via Kubernetes service discovery, using the `chaosUID` label that already exists on chaos experiment pods.

---

## Implementation Approach

Based on my demo experience, here's how I'd approach the implementation:

### General Pattern

Each component would follow this pattern:
1. Add Prometheus client library dependency
2. Define metrics using appropriate types (Counter/Gauge/Histogram)
3. Instrument key code paths (API handlers, reconciliation loops, experiment lifecycle)
4. Expose `/metrics` endpoint
5. Configure Prometheus scraping

### GraphQL Server

The GraphQL server would be instrumented at the middleware layer to capture all operations automatically. Metrics would track request count, duration, and errors for each operation type. The `/metrics` endpoint would be added alongside the existing `/graphql` endpoint.

### Chaos Operator

The operator's reconciliation loop is the natural place to add metrics. Each reconciliation would increment counters and observe durations. Queue depth would be tracked as a gauge that increments when reconciliation starts and decrements when it completes.

### Chaos Experiments

Building on the existing PR #791 work, metrics would be added at key points in the experiment lifecycle - when experiments start, when chaos is injected, when targets recover, and when experiments complete. Resource impact would be measured by reading pod metrics during execution.

The specifics of where to place instrumentation calls and how to handle edge cases are things I'd figure out during the mentorship with guidance from my mentor.

---

## Implementation Timeline

I envision this work happening in three phases:

**Phase 1: Foundation**
Start with control plane metrics (GraphQL, Auth) and chaos operator. These are the core services that are always running, so getting observability here first provides immediate value and helps validate the approach.

**Phase 2: Experiment Coverage**
Build on PR #791's work to add comprehensive experiment tracking - lifecycle states, failures, recovery, resource impact. This is where users will see the most value.

**Phase 3: User Experience**
Create Grafana dashboards, alerting rules, and documentation. Make the metrics actually usable for the community.

The specific timeline and sequencing would be determined with my mentor based on priorities and what makes sense technically.
---

## Grafana Dashboards

Here's an example from my demo of what the dashboards would look like:

![Grafana Dashboard Example](images/grafana-dashboard.png)

*Example dashboard from my demo showing real-time chaos metrics visualization*

---

I'm picturing two main dashboards for LitmusChaos:

### Dashboard 1: LitmusChaos Control Plane

**Purpose:** Monitor the health and performance of the LitmusChaos platform itself

**What it would show:**

**API Health Row:**
- Requests per second over time
- Error rate percentage
- P95 latency
- Alert if error rate > 5% or latency > 500ms

**System Health Row:**
- Active logged-in users
- Authentication success rate
- Operator reconciliation rate
- Alert if auth success < 95%

**Performance Insights Row:**
- Top 10 slowest API operations
- Most frequently used operations
- Operator queue depth over time
- Alert if queue grows continuously

This gives platform operators visibility into whether LitmusChaos itself is healthy.

### Dashboard 2: Chaos Experiments Overview

**Purpose:** Monitor chaos experiment execution and results

**What it would show:**

**Experiment Activity Row:**
- Currently active experiments (big number)
- Experiments per hour trend
- Overall success rate
- Alert if success rate < 90%

**Performance by Type Row:**
- P95 duration by experiment type (which types are slowest?)
- Failure rate heatmap by experiment type
- Identify problematic experiment patterns

**Detailed Debugging Row:**
- Recent experiments table (last 50 with type, duration, status)
- State transition tracking over time
- Target recovery duration
- Resource impact graphs

**Resource Impact Row:**
- CPU usage during experiments by type
- Memory usage during experiments by type
- Understand the cost of chaos

This gives chaos engineers visibility into their experiment results and helps debug failures.

---

## What I Still Need to Learn

I'm being honest here - I understand the basics from building my demo, but there's stuff I'll need help with during the mentorship:

**Architecture questions:**
- Where exactly should operator metrics be recorded? In the reconciliation loop? During CR validation? Both?
- How do we handle high-cardinality issues? For example, if someone runs 10,000 experiments with unique names, we can't use experiment name as a label
- Should we use PodMonitor for experiment pods or ServiceMonitor? What's the right service discovery pattern?

**Technical questions:**
- What's the best way to pass experiment metadata to metrics without creating cardinality issues?
- How do we handle metrics for experiments that fail before they even start running?
- Should the chaos-exporter be instrumented separately, or is that out of scope?
- How do we track metrics across experiment retries?

**Production questions:**
- What histogram buckets make sense for real LitmusChaos usage? My demo buckets might not match production workloads
- How long should we retain metrics? Storage costs vs. debugging needs trade-off
- Do we need separate Prometheus instances for control plane vs. experiments to isolate resource usage?
- What are realistic SLOs for the LitmusChaos control plane?

**Community questions:**
- How do we make this work well for both self-hosted and SaaS deployments?
- Should dashboards be part of the Helm chart or separate?
- What metrics do users actually care about most? (We should validate with community)

I'm comfortable figuring these out with guidance, but I don't want to pretend I have all the answers. That's what mentorship is for.

---

## Success Criteria

How do we know this project is done and working well?

**Technical Success:**
- All major components expose `/metrics` endpoints
- Prometheus scrapes all targets without errors
- Dashboards load in < 2 seconds with reasonable data volumes
- Metrics accurately reflect system state (validated through testing)
- Performance overhead is minimal (< 1% CPU/memory impact from instrumentation)
- No label cardinality explosions in production

**User Success:**
- Users can answer "how is my chaos engineering going?" by looking at dashboards
- Common debugging questions are answerable via metrics ("why did this fail?", "why was this slow?")
- SREs running LitmusChaos can set up meaningful alerts
- The dashboards are actually used (track this via Grafana analytics)

**Community Success:**
- Documentation is clear enough that contributors can add new metrics
- Dashboards are included in the official Helm charts
- Metrics become part of LitmusChaos best practices documentation
- Community feedback is positive (discussed in community calls, GitHub discussions)
- Other chaos tools reference our observability approach as an example

---

## Why I'm Prepared for This

**What I bring:**
- I'm already a LitmusChaos contributor with merged PRs.
- I built a working Prometheus/Grafana demo from scratch and understand the observability stack
- I've set up LitmusChaos locally, run experiments, and understand the architecture (control plane, operator, experiment execution flow)
- I know the codebase - I've read through litmus-portal, chaos-operator, and litmus-go code
- I'm active in the community and responsive to feedback
- I understand both the technical side (how to instrument code) and the user side (what people need to monitor)

**What I need from mentorship:**
- Guidance on architectural decisions and trade-offs
- Code review and production best practices
- Help with edge cases and corner cases I haven't thought of
- Experience making this production-grade and scalable
- Feedback on what metrics matter most to real users

I'm not claiming to be an expert. I'm a first-year student who's done enough hands-on work to understand the fundamentals, and I learn quickly when given good guidance. That's exactly what a mentorship is for.

---

## References & Links

- **My Observability Demo:** https://github.com/akhilmukkara/prometheus-grafana-observability-demo
- **PR #791 (Existing metrics work):** https://github.com/litmuschaos/litmus-go/pull/791
- **LFX Mentorship Issue:** https://github.com/litmuschaos/litmus/issues/5338
- **My Application Comment:** https://github.com/litmuschaos/litmus/issues/5338#issuecomment-3721702000
- **Prometheus Best Practices:** https://prometheus.io/docs/practices/naming/
- **Prometheus Client Library (Go):** https://github.com/prometheus/client_golang
- **LitmusChaos Documentation:** https://docs.litmuschaos.io

---

*This is a living document. I'll update it based on community feedback and as I learn more during the mentorship preparation period.*
