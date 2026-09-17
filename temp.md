Below is the consolidated implementation specification, using **`code_name` consistently** and preserving the single-table, relational-value approach.

# Code Master Configuration Service

## 1. Purpose

Introduce a centralized **Code Master Configuration** mechanism for maintaining application configuration values in a single database table and exposing them through a strongly typed Java API.

The solution must support:

* Single/scalar values
* Multiple/list values
* String, Integer, Boolean, Decimal and extensible value types
* Human-readable relational database representation
* In-memory caching
* Typed Java accessors
* Fail-fast validation for incorrect type/cardinality access
* Cache refresh without exposing a partially rebuilt cache

The application must not need to know how configuration values are persisted.

---

# 2. Goals

### Functional goals

1. Store configuration in a single `CODE_MASTER_CONFIG` table.
2. Identify each logical configuration using `code_name`.
3. Support both scalar and list values.
4. Preserve list ordering.
5. Store values in a human-readable relational form.
6. Load active configuration into an application cache.
7. Expose typed access methods.
8. Detect invalid configuration during cache construction.
9. Detect incorrect API usage at runtime.
10. Support cache refresh.

### Non-goals

This component is not intended to be:

* A secrets-management system
* A general-purpose JSON configuration store
* A replacement for application properties that must be known before database connectivity exists
* A relational master-data framework for complex business entities

---

# 3. High-Level Architecture

```text
                    ┌─────────────────────┐
                    │       Database      │
                    │                     │
                    │ CODE_MASTER_CONFIG  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Repository       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Cache Builder      │
                    │                     │
                    │ Validate            │
                    │ Group               │
                    │ Convert             │
                    │ Order               │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   In-Memory Cache   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Config Service    │
                    │                     │
                    │ getString()         │
                    │ getInteger()        │
                    │ getBoolean()        │
                    │ getDecimal()        │
                    │ getList()           │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Application      │
                    └─────────────────────┘
```

---

# 4. Database Design

Create one table:

```sql
CODE_MASTER_CONFIG
```

Recommended structure:

```sql
CREATE TABLE CODE_MASTER_CONFIG (
    ID            BIGINT PRIMARY KEY,
    CODE_NAME     VARCHAR(100) NOT NULL,
    CONFIG_TYPE   VARCHAR(50)  NOT NULL,
    DESCRIPTION   VARCHAR(500),
    VALUE         VARCHAR(1000) NOT NULL,
    VALUE_TYPE    VARCHAR(30)  NOT NULL,
    IS_LIST       BOOLEAN NOT NULL DEFAULT FALSE,
    SEQUENCE_NO   INTEGER,
    ACTIVE        BOOLEAN NOT NULL DEFAULT TRUE,
    CREATED_AT    TIMESTAMP NOT NULL,
    UPDATED_AT    TIMESTAMP NOT NULL
);
```

The exact SQL types should follow the database conventions already used by the existing project.

---

# 5. Column Semantics

## `ID`

Unique physical row identifier.

## `CODE_NAME`

Logical identifier of the Code Master configuration.

Examples:

```text
MAX_RETRY
TIMEOUT_MS
FEATURE_ENABLED
SUPPORTED_CURRENCY
RETRYABLE_STATUS_CODES
```

All rows belonging to the same logical list share the same `CODE_NAME`.

## `CONFIG_TYPE`

Business/domain classification.

Examples:

```text
SYSTEM
BUSINESS
FEATURE
REFERENCE
INTEGRATION
```

This must not be confused with `VALUE_TYPE`.

## `DESCRIPTION`

Human-readable explanation of what the Code Master entry represents.

## `VALUE`

Actual configuration value.

Values are stored as relational rows rather than JSON arrays.

Examples:

```text
3
5000
true
USD
EUR
GBP
```

## `VALUE_TYPE`

Defines how the value should be interpreted.

Initial supported types:

```text
STRING
INTEGER
BOOLEAN
DECIMAL
```

The design should allow additional types to be introduced later.

## `IS_LIST`

Defines the cardinality of the logical configuration.

```text
FALSE → single value
TRUE  → list of values
```

This is intentionally explicit rather than inferred from the number of database rows.

## `SEQUENCE_NO`

Defines ordering for list values.

Example:

```text
SUPPORTED_CURRENCY

1 → USD
2 → EUR
3 → GBP
4 → INR
```

For scalar values, `SEQUENCE_NO` should be `NULL`.

## `ACTIVE`

Only active configuration participates in the runtime cache.

## `CREATED_AT / UPDATED_AT`

Standard audit fields.

---

# 6. Data Examples

## Single value

```text
CODE_NAME          VALUE    VALUE_TYPE    IS_LIST    SEQUENCE_NO
-----------------------------------------------------------------
MAX_RETRY          3        INTEGER       FALSE      NULL
```

## Boolean

```text
CODE_NAME          VALUE    VALUE_TYPE    IS_LIST    SEQUENCE_NO
-----------------------------------------------------------------
FEATURE_ENABLED    true     BOOLEAN       FALSE      NULL
```

## String list

```text
CODE_NAME           VALUE    VALUE_TYPE    IS_LIST    SEQUENCE_NO
------------------------------------------------------------------
SUPPORTED_CURRENCY  USD      STRING        TRUE       1
SUPPORTED_CURRENCY  EUR      STRING        TRUE       2
SUPPORTED_CURRENCY  GBP      STRING        TRUE       3
SUPPORTED_CURRENCY  INR      STRING        TRUE       4
```

## Integer list

```text
CODE_NAME             VALUE    VALUE_TYPE    IS_LIST    SEQUENCE_NO
-------------------------------------------------------------------
RETRYABLE_STATUS_CODE 408      INTEGER       TRUE       1
RETRYABLE_STATUS_CODE 429      INTEGER       TRUE       2
RETRYABLE_STATUS_CODE 500      INTEGER       TRUE       3
RETRYABLE_STATUS_CODE 502      INTEGER       TRUE       4
```

---

# 7. Data Integrity Rules

All rows with the same `CODE_NAME` must have identical:

```text
CONFIG_TYPE
VALUE_TYPE
IS_LIST
```

For example, this is invalid:

```text
SUPPORTED_CURRENCY | USD | STRING  | TRUE
SUPPORTED_CURRENCY | 1   | INTEGER | TRUE
```

A list must have:

```text
IS_LIST = TRUE
SEQUENCE_NO IS NOT NULL
```

A scalar must have:

```text
IS_LIST = FALSE
SEQUENCE_NO IS NULL
```

A scalar configuration must have exactly one active row.

A list configuration may have multiple active rows.

Sequence numbers within a list must be unique.

---

# 8. Java Domain Model

Create an internal database representation:

```java
public record CodeMasterConfigRow(
        String codeName,
        String configType,
        String description,
        String value,
        ValueType valueType,
        boolean list,
        Integer sequenceNo
) {}
```

Value type:

```java
public enum ValueType {
    STRING,
    INTEGER,
    BOOLEAN,
    DECIMAL
}
```

The enum should be extensible if additional configuration types are required.

---

# 9. Cached Representation

The cache should represent the logical configuration rather than individual database rows.

```java
public record CachedCodeMaster(
        String codeName,
        String configType,
        ValueType valueType,
        boolean list,
        List<String> values
) {}
```

Examples:

```text
MAX_RETRY

codeName  = MAX_RETRY
valueType = INTEGER
list      = false
values    = ["3"]
```

and:

```text
SUPPORTED_CURRENCY

codeName  = SUPPORTED_CURRENCY
valueType = STRING
list      = true
values    = ["USD", "EUR", "GBP", "INR"]
```

---

# 10. Cache Structure

Use:

```java
Map<String, CachedCodeMaster>
```

keyed by `codeName`.

Conceptually:

```text
cache
│
├── MAX_RETRY
│      ├── INTEGER
│      ├── SINGLE
│      └── ["3"]
│
├── FEATURE_ENABLED
│      ├── BOOLEAN
│      ├── SINGLE
│      └── ["true"]
│
└── SUPPORTED_CURRENCY
       ├── STRING
       ├── LIST
       └── ["USD", "EUR", "GBP", "INR"]
```

The cache must contain active configurations only.

---

# 11. Repository

Provide a repository operation capable of retrieving all active rows.

Example:

```java
List<CodeMasterConfigRow> findAllActive();
```

The database query should return list values in a deterministic order, preferably:

```sql
ORDER BY CODE_NAME, SEQUENCE_NO
```

The cache builder must still explicitly sort list values by `sequenceNo`; it must not rely solely on database ordering.

---

# 12. Cache Builder

Create a dedicated component:

```java
CodeMasterCacheBuilder
```

Responsibilities:

1. Retrieve configuration rows.
2. Group rows by `codeName`.
3. Validate metadata consistency.
4. Validate scalar/list cardinality.
5. Validate sequence numbers.
6. Validate and convert values.
7. Build `CachedCodeMaster`.
8. Produce a complete new cache.

Example flow:

```text
Rows
 │
 ▼
Group by codeName
 │
 ▼
Validate
 │
 ├── type consistency
 ├── list consistency
 ├── cardinality
 ├── sequence
 └── value conversion
 │
 ▼
CachedCodeMaster
 │
 ▼
Map<String, CachedCodeMaster>
```

---

# 13. Cache Initialization

At application startup:

```text
Application startup
       │
       ▼
Load active rows
       │
       ▼
Build cache
       │
       ├── Invalid → startup failure
       │
       └── Valid
             │
             ▼
        Publish cache
             │
             ▼
       Application ready
```

If configuration is mandatory for safe application operation, invalid configuration must prevent successful startup.

---

# 14. Public API

The application-facing interface is:

```java
public interface CodeMasterService {

    String getString(String codeName);

    int getInteger(String codeName);

    boolean getBoolean(String codeName);

    BigDecimal getDecimal(String codeName);

    <T> List<T> getList(
            String codeName,
            Class<T> elementType);
}
```

The application should not access:

```text
CodeMasterConfigRow
CachedCodeMaster
Map<String, ...>
Repository
```

directly.

---

# 15. Application Usage

### Integer

```java
int retryCount =
        codeMasterService.getInteger("MAX_RETRY");
```

### Boolean

```java
boolean enabled =
        codeMasterService.getBoolean("FEATURE_ENABLED");
```

### String

```java
String currency =
        codeMasterService.getString("DEFAULT_CURRENCY");
```

### String list

```java
List<String> currencies =
        codeMasterService.getList(
                "SUPPORTED_CURRENCY",
                String.class);
```

### Integer list

```java
List<Integer> retryableCodes =
        codeMasterService.getList(
                "RETRYABLE_STATUS_CODE",
                Integer.class);
```

The returned objects must be normal Java types and immediately usable by application code.

---

# 16. Type and Cardinality Contract

Every configuration has two independent characteristics:

```text
                  Code Master
                       │
              ┌────────┴────────┐
              │                 │
          VALUE_TYPE         CARDINALITY
              │                 │
      STRING/INTEGER       SINGLE/LIST
      BOOLEAN/DECIMAL
```

Examples:

```text
MAX_RETRY
    INTEGER + SINGLE

FEATURE_ENABLED
    BOOLEAN + SINGLE

SUPPORTED_CURRENCY
    STRING + LIST

RETRYABLE_STATUS_CODE
    INTEGER + LIST
```

---

# 17. Type Validation

`getInteger()` must only accept:

```text
VALUE_TYPE = INTEGER
IS_LIST = FALSE
```

`getBoolean()` must only accept:

```text
VALUE_TYPE = BOOLEAN
IS_LIST = FALSE
```

`getString()` must only accept:

```text
VALUE_TYPE = STRING
IS_LIST = FALSE
```

`getList()` must require:

```text
IS_LIST = TRUE
```

and the requested element type must match `VALUE_TYPE`.

---

# 18. Invalid API Usage

The service must fail fast.

## List requested for scalar

```java
codeMasterService.getList(
    "MAX_RETRY",
    Integer.class);
```

when:

```text
MAX_RETRY
INTEGER
SINGLE
```

must throw:

```text
ConfigTypeMismatchException
```

Do not convert:

```text
3 → [3]
```

---

## Scalar requested for list

```java
codeMasterService.getInteger(
    "SUPPORTED_CURRENCY");
```

when:

```text
SUPPORTED_CURRENCY
STRING
LIST
```

must throw:

```text
ConfigTypeMismatchException
```

Do not:

* return the first value
* concatenate values
* choose an arbitrary value

---

## Wrong scalar type

```java
codeMasterService.getInteger(
    "FEATURE_ENABLED");
```

when the configuration is:

```text
BOOLEAN + SINGLE
```

must throw:

```text
ConfigTypeMismatchException
```

---

# 19. Missing Configuration

If a requested `codeName` does not exist:

```java
codeMasterService.getInteger("UNKNOWN_CODE");
```

throw:

```java
ConfigNotFoundException
```

Do not silently return:

```text
0
false
null
empty list
```

unless an explicit default/optional API is introduced.

---

# 20. Value Conversion

Database values are stored as strings and converted into Java types.

Examples:

```text
"3"       → Integer 3
"true"    → Boolean true
"USD"     → String "USD"
"10.25"   → BigDecimal 10.25
```

Invalid conversion must fail.

Example:

```text
VALUE = "ABC"
VALUE_TYPE = INTEGER
```

must result in:

```text
ConfigValueConversionException
```

during cache construction.

This is preferable to discovering malformed configuration only when a particular code path accesses it.

---

# 21. Generic List API

The generic list method:

```java
<T> List<T> getList(
        String codeName,
        Class<T> elementType);
```

must verify both:

1. Configuration is actually a list.
2. Requested element type matches the configured value type.

Example:

```java
List<String> currencies =
    codeMasterService.getList(
        "SUPPORTED_CURRENCY",
        String.class);
```

Valid.

But:

```java
List<Integer> currencies =
    codeMasterService.getList(
        "SUPPORTED_CURRENCY",
        Integer.class);
```

must throw `ConfigTypeMismatchException`.

---

# 22. Cache Refresh

Provide:

```java
void refresh();
```

Refresh must follow this sequence:

```text
Existing cache
      │
      │ remains available
      ▼

Load database
      │
      ▼
Build NEW cache
      │
      ▼
Validate NEW cache
      │
      ├── Invalid
      │      │
      │      └── Keep existing cache
      │
      └── Valid
             │
             ▼
       Atomically replace
             │
             ▼
        New cache active
```

Never:

```text
clear cache
    ↓
load database
    ↓
rebuild
```

because this can expose an empty or partially constructed cache.

---

# 23. Thread Safety

The runtime cache is read-heavy and should be immutable after construction.

Preferred implementation:

```java
private volatile Map<String, CachedCodeMaster> cache;
```

Build a new map independently and replace the reference only after successful validation.

Application threads therefore see either:

```text
OLD COMPLETE CACHE
```

or:

```text
NEW COMPLETE CACHE
```

but never a partially populated cache.

---

# 24. Optional Default API

If the application needs defaults, they must be explicit.

For example:

```java
int retry =
    codeMasterService.getInteger(
        "MAX_RETRY",
        3);
```

or:

```java
Optional<Integer> retry =
    codeMasterService.findInteger("MAX_RETRY");
```

The standard typed getter should remain strict.

---

# 25. Exception Model

Introduce dedicated exceptions.

```java
ConfigNotFoundException
ConfigTypeMismatchException
ConfigValueConversionException
InvalidConfigurationException
```

### `ConfigNotFoundException`

Requested `codeName` does not exist in the active cache.

### `ConfigTypeMismatchException`

Requested Java type/cardinality does not match configuration.

### `ConfigValueConversionException`

Stored value cannot be converted to configured type.

### `InvalidConfigurationException`

Database configuration violates structural rules.

---

# 26. Error Message Requirements

Errors should contain enough information to diagnose the problem.

Example:

```text
Configuration 'SUPPORTED_CURRENCY' has type
STRING + LIST but INTEGER + SINGLE was requested.
```

For malformed configuration:

```text
Invalid Code Master configuration 'MAX_RETRY':
expected INTEGER value but found 'ABC'.
```

Do not include sensitive configuration values in logs if the component is later extended to handle sensitive data.

---

# 27. Validation Rules

The cache builder must reject:

### Duplicate scalar rows

```text
MAX_RETRY → 3
MAX_RETRY → 5
```

when `IS_LIST = FALSE`.

### Mixed value types

```text
SUPPORTED_CURRENCY → USD → STRING
SUPPORTED_CURRENCY → 1   → INTEGER
```

### Mixed cardinality

```text
SUPPORTED_CURRENCY → USD → LIST
SUPPORTED_CURRENCY → EUR → SINGLE
```

### Duplicate sequence

```text
USD → 1
EUR → 1
```

### Missing sequence

```text
USD → NULL
```

when `IS_LIST = TRUE`.

### Invalid sequence

Negative or otherwise invalid sequence numbers should be rejected according to project conventions.

### Invalid value

```text
MAX_RETRY → ABC → INTEGER
```

---

# 28. Database Indexing

At minimum, create an index supporting cache loading and lookup:

```text
(CODE_NAME, ACTIVE)
```

For list ordering:

```text
(CODE_NAME, SEQUENCE_NO)
```

The exact indexes should be aligned with the existing project's database standards and expected data volume.

---

# 29. Testing

## Repository tests

Verify:

* active records retrieved
* inactive records excluded
* ordering
* expected mapping

## Cache builder tests

Verify:

* scalar String
* scalar Integer
* scalar Boolean
* scalar Decimal
* String list
* Integer list
* duplicate scalar
* mixed value types
* mixed cardinality
* duplicate sequence
* missing sequence
* invalid conversion

## Service tests

Verify:

```java
getString()
getInteger()
getBoolean()
getDecimal()
getList()
```

and invalid calls:

```java
getList(scalar)
getInteger(list)
getBoolean(integer)
getInteger(string)
getList(wrongElementType)
```

## Cache refresh tests

Verify:

1. Existing cache is available.
2. New cache is constructed.
3. Invalid refresh does not replace existing cache.
4. Valid refresh replaces existing cache atomically.

---

# 30. Acceptance Criteria

The implementation is complete when all of the following are true:

* [ ] A single `CODE_MASTER_CONFIG` table is used.
* [ ] `CODE_NAME` is the logical configuration identifier.
* [ ] Scalar values are represented as one row.
* [ ] List values are represented as multiple rows.
* [ ] `IS_LIST` explicitly identifies cardinality.
* [ ] `VALUE_TYPE` explicitly identifies data type.
* [ ] List ordering is maintained using `SEQUENCE_NO`.
* [ ] Only active configurations are loaded.
* [ ] Configuration is loaded into an in-memory cache.
* [ ] Cache is validated before publication.
* [ ] Application accesses configuration through typed APIs.
* [ ] `getString()` returns `String`.
* [ ] `getInteger()` returns `int`.
* [ ] `getBoolean()` returns `boolean`.
* [ ] `getDecimal()` returns `BigDecimal`.
* [ ] `getList()` returns `List<T>`.
* [ ] Incorrect type access throws an explicit exception.
* [ ] Incorrect cardinality access throws an explicit exception.
* [ ] Missing configuration throws an explicit exception.
* [ ] Invalid database values are detected during cache construction.
* [ ] Cache refresh does not expose partial state.
* [ ] Unit and integration tests cover the defined validation rules.

---

# 31. Example Final Usage

The intended application experience is deliberately simple:

```java
int maxRetry =
        codeMasterService.getInteger("MAX_RETRY");

boolean featureEnabled =
        codeMasterService.getBoolean("FEATURE_ENABLED");

String defaultCurrency =
        codeMasterService.getString("DEFAULT_CURRENCY");

List<String> supportedCurrencies =
        codeMasterService.getList(
                "SUPPORTED_CURRENCY",
                String.class);
```

The business code should not contain:

```java
databaseRepository.find...
```

or:

```java
cache.get(...)
```

or:

```java
if (isList) ...
```

or:

```java
Integer.parseInt(...)
```

The Code Master service owns all of those concerns.

---

# 32. Final Design Principle

The implementation deliberately separates the concerns:

```text
┌──────────────────────────────────────────────┐
│ DATABASE                                     │
│                                              │
│ Human-readable relational rows               │
│                                              │
│ USD                                          │
│ EUR                                          │
│ GBP                                          │
│ INR                                          │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│ CACHE                                        │
│                                              │
│ SUPPORTED_CURRENCY                           │
│   STRING + LIST                              │
│   [USD, EUR, GBP, INR]                       │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│ JAVA API                                     │
│                                              │
│ getList("SUPPORTED_CURRENCY", String.class)  │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│ APPLICATION                                  │
│                                              │
│ List<String> currencies                      │
└──────────────────────────────────────────────┘
```

**The database optimizes for maintainability and readability. The cache optimizes for runtime access. The Java API optimizes for type safety and developer experience.**

This is the intended architecture for the Code Master Configuration implementation.
