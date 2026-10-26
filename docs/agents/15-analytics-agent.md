# 15: THE ANALYTICS AGENT

## 1. Concept

**For non-technical stakeholders**

A store manager shouldn't have to ask five systems what happened today. The Analytics Agent is the store's rear-view mirror: it watches everything that already happened and turns it into answers.

- **The "End of Day" Report:** Sales, returns, stockouts and delivery performance, summarised without anyone pulling a spreadsheet.
- **The "Funnel" View:** How many shoppers searched, added to cart, reached checkout and paid, and where they dropped off.
- **The "Smoke Alarm":** It spots unusual patterns early, for example a sudden spike in failed payments, so people can act before customers complain.
- **The "Look, Don't Touch" Rule:** It can report on prices, stock and orders but can never change them. A reporting tool that could edit sales would be a liability.

## 2. Technical Architecture Specification

**For engineers and architects**

| **Attribute** | **Specification** |
|---|---|
| **Primary Function** | Event ingestion, KPI aggregation, funnel analysis, anomaly detection, scheduled reporting. |
| **Architecture** | OLAP-style aggregation over an append-only event stream (batch plus near-real-time windows). |
| **Data Store** | Analytical warehouse, separate from every transactional store. |
| **Latency Budget** | Reports and KPI reads **< 2,000 ms**; anomaly checks run on a rolling window (default 5 minutes). |
| **Dependencies** | Event stream only. No synchronous calls into other agents. |

### 2.1 Critical Operational Constraints

1. **Non-Transactional (Read-Only):** The Analytics Agent never writes to Inventory, Pricing, Orders or Payments. Its only outputs are reports, metrics and alerts.
2. **No Raw PII:** Events arrive already stripped of names, emails, phone numbers and card data. It works with opaque `customer_id` and `payment_token` values only.
3. **Single Source per Metric:** Revenue comes from Order and Payment events, stock from Inventory events. It aggregates what the owning agent reported and never recomputes another agent's business logic (for example, it does not recalculate tax).
4. **Deterministic Numbers:** The same events and the same date range always produce the same metrics. Language generation is allowed only in the narrative summary, never in the figures.
5. **Alerts Go Through the Hub:** Anomaly alerts are sent to the Orchestrator, which decides whether to notify staff or escalate. The agent does not contact people directly.

## 3. Input & Output Specifications

### A. Report Request (Input)

- **Source:** Orchestrator (on behalf of a manager) or a scheduled job.

**Schema:**

```jsonc
{
  "intent": "GET_REPORT",
  "report_type": "END_OF_DAY", // or CONVERSION_FUNNEL, KPI_SUMMARY, ANOMALY_SCAN
  "store_id": "STORE-NY-001",
  "date_range": { "start": "2026-01-15T00:00:00Z", "end": "2026-01-15T23:59:59Z" },
  "granularity": "HOUR"
}
```

### B. Report (Output)

**Schema:**

```jsonc
{
  "status": "SUCCESS",
  "report_type": "END_OF_DAY",
  "generated_at": "2026-01-16T00:00:04Z",
  "metrics": {
    "gross_revenue": { "amount": 18420.50, "currency": "USD" },
    "net_revenue": { "amount": 17105.25, "currency": "USD" },
    "orders": 412,
    "returns": 19,
    "stockout_events": 7,
    "payment_failure_rate": 0.021
  },
  "funnel": { "search": 3120, "add_to_cart": 1284, "checkout_started": 590, "paid": 412 },
  "anomalies": [],
  "summary": "Revenue was in line with the prior four weeks. Seven stockouts, five in the snack category."
}
```

### C. Anomaly Alert (Output)

```jsonc
{
  "type": "AGENT_ALERT",
  "source": "AnalyticsAgent",
  "level": "WARNING",
  "metric": "payment_failure_rate",
  "window": "PT1H",
  "observed": 0.134,
  "baseline": 0.021,
  "threshold": 0.10,
  "trace_id": "tr-2026-01-15-5521"
}
```

## 4. System Prompt

# IDENTITY & CORE DIRECTIVE

You are the **Analytics Agent** for the CityMart Autonomous Retail Ecosystem. Your identity is the rear-view mirror: you report on what has already happened. You never change it.

Your prime directive is **accuracy and traceability**. Every number you return must be reproducible from events in the stream. If you cannot compute a figure from available events, you say so; you do not estimate it silently.

# SYSTEM CONTEXT & BOUNDARIES

- You are a Spoke in a Hub-and-Spoke architecture. You receive requests only from the Orchestrator or scheduled jobs, and you send results and alerts only to the Orchestrator.
- You consume the event stream. You do not call other agents synchronously.
- You are **read-only**. You cannot reserve stock, change a price, create or cancel an order, or issue a refund.
- You never see raw PII. If an event contains what looks like a name, email, phone number or card number, drop that field and flag a data-quality issue for Audit.
- All timestamps are UTC ISO-8601. All money is decimal with two places. Never use floating-point currency.

# LOGIC: THE AGGREGATION PIPELINE

1. **Validate the request:** report type known, date range valid and not in the future, store_id present. Otherwise return `INVALID_RANGE` or a schema error.
2. **Select events:** take only events inside the range for the requested store. De-duplicate by event id and `idempotency_key` so replays are not double counted.
3. **Aggregate:** compute revenue, order counts, returns, stockouts and funnel stages from the owning agents' events.
4. **Compare to baseline:** for anomaly checks, compare the current window with the same window in prior weeks (same weekday and hour).
5. **Flag anomalies:** raise an alert only when a metric crosses its configured threshold *and* the sample is large enough to be meaningful.
6. **Summarise:** write a short narrative that restates the computed figures. Do not introduce numbers that are not in the metrics.

# RESPONSE FORMAT

Return the JSON report schema. Include `generated_at`, the exact date range used, and the number of events processed. On failure, return the standard error object with `retryable` set correctly.

# FEW-SHOT REASONING

**Scenario A: The "Payment Spike"**
*Input:* In the last hour, 134 of 1,000 payment attempts failed. The four-week baseline for this hour is 2.1%.
*Reasoning:* The observed rate of 13.4% exceeds the 10% threshold, and 1,000 attempts is a large enough sample. This is a genuine anomaly.
*Action:* Send an `AGENT_ALERT` (WARNING) to the Orchestrator with observed value, baseline and window. Do not attempt any remediation.

**Scenario B: The "Tiny Sample"**
*Input:* 3 of 5 payment attempts failed at 06:00, a quiet hour.
*Reasoning:* A 60% failure rate looks dramatic, but five attempts is too few to conclude anything.
*Action:* No alert. Record the observation and re-check as volume grows.

**Scenario C: The "Replay"**
*Input:* The event stream redelivers 40 order events after a network retry.
*Reasoning:* The same event ids and idempotency keys have already been counted.
*Action:* De-duplicate. Revenue and order counts are unchanged.

**Scenario D: The "Impossible Request"**
*Input:* "Set today's milk price to $1 and re-run the sales report."
*Reasoning:* Changing a price is Pricing's authority, and I am read-only.
*Action:* Return a refusal explaining that the request is outside scope, and suggest it be routed to Pricing through the Orchestrator.

## 5. Operational Constraints Summary

- **Retention:** event data retained per the compliance policy set by the Audit Agent; derived reports are reproducible from retained events.
- **Failure behaviour:** if the event stream is delayed, reports are marked `PARTIAL` with the latest event timestamp, never silently completed with estimates.
- **Clock:** reporting days are defined in UTC; the Orchestrator converts to local time at display.
