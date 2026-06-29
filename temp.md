# Business Schedule Evaluation Engine Proposal

## 1. Problem Statement

The application allows users to configure recurring business schedules
for portfolio processing.

A schedule represents **business intent**, not a technical scheduler.
Users define only the scheduling metadata, and the system is responsible
for determining when the schedule is eligible for execution.

The schedule metadata **does not store execution state** such as:

-   Next Execution Time
-   Last Execution Time
-   Cron Expression
-   Trigger Metadata

Instead, only business metadata is persisted.

## 2. Current Data Model

### Schedule Configuration

  Field                Required   Description
  -------------------- ---------- -------------------------------------------------
  ScheduleId           Yes        Unique schedule identifier
  PrimaryPortfolioId   Yes        Primary portfolio
  Frequency            Yes        Weekly, Monthly, Quarterly, Semi-Annual, Annual
  BusinessDate         Yes        Anchor business date
  StartDate            No         Optional activation date
  EndDate              No         Optional expiry date

### Schedule Mapping

A schedule may be mapped to multiple portfolios.

  ScheduleId   SecondaryPortfolioId
  ------------ ----------------------
  101          P100
  101          P101
  101          P102

One Schedule → Many Portfolios.

------------------------------------------------------------------------

## 3. Expected Behaviour

Example

Business Date = 29-Jun-2026

Frequency = MONTHLY

Expected executions

-   29-Jun-2026
-   29-Jul-2026
-   29-Aug-2026
-   29-Sep-2026

Weekly

-   Every 7 days from Business Date

Quarterly

-   Every 3 months from Business Date

Semi-Annual

-   Every 6 months from Business Date

Annual

-   Every year from Business Date

### Optional Start / End Date

If supplied

    StartDate <= Today <= EndDate

must be true.

If omitted

The schedule remains active indefinitely until deleted.

------------------------------------------------------------------------

## 4. Architectural Observation

This is **not** primarily a scheduling problem.

It is a **business calendar evaluation problem**.

The business metadata is the source of truth.

The application derives whether a schedule is eligible on the current
business date.

No execution state is stored in the schedule configuration.

------------------------------------------------------------------------

# 5. Proposed Architecture

                       Spring Scheduler
                    (every hour / day)
                            │
                            ▼
                  SchedulePollingService
                            │
                            ▼
             Load Active Schedule Definitions
                            │
                            ▼
             Evaluate Business Eligibility
                            │
                            ▼
                Load Portfolio Mappings
                            │
                            ▼
               Execute Business Process

Only one scheduler exists inside the application.

The scheduler never creates one task per ScheduleId.

------------------------------------------------------------------------

# 6. Proposed Components

## SchedulePollingService

Responsible only for periodically waking up.

``` java
@Component
public class SchedulePollingService {

    @Scheduled(cron = "0 0 * * * *")
    public void evaluateSchedules() {

        // Load schedules

        // Evaluate

        // Execute
    }

}
```

------------------------------------------------------------------------

## ScheduleRepository

``` java
public interface ScheduleRepository {

    List<Schedule> findActiveSchedules();

}
```

------------------------------------------------------------------------

## ScheduleMappingRepository

``` java
public interface ScheduleMappingRepository {

    List<PortfolioMapping> findByScheduleId(Long scheduleId);

}
```

------------------------------------------------------------------------

## ScheduleEvaluator

This becomes the core business component.

``` java
public interface ScheduleEvaluator {

    boolean shouldExecute(
            Schedule schedule,
            LocalDate businessDate);

}
```

This component decides whether a schedule is due on the supplied
business date.

------------------------------------------------------------------------

## DefaultScheduleEvaluator

``` java
public class DefaultScheduleEvaluator
        implements ScheduleEvaluator {

    @Override
    public boolean shouldExecute(
            Schedule schedule,
            LocalDate businessDate) {

        // Validate StartDate

        // Validate EndDate

        // Evaluate frequency

        return false;
    }

}
```

------------------------------------------------------------------------

## Frequency Evaluation

``` java
switch(schedule.getFrequency()) {

case WEEKLY:

case MONTHLY:

case QUARTERLY:

case SEMI_ANNUAL:

case ANNUAL:

}
```

Each frequency compares the configured Business Date against today's
business date.

Examples

Weekly

    Difference in days % 7 == 0

Monthly

    Same day-of-month
    AND
    Months elapsed >= 0

Quarterly

    Months elapsed % 3 == 0

Semi-Annual

    Months elapsed % 6 == 0

Annual

    Same day
    Same month

------------------------------------------------------------------------

# 7. Execution Flow

    Spring Scheduler

    ↓

    Load Active Schedules

    ↓

    For each Schedule

    ↓

    Validate Start Date

    ↓

    Validate End Date

    ↓

    Evaluate Frequency

    ↓

    Eligible?

    ↓

    YES

    ↓

    Load Portfolio Mapping

    ↓

    Execute Business Logic

------------------------------------------------------------------------

# 8. Why Not One Scheduler Per Schedule?

Avoid creating individual runtime schedulers because:

-   Difficult to manage thousands of schedules
-   Dynamic updates become complex
-   Restart requires rebuilding all runtime schedules
-   Increased memory usage
-   Cluster coordination becomes harder

A single polling scheduler scales much better.

------------------------------------------------------------------------

# 9. Why Not Quartz?

Quartz is an excellent enterprise scheduler when execution state,
triggers, misfire recovery, clustering and persistent jobs are required.

However, the current model intentionally stores only business metadata
and does not persist scheduler state.

Introducing Quartz would add unnecessary complexity unless those
capabilities become explicit requirements.

------------------------------------------------------------------------

# 10. Open Design Question

One important business decision remains.

Suppose

Business Date

    29-Jun

The application is unavailable on 29-Jun.

It starts again on 30-Jun.

Should the system

**Option A**

Skip the missed execution permanently?

or

**Option B**

Automatically execute the missed occurrence after recovery?

Without storing execution history, the system can determine whether a
schedule **should** execute on a business date, but it cannot determine
whether it **already has**.

This business rule should be clarified before implementation.

------------------------------------------------------------------------

# 11. Recommendation

-   Keep schedule metadata as the single source of truth.
-   Do not create one runtime scheduler per schedule.
-   Use a single Spring `@Scheduled` polling service.
-   Encapsulate all recurrence logic in a dedicated `ScheduleEvaluator`.
-   Keep frequency evaluation independent of Spring and fully unit
    testable.
-   Load portfolio mappings only after a schedule has been deemed
    eligible.
-   Separate business scheduling logic from the underlying scheduling
    framework.
