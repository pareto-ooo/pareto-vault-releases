---
id: sizing
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Infrastructure standards: Sizing and cost

How we decide how big an environment is and what it will cost. These are the ground truth rules for choosing service tiers and capacity for every workload, whatever services it uses. They follow the Cost Optimization and Performance Efficiency pillars of the Azure Well-Architected Framework and the FinOps Framework.

## Sizing philosophy

- **Just enough, proven by measurement.** The goal is the lowest cost at which the workload meets its agreed performance. A tier is chosen from the workload's needs and confirmed by a performance test, not picked by habit or for safety.
- **Sizing is decided per environment.** Development, test, acceptance and production have different goals and get different capacity. The same tier everywhere is a choice that needs a reason, not a default.
- **Rules, not price lists.** This standard names no service tiers, machine sizes or prices, because Azure changes them over time. Every decision uses the current Azure offering and prices at the moment it is made.
- **Requirements before load.** A required feature (private networking, deployment slots, an SLA, zone redundancy) can force a minimum tier. Feature requirements set the floor; load decides how far above it.

## Workload profile

Every workload records its profile in the project definition before any tier is chosen. The profile is the input for every sizing and cost decision.

- **Users.** The total number of users, how their use is spread over time, and the number of concurrent users at peak.
- **Usage pattern.** Office hours, around the clock, or batch, and any known peaks (month end, campaigns, seasonal).
- **Data.** Volume at go-live and expected growth per year.
- **Performance targets.** Measurable targets, such as response time at a percentile, throughput, or the window a batch must finish in.
- **Availability.** Derived from the workload's business criticality tag and its recovery time objective (see Availability by business criticality).

## Environment profiles

| Environment | Goal | Sizing |
|-------------|------|--------|
| Development and test | The application works functionally | The cheapest option that runs the application: free or entry tiers, consumption or serverless options that scale to zero or pause, no redundancy, short retention |
| Acceptance | A representative place to prove production behaviour | The same service tiers as production while a performance test runs; scaled down outside test windows |
| Production | The agreed performance at the lowest cost | The tier proven by the performance test; redundancy, backups and retention follow the business criticality |

- **Non-production never needs production availability.** Redundancy, high-availability options and long retention are production concerns, unless acceptance is used to test them.
- **Non-production runs on dev/test pricing.** Non-production environments live in dev/test subscriptions, which have lower licence costs.

## Availability by business criticality

Production availability follows the business criticality, as backups do (`compliance`). Each step up is a floor on the tier, paid for only by workloads that need it.

| Business criticality | Availability target per month | App Service | Azure SQL | Regions |
|----------------------|-------------------------------|-------------|-----------|---------|
| `NonCritical` | None agreed | One instance | Any tier; serverless with auto-pause allowed | One |
| `BusinessSupport` | 99.5% | One instance | Any tier; no auto-pause in production | One |
| `BusinessCritical` | 99.9% | At least two instances, zone redundant where the tier allows (`app-service`) | Zone redundant | One |
| `MissionCritical` | 99.95% | At least two instances, zone redundant | Zone redundant, with a failover group to the paired region | Two: the second region can take over, and the failover is tested every year |

- **Other services follow the same columns:** zone redundancy from `BusinessCritical` up, and a second region only for `MissionCritical`.
- **The project definition may set a stricter target** for a workload, with the tiers that meet it. A lower target than its criticality gives is a deviation (`deviations`).
- **The target covers normal operation.** A disaster such as the loss of a region is covered by the recovery time objective of `compliance` instead, which for `MissionCritical` is met by the failover to the second region.
- **The target is measured,** by the availability tests (`observability`), and reviewed with the cost after go-live.

## Choosing a tier

- **Start with the smallest tier that meets the requirements.** Move up only when a measurement shows the need.
- **Pay for use where usage is low or variable.** Consumption, serverless and autoscaling options are preferred over fixed capacity when the load is not constant.
- **Idle capacity stops.** Compute that is only used part of the time scales to zero, pauses or terminates when idle, in every environment where the usage pattern allows it.
- **A paused resource has a cold start, and the workload accepts it knowingly.** The first request after a pause waits while the resource resumes; for a serverless database with auto-pause that can take up to a minute, and the first connection may fail. The application's connection timeout and retry cover the resume, so the user sees a slow first request rather than an error. The workload profile records that the delay is accepted, and an environment whose performance targets cannot absorb it does not auto-pause.
- **Scale out for variable load, scale up for heavy single operations.** Horizontal scaling with a minimum and maximum is the default for load that varies.
- **Known peaks are planned, not provisioned all year.** Capacity is raised ahead of a known seasonal or business peak and lowered after it. Heavy processing that can run before the peak (such as precalculation) is moved out of it, so the peak only carries the user load.
- **Sizing values are per environment in configuration.** Tiers, instance counts and retention are inputs set per environment, never fixed inside a module, so each environment can be sized on its own.

## Cost estimate

- **Every workload has a cost estimate before its first deployment.** The estimate is per environment, per month, and is recorded in the project definition next to the workload profile.
- **The estimate is built from the workload profile and the chosen tiers,** using the Azure Pricing Calculator or the Azure Retail Prices API with the prices of that moment.
- **The estimate covers every cost driver,** not only compute: storage and its growth, data transfer, logging and monitoring ingestion, backups and licences.
- **The estimate is checked against the subscription budget** and refreshed whenever a tier changes.

## Performance test

- **Production sizing of critical workloads is proven before go-live.** For workloads tagged `BusinessCritical` or `MissionCritical`, a performance test on acceptance, with production tiers, against the targets in the workload profile decides the production tier.
- **The result is recorded** next to the cost estimate: the tier tested, the load applied, and whether the targets were met.

## After go-live

- **Actual cost and utilisation are reviewed regularly** against the estimate and the workload profile, using Azure Cost Management and Azure Advisor.
- **Consistently low utilisation means sizing down.** Headroom that is never used is cost without value.
- **Commitments come after measurement.** Reservations and savings plans are only taken for stable production load whose size has been confirmed by real usage.
- **A changed workload profile triggers a new sizing.** More users, new peaks or a new usage pattern mean the profile, the estimate and if needed the performance test are redone.
