---
title: "Building a Multi-Region API Platform at Scale"
description: "Designed and deployed a globally distributed API platform handling 500k+ daily requests with 120ms p99 latency and 99.9% uptime across 3 AWS regions."
date: 2025-04-10
tags: ["AWS", "Terraform", "Go", "Redis", "CloudFront", "Multi-Region"]
featured: true
metricHighlight: "120ms p99 latency"
problem: "SecurePKI, a SaaS platform serving enterprise clients in North America, Europe, and APAC, was running from a single US-East region. European customers experienced 400ms+ API latencies, violating SLAs. Two major outages in Q1 (totaling 4 hours of downtime) triggered contract penalty clauses and threatened client renewals worth ₹40L ARR."
architecture: "Deployed active-active API clusters in us-east-1, eu-west-1, and ap-southeast-1. Route 53 latency-based routing directs users to the nearest region. Each region runs an identical Go API stack behind an ALB, with a regional Redis cluster for caching and session state. PostgreSQL uses a primary-replica setup with cross-region async replication. Static assets served via CloudFront. All infrastructure managed with Terraform modules."
decisions:
  - "Chose active-active over active-passive - the latency SLAs required serving from the nearest region, not just failing over"
  - "Used async replication for PostgreSQL instead of synchronous - accepted eventual consistency (sub-second lag) to avoid cross-region write latency penalties"
  - "Implemented a conflict resolution strategy using last-writer-wins with vector clocks for the small subset of data that could be written from multiple regions"
  - "Chose Terraform over Pulumi - the ops team had existing Terraform expertise and the module ecosystem was more mature"
  - "Deployed Redis Cluster per region rather than a global cache - network hops to a central cache would negate the latency gains"
results:
  - "p99 API latency reduced from 400ms to 120ms for all regions"
  - "Achieved 99.9% uptime over 12 months (vs. 99.5% prior year)"
  - "Zero SLA penalty events since deployment"
  - "Retained all at-risk enterprise contracts (₹40L ARR preserved)"
  - "Infrastructure cost increased only 15% despite tripling regional presence - optimized with spot instances and reserved capacity"
lessons:
  - "Multi-region is not just infrastructure - application code must be region-aware for data locality and conflict handling"
  - "Async replication lag is manageable but must be visible - we built a real-time lag monitoring dashboard that alerts at 500ms"
  - "Terraform state management across regions requires careful design - we used a dedicated S3 backend per region with a global state orchestrator"
  - "Load testing must simulate realistic geographic distribution, not just throughput - latency profiles differ dramatically by region"
---

## Infrastructure Layout

<div class="mermaid">
graph TD
  Route53["Route 53 Latency Routing"] --> US
  Route53 --> EU
  Route53 --> AP

  subgraph US["us-east-1"]
    ALB1["ALB"] --> API1["API"]
    API1 --> Redis1[("Redis")]
    API1 --> PG1[("PostgreSQL Primary")]
  end

  subgraph EU["eu-west-1"]
    ALB2["ALB"] --> API2["API"]
    API2 --> Redis2[("Redis")]
    API2 --> PG2[("PostgreSQL Replica")]
  end

  subgraph AP["ap-southeast-1"]
    ALB3["ALB"] --> API3["API"]
    API3 --> Redis3[("Redis")]
    API3 --> PG3[("PostgreSQL Replica")]
  end

  PG1 -.->|Async Replication| PG2
  PG1 -.->|Async Replication| PG3
</div>
