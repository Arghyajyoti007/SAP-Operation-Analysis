# SAP GRC & IAM Operational Analytics (Microsoft Fabric & Power BI)

An end-to-end operational analytics solution built on **Microsoft
Fabric** and **Power BI** to monitor SAP GRC / Identity & Access
Management (IAM) service delivery, SLA compliance, lifecycle
bottlenecks, and team workload capacity.

> **Note:** All datasets used in this repository are synthetically
> generated to replicate realistic enterprise SAP GRC and ServiceNow
> operational workflows without exposing proprietary organizational
> data.

------------------------------------------------------------------------

## Table of Contents

-   [Business Problem Statement](#business-problem-statement)
-   [Architecture & Data Pipeline](#architecture--data-pipeline)
-   [Data Model (Star Schema)](#data-model-star-schema)
-   [DAX Measure Architecture](#dax-measure-architecture)
-   [Dashboard Overview (4-Page
    Report)](#dashboard-overview-4-page-report)
-   [Key Operational Findings](#key-operational-findings)
-   [Repository Structure](#repository-structure)
-   [Tech Stack](#tech-stack)

------------------------------------------------------------------------

## Business Problem Statement

In enterprise SAP GRC/IAM operations, support teams process thousands of
access requests monthly across critical landscapes (`PRD-ECC`,
`S4H-CORE`, `GRC-AC`, `BW-PRD`, `QAS-ECC`). Relying on flat ticket
exports creates three operational blind spots:

1.  **Throughput vs. Intake Blindness:** Inability to separate creation
    dates from closure dates, obscuring whether the team is burning down
    or accumulating backlog against the **1,500 tasks/month** target.
2.  **Hidden SLA Bottlenecks:** Knowing an SLA breached does not explain
    *why*---whether delays stemmed from initial queue triage, GRC
    risk-owner approvals, user clarification holds, or hands-on SAP
    provisioning (`SU01` / `PFCG`).
3.  **Unbalanced Queue & Aging Risk:** Lack of visibility into open
    ticket aging buckets and workload distribution between the Team Lead
    and Operations Analysts.

------------------------------------------------------------------------

## Architecture & Data Pipeline

The pipeline follows a **Medallion Architecture** inside a Microsoft
Fabric Lakehouse, feeding a **Direct Lake** semantic model.

``` text
[Raw Ticket Logs]
       |
       v
[Bronze Layer]
       |
       v
[Silver Layer (PySpark)]
       |
       v
[Gold Star Schema]
       |
       v
[Power BI Direct Lake Semantic Model]
       |
       v
[4-Page Operational Analytics Report]
```

### Bronze Layer --- Raw Ingestion

-   Ingests approximately **37,982 synthetic operational tickets**.
-   Preserves the raw ticket data for traceability and repeatable
    downstream processing.
-   Partitions ingested data by load date to support organized ingestion
    and processing.

### Silver Layer --- PySpark Transformation

-   Enforces the expected source schema and data types.
-   Performs deterministic deduplication using `Request_Id`.
-   Standardizes timestamps for consistent duration and date
    calculations.
-   Validates `SLA_Eligiblity` flags and excludes cancelled or
    transferred records from the eligible SLA population where
    appropriate.
-   Produces cleansed, consistent records for dimensional modeling.

> **Implementation note:** `SLA_Eligiblity` is retained here as
> supplied. If the actual source column is spelled `SLA_Eligibility`,
> use that exact name consistently in the notebook, Delta table, and
> semantic model.

### Gold Layer --- Dimensional Model

-   Publishes curated Delta tables in a dimensional Star Schema.
-   Structures the fact table at ticket-level grain.
-   Includes pre-calculated operational aging fields:
    `Open_Aging_Bucket` and `Open_Aging_Sort`.
-   Uses nullable `Int64` foreign keys to support missing or
    not-yet-assigned dimension relationships.
-   Optimizes the curated layer for consumption by the Power BI Direct
    Lake semantic model.

### Orchestration

A scheduled Microsoft Fabric Data Pipeline automates the end-to-end
sequence:

1.  Bronze ingestion.
2.  Silver cleaning, standardization, deduplication, and validation.
3.  Gold fact and dimension table generation.
4.  Direct Lake semantic model framing so the model can recognize the
    latest Delta table state.

------------------------------------------------------------------------

## Data Model (Star Schema)

The Gold layer is structured as a dimensional Star Schema centered on
`fact_operational_request`.

  ----------------------------------------------------------------------------
  Table Name                   Type                    Description
  ---------------------------- ----------------------- -----------------------
  `fact_operational_request`   Fact                    Ticket-level grain
                                                       containing request
                                                       timestamps, stage
                                                       durations, SLA flags,
                                                       and open-aging buckets.

  `dim_date_created`           Dimension               Role-playing date
                                                       dimension linked
                                                       through
                                                       `Created_Date_Key` for
                                                       intake tracking.

  `dim_date_closed`            Dimension               Role-playing date
                                                       dimension linked
                                                       through
                                                       `Closure_Date_Key` for
                                                       throughput and SLA
                                                       tracking.

  `dim_system`                 Dimension               SAP landscape
                                                       attributes, including
                                                       `PRD-ECC`, `S4H-CORE`,
                                                       `GRC-AC`, `BW-PRD`, and
                                                       `QAS-ECC`.

  `dim_business_unit`          Dimension               Requesting departments,
                                                       including Global
                                                       Finance, HR & Payroll,
                                                       Sales, Supply Chain,
                                                       and IT Ops.

  `dim_priority`               Dimension               Urgency tiers: P1 -
                                                       Critical, P2 - High,
                                                       P3 - Medium, P4 - Low,
                                                       and Standard SR.

  `dim_ticket_type`            Dimension               Request classification
                                                       (Incident vs. Service
                                                       Request) and subtypes
                                                       such as Role
                                                       Assignment, SAP ID
                                                       Creation, Firefighter /
                                                       EAM, and SU53.

  `dim_team_member`            Dimension               Analyst roster, role
                                                       titles, shift
                                                       assignments, and the
                                                       `is_lead` hierarchy
                                                       indicator.
  ----------------------------------------------------------------------------

### Modeling Notes

-   The fact table represents one operational request per record.
-   Created-date and closure-date dimensions support separate intake and
    throughput analysis.
-   SLA measures use an explicitly defined eligible closed-ticket
    population.
-   Open-ticket aging is evaluated independently of closure-date
    filtering.
-   The `is_lead` attribute enables role-based workload analysis between
    the Team Lead and Operations Analysts.

------------------------------------------------------------------------

## DAX Measure Architecture

All measures are organized inside a dedicated `_Measures` table and
grouped into five operational display folders.

### 01 --- Throughput & Workload

  ---------------------------------------------------------------------------
  Measure                             Purpose
  ----------------------------------- ---------------------------------------
  `Total Requests Ingested`           Gross volume of incoming tickets
                                      logged.

  `Closed Tasks Throughput`           Completed operational volume where
                                      `Request_Status = "Closed Complete"`.

  `Cancelled Tasks Count`             Non-actionable or duplicate requests
                                      excluded from SLA denominators.

  `Active Open Backlog`               In-flight requests (`Assigned`,
                                      `Work In Progress`, and
                                      `Pending User Info`), ignoring
                                      closed-date filters.

  `Monthly Target Variance`           Difference between closed-task
                                      throughput and the 1,500-task monthly
                                      benchmark.

  `Monthly Target Achievement %`      Percentage of the monthly closure
                                      target achieved.
  ---------------------------------------------------------------------------

### 02 --- SLA & Compliance

  -----------------------------------------------------------------------
  Measure                             Purpose
  ----------------------------------- -----------------------------------
  `SLA Eligible Closed Tasks`         Validated baseline denominator:
                                      closed-complete tickets with
                                      `SLA_Eligiblity = 1`.

  `SLA Met Tasks`                     Eligible closed tickets completed
                                      within the applicable SLA target.

  `SLA Breached Tasks`                Eligible closed tickets that
                                      exceeded the applicable SLA target.

  `SLA Compliance %`                  Core service-delivery KPI evaluated
                                      against a 96%--98% target.

  `SLA Compliance % MoM Delta`        Month-over-month SLA trajectory
                                      using explicit integer serial-month
                                      arithmetic
                                      (`Year * 12 + Month_Number`) for
                                      Direct Lake compatibility.

  `P1 Breach Rate %`                  Breach exposure isolated to P1 -
                                      Critical emergency requests.
  -----------------------------------------------------------------------

### 03 --- Cycle Time & Bottlenecks

  -----------------------------------------------------------------------
  Measure                             Purpose
  ----------------------------------- -----------------------------------
  `Avg Total Resolution Days`         End-to-end turnaround time from
                                      request creation to closure.

  `Avg Queue Wait Hours`              Average triage and dispatch
                                      latency.

  `Avg GRC Approval Wait Hours`       Average risk-owner / role-owner
                                      sign-off hold time.

  `Avg User Info Wait Hours`          Average time spent waiting for
                                      requester clarification.

  `Avg Fulfilment Hours`              Average hands-on technical
                                      execution time in SAP.

  `Avg Closure Lag Hours`             Average administrative wrap-up time
                                      after fulfilment.
  -----------------------------------------------------------------------

### 04 --- Backlog & Aging

  -----------------------------------------------------------------------
  Measure                             Purpose
  ----------------------------------- -----------------------------------
  `Current Open Age Days`             Average elapsed age of open
                                      tickets, anchored to the snapshot
                                      reference date using `AVERAGEX`.

  `Open Aging > 3 Days`               Early-warning indicator for stale
                                      queue items.

  `Open At-Risk SLA Breached`         Active open tickets where elapsed
                                      hours exceed `SLA_Target_Hours`.
  -----------------------------------------------------------------------

### 05 --- Team Performance

  -----------------------------------------------------------------------
  Measure                             Purpose
  ----------------------------------- -----------------------------------
  `Analyst Workload Share %`          Individual request-volume
                                      proportion using
                                      `ALLSELECTED('dim_team_member')`.

  `Team Lead Share %`                 Validates the Team Lead dispatch
                                      split (`is_lead = 1`) against total
                                      team volume using `REMOVEFILTERS`.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Dashboard Overview (4-Page Report)

### Page 1 --- Executive Operations Overview

-   Tracks Total Ingested Workload, Closed Throughput, Active Open
    Backlog, SLA Compliance %, and the month-over-month SLA shift.
-   Compares monthly intake and closure volume against the **1,500-task
    monthly benchmark**.
-   Monitors SLA performance against the **96%--98% target range**.
-   Breaks down request composition across five Business Units:
    -   Role Assignment: approximately 30%
    -   SAP ID Creation: approximately 20%
    -   Access Extension: approximately 16%
    -   User Unlock: approximately 12%
    -   SAP ID Deletion: approximately 10%
    -   SU53: approximately 7%
    -   Firefighter / EAM: approximately 3%

### Page 2 --- SLA Compliance & Stage Bottlenecks

-   **Dynamic Context Header:** DAX-driven title reflecting the active
    Year, Priority, and System selections.
-   **Breach Concentration Matrix:** Conditional-formatting heatmap
    crossing SAP Systems against Priority tiers, from P1 - Critical
    through Standard SR.
-   **Interactive Bookmark Toggle:** Switches between:
    -   **Stage Bottlenecks Chart:** Compares Queue Wait, GRC Approval,
        User Hold, Execution, and Closure durations in hours.
    -   **Root Cause Decomposition Tree:** Enables analysis through the
        hierarchy `System → Priority → Request Subtype → Business Unit`.

### Page 3 --- Team Capacity & Operational Backlog

-   **Analyst Scorecard Matrix:** Evaluates throughput, Workload Share
    %, individual SLA Compliance %, average resolution days, and active
    queue load across:
    -   Team Lead: Rajesh Sharma
    -   Operations Analysts: Amit Verma, Sneha Kulkarni, Vikas Patel,
        and Priya Nair
-   **Backlog Risk & Queue Distribution:** Monitors open-ticket aging
    buckets, including 8--15 Days and 16+ Days.
-   Breaks down the **336 in-flight tickets** across SAP landscapes and
    priority tiers.

### Page 4 --- Request Investigation & Audit Detail (Hidden Drill-Through)

-   Accessible through right-click drill-through from visuals across
    Pages 1--3.
-   Provides granular ticket-level audit details, including:
    -   `Request_Id`
    -   Request and closure timestamps
    -   Stage durations
    -   `SLA_Target_Hours`
    -   Color-coded `SLA_Status`
-   Includes context summary cards and a return-navigation button.

------------------------------------------------------------------------

## Key Operational Findings

### 1. P1 Triage Bottleneck

Overall 2025 SLA compliance reached **96.6%**, with **622 breaches
across 18,324 closed tasks**. However, P1 - Critical incidents
experienced a **79.31% breach rate**, accounting for **414 breaches**.
Stage decomposition showed that P1 tickets spent an average of **2.85
hours** waiting in the initial triage queue before assignment, consuming
more than 70% of their four-hour SLA window.

### 2. Standard SR Approval Delays

Standard Service Requests maintained **100.0% SLA compliance** against a
14-day SLA window but averaged **6.20 hours** in GRC Risk Approval
holds. This was materially higher than the negligible approval wait
times observed for urgent incidents.

### 3. Role-Aligned Capacity Split

Workload-share analysis showed that the Team Lead handled **7%** of
total requests, focusing on complex or critical escalations, with an
**86.8% SLA rate**. The four Operations Analysts each absorbed
approximately **23%--24%** of total volume, with SLA compliance ranging
from **97.2% to 97.6%**.

### 4. System Concentration

`PRD-ECC` (**280 breaches**) and `S4H-CORE` (**190 breaches**) accounted
for **75.5% of all breached tasks in 2025**. The leading contributors
included Firefighter / EAM Emergency Access and Critical Role Revocation
requests.

------------------------------------------------------------------------

## Repository Structure

``` text
SAP-Operation-Analysis/
├── notebooks/
│   ├── 01_bronze_to_silver_cleaning.ipynb
│   │   # PySpark cleaning, deduplication, and validation
│   └── 02_silver_to_gold_star_schema.ipynb
│       # Fact/dimension modeling and aging buckets
├── powerbi/
│   ├── SAP_GRC_Enterprise_Theme.json
│   │   # Custom JSON report theme
│   └── dax_measures_reference.md
│       # DAX measures organized across five display folders
├── screenshots/
│   ├── page1_executive_overview.png
│   ├── page2_sla_bottlenecks.png
│   ├── page3_team_capacity_backlog.png
│   └── page4_audit_drillthrough.png
└── README.md
```

------------------------------------------------------------------------

## Tech Stack

  -----------------------------------------------------------------------
  Area                                Technologies
  ----------------------------------- -----------------------------------
  Cloud & Data Engineering            Microsoft Fabric, Lakehouse, Delta
                                      Tables, PySpark Notebooks, Data
                                      Pipelines

  Querying & Data Modeling            Spark SQL, Star Schema Dimensional
                                      Modeling, Direct Lake

  Visualization & Business            Power BI Desktop, DAX, Bookmarks,
  Intelligence                        Drill-Through, Custom JSON Theming

  DAX Concepts                        Time Intelligence, Iterators,
                                      Filter Context, Context Transition,
                                      `ALLSELECTED`, `REMOVEFILTERS`
  -----------------------------------------------------------------------

------------------------------------------------------------------------

