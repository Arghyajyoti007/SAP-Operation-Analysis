# DAX (Data Analysis Expressions) Reference

All measures are stored inside the dedicated `_Measures` table and organized into five operational display folders. 

---

## Folder 01: Throughput & Workload

```dax
// Gross volume of incoming tickets
Total Request Ingested = 
COUNTROWS('fact_operational_request')

// Alias used across visuals and downstream measures
Total Task Created = 
COUNTROWS('fact_operational_request')

// Completed operational throughput
Closed Tasks Throughput = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] = "Closed Complete"
)

// Alias used on Page 1 & Page 3 visuals
Total Task Closed = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] = "Closed Complete"
)

// Non-actionable / duplicate requests
Cancelled Tasks Count = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] IN { "Cancelled", "Canceled" }
)

// In-flight queue (ignores closed-date filters so Page 1 card always displays live queue)
Active Open Backlog = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" },
    REMOVEFILTERS('dim_date_closed')
)

// Percentage of the 1,500 monthly team benchmark achieved
Monthly Target Achievement % = 
DIVIDE([Closed Tasks Throughput], 1500, 0)

// Volume variance against the 1,500 monthly team benchmark
Monthly Target Variance = 
[Closed Tasks Throughput] - 1500

// Net queue accumulation (+) or burn-down (-)
Net Queue Delta = 
[Total Request Ingested] - [Closed Tasks Throughput] - [Cancelled Tasks Count]
```

## Folder 02: SLA & Compliance
```dax
// Baseline denominator for SLA calculations
SLA Eligible Closed Tasks = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] = "Closed Complete",
    'fact_operational_request'[SLA_Eligiblity] = 1
)

// Updated to enforce Closed Complete & SLA_Eligiblity = 1 (prevents >100% SLA bug on Standard SRs)
SLA Met Tasks = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] = "Closed Complete",
    'fact_operational_request'[SLA_Eligiblity] = 1,
    'fact_operational_request'[SLA_Status] = "SLA Met"
)

// Updated to enforce Closed Complete & SLA_Eligiblity = 1
SLA Breached Tasks = 
CALCULATE(
    COUNTROWS('fact_operational_request'),
    'fact_operational_request'[Request_Status] = "Closed Complete",
    'fact_operational_request'[SLA_Eligiblity] = 1,
    'fact_operational_request'[SLA_Status] = "SLA Breached"
)

// Core SLA compliance KPI (Target: >= 96% - 98%)
SLA Compliance % = 
DIVIDE([SLA Met Tasks], [SLA Eligible Closed Tasks], 0)

// Overall SLA failure rate
Breach Rate % = 
DIVIDE([SLA Breached Tasks], [SLA Eligible Closed Tasks], 0)

// High-severity KPI card on Page 2 isolating P1 emergency access breach rate
Critical (P1) Breach Rate % = 
CALCULATE(
    [Breach Rate %],
    'dim_priority'[priority_name] = "P1 - Critical"
)

// Updated with explicit integer serial-month arithmetic for Direct Lake / DirectQuery compatibility
SLA Compliance % MoM Delta = 
VAR CurrentYear = MAX('dim_date_closed'[year])
VAR CurrentMonthNum = MAX('dim_date_closed'[month_number])
VAR CurrentSerialMonth = (CurrentYear * 12) + CurrentMonthNum
VAR PriorSerialMonth = CurrentSerialMonth - 1
VAR CurrentSLA = [SLA Compliance %]
VAR PriorMonthSLA = 
    CALCULATE(
        [SLA Compliance %],
        REMOVEFILTERS('dim_date_closed'),
        ('dim_date_closed'[year] * 12 + 'dim_date_closed'[month_number]) = PriorSerialMonth
    )
RETURN
    IF(
        NOT ISBLANK(CurrentSLA) && NOT ISBLANK(PriorMonthSLA),
        CurrentSLA - PriorMonthSLA
    )

// Numeric benchmark for card reference lines/labels
SLA Target Benchmark = 0.98
```

## Folder 03: Cycle Time & Bottlenecks
```dax
// End-to-end resolution turnaround in days (Closed Complete only)
Avg Total Resolution Days = 
CALCULATE(
    AVERAGE('fact_operational_request'[Total_Resolution_Days]),
    'fact_operational_request'[Request_Status] = "Closed Complete"
)

// Stage 1: Initial triage & dispatch wait (Hours)
Avg Queue Wait Hours = 
AVERAGE('fact_operational_request'[Queue_Wait_Hours])

// Stage 2: GRC risk / role-owner approval wait (Days & standardized Hours for Page 2 visual)
Avg GRC Approval Wait Days = 
AVERAGE('fact_operational_request'[Grc_Approval_Wait_Days])

Avg GRC Approval Wait Hours = 
AVERAGE('fact_operational_request'[Grc_Approval_Wait_Days]) * 24

// Stage 3: User clarification hold wait (Days, Conditional Delayed Days, & standardized Hours)
Avg User Info Wait Days = 
AVERAGE('fact_operational_request'[User_Info_Wait_Days])

Avg User Info Wait Days (When Delayed) = 
CALCULATE(
    AVERAGE('fact_operational_request'[User_Info_Wait_Days]),
    'fact_operational_request'[User_Info_Wait_Days] > 0
)

Avg User Info Wait Hours = 
AVERAGE('fact_operational_request'[User_Info_Wait_Days]) * 24

// Stage 4: Hands-on SAP technical provisioning execution (Hours)
Avg Fulfilment Hours = 
AVERAGE('fact_operational_request'[Fulfilment_Hours])

// Stage 5: Administrative closure wrap-up lag (Hours)
Avg Closure Lag Hours = 
AVERAGE('fact_operational_request'[Closure_Lag_Hours])

// Dynamic Page 2 Header Title reacting to Year, Priority, and System slicers
Page2_Dynamic_Bottleneck_Title = 
VAR SelectedYear = 
    IF(
        ISFILTERED('dim_date_closed'[year]),
        FORMAT(SELECTEDVALUE('dim_date_closed'[year]), "0000"),
        "All Years"
    )
VAR SelectedPriority = 
    IF(
        ISFILTERED('dim_priority'[priority_name]),
        SELECTEDVALUE('dim_priority'[priority_name]),
        "All Priorities"
    )
VAR SelectedSystem = 
    IF(
        ISFILTERED('dim_system'[system_name]),
        SELECTEDVALUE('dim_system'[system_name]),
        "All Systems"
    )
RETURN
"Operational Stage Bottleneck Analysis | " & SelectedYear & " | " & SelectedPriority & " (" & SelectedSystem & ")"
```

## Folder 04: Backlog & Aging
```dax
// Average age of currently open tickets anchored to dataset snapshot date
Current Open Age Days = 
-- Production alternative: VAR RefDate = TODAY()
VAR RefDate = DATE(2026, 1, 15)
RETURN
AVERAGEX(
    FILTER(
        'fact_operational_request',
        'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" }
    ),
    DATEDIFF('fact_operational_request'[Request_Created_Datetime], RefDate, DAY)
)

// Count of active open requests aging beyond 3 days
Open Aging > 3 Days = 
-- Production alternative: VAR Current_Date = TODAY()
VAR Current_Date = DATE(2026, 1, 15)
RETURN
CALCULATE(
    COUNTROWS('fact_operational_request'),
    FILTER(
        'fact_operational_request',
        'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" } &&
        DATEDIFF('fact_operational_request'[Request_Created_Datetime], Current_Date, DAY) > 3
    )
)

// Active open tickets that have already exceeded their target SLA hours (or flagged as breached)
Open At-Risk SLA Breached = 
VAR RefDate = DATE(2026, 1, 15)
RETURN
CALCULATE(
    COUNTROWS('fact_operational_request'),
    FILTER(
        'fact_operational_request',
        'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" } &&
        (
            DATEDIFF('fact_operational_request'[Request_Created_Datetime], RefDate, HOUR) > 'fact_operational_request'[SLA_Target_Hours] ||
            'fact_operational_request'[SLA_Status] IN { "SLA Breached", "Breached SLA (Open)" }
        )
    )
)
```

## Folder 05: Team Performance
```dax
// Proportion of workload handled by each team member
Analyst Workload Share % = 
VAR CurrentMemberRequests = [Total Task Created]
VAR AllMembersRequests = 
    CALCULATE(
        [Total Task Created],
        ALLSELECTED('dim_team_member')
    )
RETURN
    DIVIDE(CurrentMemberRequests, AllMembersRequests, 0)

// Validates Team Lead dispatch share (~7%) against total team volume using REMOVEFILTERS
Team Lead Share % = 
VAR LeadVolume = 
    CALCULATE(
        [Total Task Created],
        'dim_team_member'[is_lead] = 1
    )
VAR TotalTeamVolume = 
    CALCULATE(
        [Total Task Created],
        REMOVEFILTERS('dim_team_member')
    )
RETURN
    DIVIDE(LeadVolume, TotalTeamVolume, 0)
```
