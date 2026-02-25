# Tech Notes

Technical design proposals and documentation.

## LitmusChaos Observability Design

Design proposal for adding comprehensive Prometheus metrics to LitmusChaos as part of the LFX Mentorship program (March-May 2026).

**Read the full proposal:** [DESIGN.md](DESIGN.md)

### Proof-of-Concept Implementation

**[GraphQL Metrics PoC](proof-of-concept/graphql-metrics/)** - Working implementation showing how Prometheus metrics would be instrumented in the LitmusChaos GraphQL server.

Demonstrates:
- Metric instrumentation with Go + Prometheus client library
- Counter, Gauge, and Histogram usage
- Label design and `/metrics` endpoint

### Overview

This proposal covers:
- Complete metrics strategy for all LitmusChaos components (control plane, operator, experiments)
- Implementation architecture and approach
- Week-by-week implementation plan
- Grafana dashboard designs
- Success criteria

### Background

I'm Akhil Mukkara, a LitmusChaos contributor preparing for the "Add Prometheus Metrics to LitmusChaos Control Plane Service" mentorship.

To understand observability deeply, I built a working demo: https://github.com/akhilmukkara/prometheus-grafana-observability-demo

### My LitmusChaos Work

**Merged Contributions:**
- [litmus-docs #425](https://github.com/litmuschaos/litmus-docs/pull/425) - Fixed ARM64 MongoDB installation documentation

**Active PRs (Under Review):**
- [litmus #5383](https://github.com/litmuschaos/litmus/pull/5383) - Implemented Helm installation method in ChaosCenter UI
- [litmus #5365](https://github.com/litmuschaos/litmus/pull/5365) - Enhanced experiment status icons with warning indicators
- [litmus #4217](https://github.com/litmuschaos/litmus/pull/4217) - Refactored MongoDB dependency injection in control plane services

**Preparation Work:**
- [Observability Demo](https://github.com/akhilmukkara/prometheus-grafana-observability-demo) - Complete Prometheus + Grafana stack built from scratch
- [Application Comment](https://github.com/litmuschaos/litmus/issues/5338#issuecomment-3721702000) - Engagement on mentorship issue
- [BlogPost](https://medium.com/@akhil.mukkara/how-im-learning-observability-by-building-it-and-why-litmuschaos-needs-it-ac8e27b1f4da) - Covers my detailed approach.
- [BlogPost](https://medium.com/@akhil.mukkara/setting-up-end-to-end-observability-for-microservices-a25f58d42713) - Implementing the full observability stack with Online Boutique, Prometheus, and Grafana.
**Current Status:** Contributor with merged PR and active contributions, local dev environment set up, familiar with control plane architecture (GraphQL, Auth, MongoDB)

---

**Author:** Akhil Mukkara  
**Date:** February 8, 2026
