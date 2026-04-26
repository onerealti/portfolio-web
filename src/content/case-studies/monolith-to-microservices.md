---
title: "Migrating a Monolith to Event-Driven Microservices"
description: "Decomposed a 500k-line Rails monolith into event-driven services, reducing deploy times from 45 minutes to 18 minutes and eliminating cascading failures."
date: 2025-09-15
tags: ["Kafka", "Kubernetes", "Event Sourcing", "PostgreSQL", "Go"]
featured: true
metricHighlight: "60% faster deploys"
problem: "Zulfikey Migrations, a Series B fintech startup, had a 500k-line Rails monolith serving 500k daily requests. Deploy times exceeded 45 minutes, a single database handled all domains, and any service failure cascaded across the entire application. The team was shipping fewer than 3 features per sprint due to merge conflicts and test instability."
architecture: "Identified four bounded contexts (Payments, Identity, Ledger, Notifications) and extracted them into independent Go services communicating via Apache Kafka. Each service owns its own PostgreSQL database. An API gateway handles routing, authentication, and rate limiting. Legacy Rails app was kept as a thin orchestration layer during the 6-month migration window."
decisions:
  - "Chose Kafka over RabbitMQ for durable event log - the team needed replay capability for financial audit trails"
  - "Used the Strangler Fig pattern to migrate incrementally without a risky big-bang cutover"
  - "Kept PostgreSQL instead of moving to a NoSQL store - the data was inherently relational and the team had deep Postgres expertise"
  - "Deployed on Kubernetes with Helm charts for consistent environments across dev, staging, and production"
  - "Implemented distributed tracing with OpenTelemetry from day one to maintain observability across service boundaries"
results:
  - "Deploy time reduced from 45 minutes to 18 minutes (60% improvement)"
  - "Zero cascading failures in 6 months post-migration (vs. 12 incidents in the prior 6 months)"
  - "Sprint velocity increased 25% - teams could deploy independently"
  - "p99 API latency dropped from 800ms to 250ms due to service-level caching"
  - "Database CPU utilization dropped 40% after splitting into domain-specific databases"
lessons:
  - "Event schema versioning is hard - invest in a schema registry early, not after your first breaking change"
  - "The Strangler Fig pattern works, but requires strict discipline about not adding features to the monolith during migration"
  - "Distributed tracing isn't optional in microservices - without it, debugging cross-service issues is nearly impossible"
  - "The team needed 3 months of upskilling on Kafka and Go before they were productive - factor training into migration timelines"
---

## Architecture Diagram

<div class="mermaid">
graph TD
  Kong["API Gateway <br/> Kong"] --> Payments["Payments <br/> Go"]
  Kong --> Identity["Identity <br/> Go"]
  Kong --> Ledger["Ledger <br/> Go"]
  Kong --> Notifications["Notifications <br/> Go"]
  
  Payments --> Kafka[("Kafka Cluster")]
  Identity --> Kafka
  Ledger --> Kafka
  Notifications --> Kafka
</div>

## Migration Timeline

The migration was executed in three phases over 6 months:

1. **Phase 1 (Months 1-2):** Extracted Identity and Notifications - lowest-risk, fewest dependencies
2. **Phase 2 (Months 3-4):** Extracted Ledger - required careful data migration and dual-write period
3. **Phase 3 (Months 5-6):** Extracted Payments - the most critical path, with extensive canary testing
