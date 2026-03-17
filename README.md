# warehouse-ops-intelligence

> BI system design and SQL framework for B2B wholesale fulfillment operations — built at ShipMonk across 10+ warehouse sites.

---

## Overview

This repo documents the data infrastructure and BI design behind ShipMonk's B2B wholesale operations scoreboard — a weekly reporting system used by ops directors and site leads to manage fulfillment performance across a multi-node warehouse network.

**The system answers four core operational questions, every week:**
1. Are orders shipping on time? *(OTRS — On-Time Release to Ship)*
2. Are orders shipping in full? *(OTS — On-Time & In-Full Shipment)*
3. Is pick accuracy holding? *(Pick Audit)*
4. Is pack accuracy holding? *(Pack Audit)*

---

## System Architecture

```
Snowflake (BDM schema)
    └── Raw fulfillment events, order lines, shipment records
            │
            ▼
Metabase (SQL Cards + Dashboards)
    └── Weekly aggregations by warehouse, W-1 vs W-2 comparison
            │
            ▼
Ops Meeting Automation (Python)
    └── Pulls Metabase card data via REST API
    └── Structures weekly agenda content
    └── Outputs to Google Docs for meeting prep
```

---

## Key Metrics Defined

### OTRS — On-Time Release to Ship
Measures whether wholesale orders were released to the warehouse for picking within the SLA window.

- **Denominator**: All wholesale orders due for release in the week
- **Numerator**: Orders released on or before the SLA date
- **Miss condition**: Release timestamp > SLA date

### OTS — On-Time & In-Full Shipment
Measures whether released orders actually shipped on time and completely.

- **Denominator**: All released wholesale orders due to ship in the week
- **Numerator**: Orders shipped on time with no short shipments
- **Miss conditions**: Ship timestamp > SLA date OR quantity shipped < quantity ordered

### Pick Audit
Random sample audit of picked units — measures picker accuracy before packing.

- **Score**: % of audited picks with zero errors
- **Tracked by**: Warehouse site, weekly rolling average

### Pack Audit
Random sample audit of packed cartons — measures pack station accuracy before shipment.

- **Score**: % of audited packs with zero errors
- **Tracked by**: Warehouse site, weekly rolling average

---

## SQL Patterns

### Weekly OTRS by Warehouse (sanitized)
```sql
SELECT
    warehouse_name,
    DATE_TRUNC('week', release_date) AS week_start,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN release_date <= sla_date THEN 1 ELSE 0 END) AS on_time_releases,
    ROUND(
        SUM(CASE WHEN release_date <= sla_date THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1
    ) AS otrs_pct
FROM wholesale_orders
WHERE
    release_date IS NOT NULL
    AND week_start >= DATEADD('week', -2, DATE_TRUNC('week', CURRENT_DATE))
GROUP BY 1, 2
ORDER BY 2 DESC, 1
```

### Late Orders Aging (sanitized)
```sql
SELECT
    warehouse_name,
    account_name,
    order_number,
    order_date,
    DATEDIFF('day', order_date, CURRENT_DATE) AS days_aged,
    ship_status
FROM wholesale_orders
WHERE
    ship_status != 'shipped'
    AND DATEDIFF('day', order_date, CURRENT_DATE) > 0
ORDER BY days_aged DESC
```

---

## Warehouse Network Coverage

The system covers the following ShipMonk warehouse sites:

| Region | Sites |
|---|---|
| West | Nevada, Nevada 2, Nevada 3, California, California 2 |
| East | Pennsylvania, Pennsylvania 2, New Jersey, New York |
| South | Texas, Texas Bonded, Florida, Florida 2 |
| Central | Kentucky, Kentucky 2, Kentucky Bonded |
| Canada | Toronto, Toronto 8 |
| International | Czechia, Mexico, Mexico 3, United Kingdom |

---

## Ops Meeting Automation

Weekly meeting prep is automated via a Python workflow that:

1. Calls Metabase REST API to pull W-1 and W-2 data from optimized SQL cards
2. Structures data into a standardized agenda format (SQCD, Scoreboard, Building Headlines, Risk)
3. Outputs formatted content to Google Docs for distribution before the weekly sync

See [`ai-ops-tooling`](https://github.com/arjkul/ai-ops-tooling) for the automation scripts.

---

## Impact

- Reduced meeting prep time from ~2 hours manual to ~5 minutes automated
- Enabled consistent W-1 vs W-2 trend visibility across all warehouse sites
- Surfaced aging order risk before it escalated to merchant escalations
- Became the operational source of truth for the B2B wholesale ops leadership team

---

## Related Repos

- [`b2b-chargeback-rca-framework`](https://github.com/arjkul/b2b-chargeback-rca-framework) — RCA process for EDI compliance violations
- [`metabase-sql-playbook`](https://github.com/arjkul/metabase-sql-playbook) — Full SQL library for warehouse ops metrics
- [`ai-ops-tooling`](https://github.com/arjkul/ai-ops-tooling) — Python automation for meeting prep and reporting
