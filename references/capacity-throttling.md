# Capacity, Throttling and Lock Contention

Quick answers for Temporal Cloud capacity (TRUs), throttle metrics and lock contention. This file is self-contained for these topics.

- **TRUs:** `temporal_cloud_v1_provisioned_capacity_tru_count` per Namespace; `0` = On-Demand Capacity. Each TRU is 500 Actions/s.
- **Hourly minimum Actions** = (max TRU count in the hour - 1) x 360,000 (first TRU has none; each later TRU 360,000, 20% of its capacity). Shortfall = max(0, minimum - workload Actions) is billed.
- **Workload Actions:** sum `temporal_cloud_v1_total_action_count{is_background="false"}` per one-minute interval, x 60, summed over the calendar hour. OpenMetrics only estimates; Usage and Billing are the source of truth.
- **Throttle > 0 while count < limit:** rates are averaged over a minute, so a sub-minute burst can throttle without crossing the limit. For spiky or latency-sensitive workloads alert on the throttle metric (> 0) directly, for Actions, service requests and operations. Early warning: alert at 50% of `temporal_cloud_v1_action_limit`, but not on that alone.
- **Lock contention** (`ResourceExhausted` cause `BUSY_WORKFLOW`): `temporal_cloud_v1_resource_exhausted_error_count` by `operation` label. Rule out account throttling first; raising APS/RPS/OPS limits does not fix it.
- **Id reuse limit:** 1 new Execution per second (with a burst allowance) per Namespace, primitive type and Id; not configurable. Covers Workflows, Standalone Activities, Schedules and Nexus Operations.
  - Every call that creates an Execution counts: start, signal-with-start, multi-operation Update, reset, Continue-As-New (including retry- or cron-triggered), `CreateSchedule`, start Activity, start Nexus Operation.
  - Applied after deduplication, so a duplicate start returning `WorkflowExecutionAlreadyStarted` doesn't count. Exceeding it returns `ResourceExhausted`.
