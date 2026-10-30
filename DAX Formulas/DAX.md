# DAX (Data Analysis Expression)

## Folder 01: Throughput & Workload
```Code snippet
Total Task Created = countrows(fact_operational_request)
Total Task Closed = calculate(countrows(fact_operational_request), fact_operational_request[Request_Status] = "Closed Complete")
Total Task Cancelled = calculate(countrows(fact_operational_request), fact_operational_request[Request_Status] = "Cancelled")
Active Open Backlog = CALCULATE(COUNTROWS('fact_operational_request'),'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" })
Monthly Target Achievement % = DIVIDE([Closed Tasks Throughput],800,0)
Monthly Target Variance = [Closed Tasks Throughput]-800
Net Queue Delta = [Total Request Ingested]-[Closed Tasks Throughput]-[Cancelled Tasks Count]
```
## Folder 02: SLA & Compliance   
```Code snippet
SLA Eligible Closed Tasks = CALCULATE(COUNTROWS(fact_operational_request), AND(fact_operational_request[Request_Status]="Closed Complete", fact_operational_request[SLA_Eligiblity]=1))

SLA Met Tasks = calculate(countrows(fact_operational_request), fact_operational_request[SLA_Status]="SLA Met")

SLA Breached Tasks = calculate(countrows(fact_operational_request), fact_operational_request[SLA_Status]="SLA Breached")

SLA Compliance % = DIVIDE([SLA Met Tasks],[SLA Eligible Closed Tasks],0)

SLA Compliance % MoM Delta = 
VAR Current_Month_SLA = [SLA Compliance %]
VAR Prior_Month_SLA = CALCULATE([SLA Compliance %], DATEADD(dim_date_closed[calendar_date],-1,MONTH))
RETURN 
IF(
    NOT ISBLANK(Current_Month_SLA) && NOT ISBLANK(Prior_Month_SLA), 
    Current_Month_SLA-Prior_Month_SLA
)
```

## Folder 03: Cycle Time & Bottlenecks   
```Code snippet
Avg Total Resolution Days = CALCULATE(AVERAGE(fact_operational_request[Total_Resolution_Days]), fact_operational_request[Request_Status]="Closed Complete")
Avg Queue Wait Hours = AVERAGE(fact_operational_request[Queue_Wait_Hours])
Avg User Info Wait Days = AVERAGE(fact_operational_request[User_Info_Wait_Days])
Avg GRC Approval Wait Days = AVERAGE(fact_operational_request[Grc_Approval_Wait_Days])
Avg Fulfilment Hours = AVERAGE(fact_operational_request[Fulfilment_Hours])
Avg Closure Lag Hours = AVERAGE(fact_operational_request[Closure_Lag_Hours])
```

## Folder 04: Backlog & Aging   
```Code snippet
Current Open Age Days = 
VAR Current_Date = today()
RETURN
AVERAGEX(
    FILTER(
        fact_operational_request, fact_operational_request[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" }
    ),
    DATEDIFF(fact_operational_request[Request_Created_Datetime], Current_Date,DAY)
)

Open Aging > 3 Days = 
VAR Current_Date = TODAY()
RETURN
CALCULATE(
    COUNTROWS(fact_operational_request), 
    fact_operational_request[Request_Status] in { "Assigned", "Work In Progress", "Pending User Info" },
    DATEDIFF(fact_operational_request[Request_Created_Datetime],Current_Date,DAY)>3
)

Open At-Risk SLA Breached = CALCULATE(COUNTROWS('fact_operational_request'),'fact_operational_request'[Request_Status] IN { "Assigned", "Work In Progress", "Pending User Info" }, fact_operational_request[SLA_Status]="SLA Breached")
```

## Folder 05: Team Performance   
```Code snippet
Analyst Workload Share % = 
VAR CurrentMemberRequests = [Total Requests Created]
VAR AllMembersRequests = 
    CALCULATE(
        [Total Requests Created],
        ALLSELECTED('dim_team_member')
    )
RETURN
    DIVIDE(CurrentMemberRequests, AllMembersRequests, 0)
```
