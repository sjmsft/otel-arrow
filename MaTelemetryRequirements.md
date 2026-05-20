# Geneva Agent Telemetry Component Input Requirements

This document describes the input requirements of the four Geneva Agent telemetry sinks
(`MaEventTable`, `MaQosTable`, `AgentMetrics`, and `MaTelemetryEvents`). It is intended
to inform the design of a `TelemetryListener` layer that acts as an interface between a
hosted df_engine library and the Geneva Agent (hosting process).

---

## 1. MaEventTable

`MaEventTable` is the primary logging table for all Geneva Agent internal events and
diagnostics. Schema constants are defined in
[src/shared/publicinc/IEvents.h](src/shared/publicinc/IEvents.h).

### MaEventTable Inputs

- **Level** (`INT`): Event severity level
  - `MA_LEVEL_NONE` (0)
  - `MA_LEVEL_CRITICAL` (1)
  - `MA_LEVEL_ERROR` (2)
  - `MA_LEVEL_WARNING` (3)
  - `MA_LEVEL_INFO` (4)
  - `MA_LEVEL_VERBOSE` (5)
  - `MA_LEVEL_FATAL` (-1), `MA_LEVEL_FATAL_DUMP` (-2)
- **Pid** (`ULONG`): Process ID
- **Tid** (`ULONG`): Thread ID
- **Stream** (`WCHAR[64]`): Event stream / component name (`MA_EVENTS_MAX_STREAM_LEN`)
- **Instance** (`ULONG`): Stream instance number
- **ActivityId** (`WCHAR[37]`): GUID-format activity ID for correlation
- **File** (`PCWSTR`): Source file path (`__WFILE__`)
- **Function** (`PCWSTR`): Function name (`__WFUNCTION__`)
- **Line** (`ULONG`): Source line number
- **MDRESULT** (`LONG`): MA-specific result code
- **ErrorCode** (`DWORD`): System error code (`GetLastError()`)
- **ErrorCodeMsg** (`PCWSTR`): Human-readable error message
- **Message** (`PCWSTR`): Log message text
- **Category** (`MaEventType`): Event category enumeration (e.g. `MaCatDataUpload`,
  `MaCatIngestion`, `MaCatInfrastructure`)

The schema has 15 fields total (`MA_EVENT_TABLE_FIELD_COUNT`).

### MaEventTable API

```cpp
IEventStream1::Message(
    INT      ErrorLevel,
    MDRESULT ErrorResult,
    PCWSTR   File,
    PCWSTR   Function,
    ULONG    Line,
    PCWSTR   Fmt, ...);
```

Convenience macros: `MAWriteEvent`, `MAWriteEvent1`, … `MAWriteEvent5` automatically
populate `__WFILE__`, `__WFUNCTION__`, and `__LINE__`.

For events whose `Message` originates outside MA (e.g. from a hosted library), use:

```cpp
IEventStream1::ExternalMessage(
    INT         ErrorLevel,
    MDRESULT    ErrorResult,
    PCWSTR      File,
    PCWSTR      Function,
    ULONG       Line,
    ULONG       Pid,
    ULONG       ThreadId,
    MaEventType ErrorType,
    PCWSTR      DiagStreamName,
    PCWSTR      Message);
```

`ExternalMessage` is preferred for the `TelemetryListener` because it does not interpret
the message as a `printf` format string.

---

## 2. MaQosTable (`MaQosEvent`)

`MaQosTable` tracks operational metrics for performance and reliability monitoring.
Defined in [src/shared/publicinc/IEvents.h](src/shared/publicinc/IEvents.h).

### MaQosTable Inputs

- **operation** (`LPCWSTR`): Operation name (e.g. `MaPush`, `MaPull`,
  `MaRunTaskTransmit`)
- **object** (`LPCWSTR`): Object/resource being operated on (e.g. table name, endpoint)
- **success** (`bool`): Whether operation succeeded
- **result** (`int`): HTTP status code or result code
- **retries** (`int`): Number of retry attempts
- **dataDelayInMilliseconds** (`INT64`): Delay between data timestamp and processing
- **durationInMilliseconds** (`INT64`): Operation duration
- **dataSizeInBytes** (`INT64`): Size of data processed
- **dataItemReadCount** (`INT64`): Number of items read
- **dataItemWriteCount** (`INT64`): Number of items written
- **startTime** (`ULONGLONG`): Operation start timestamp (UTC)
- **endTime** (`ULONGLONG`): Operation end timestamp (UTC)

### MaQosTable API

```cpp
IEventStream1::LogQosEvent(
    LPCWSTR   operation,
    LPCWSTR   object,
    bool      success,
    int       result,
    int       retries,
    INT64     dataDelayInMilliseconds,
    INT64     durationInMilliseconds,
    INT64     dataSizeInBytes,
    INT64     dataItemReadCount,
    INT64     dataItemWriteCount,
    ULONGLONG startTime,
    ULONGLONG endTime);
```

Two simpler overloads also exist for cases without retry/read/write counts.

---

## 3. AgentMetrics (MDM Metrics)

`AgentMetrics` publishes MDM (Metrics Data Model) metrics to a customer's MDM account
for monitoring. Implemented in
[src/shared/MAEvents/MAEvents.cpp](src/shared/MAEvents/MAEvents.cpp).

### AgentMetrics Inputs

- **MetricName** (`LPCSTR`): Name of the metric
- **MetricValue** (`double`): Numeric value of the metric
- **EventName** (`std::wstring`, optional): Event name dimension
- **AdditionalDimensions** (`std::unordered_map<std::string, std::wstring>`, optional):
  Additional key/value dimensions

### Standard Metric Names

| Constant | String |
| --- | --- |
| `MDM_METRIC_NAME_DATA_DELAY_IN_SECONDS` | `DataDelayInSeconds` |
| `MDM_METRIC_NAME_ROWS_SENT` | `EventsSent` |
| `MDM_METRIC_NAME_ROWS_LOGGED` | `EventsLogged` |
| `MDM_METRIC_NAME_ROWS_DROPPED` | `EventsDropped` |
| `MDM_METRIC_NAME_SUCCEEDED_UPLOAD_TASKS` | `SucceededUploadTasks` |
| `MDM_METRIC_NAME_FAILED_UPLOAD_TASKS` | `FailedUploadTasks` |
| `MDM_METRIC_NAME_TIMEDOUT_UPLOAD_TASKS` | `TimedoutUploadTasks` |
| `MDM_METRIC_NAME_ETW_EVENTS_LOGGED` | `EtwEventsLogged` |
| `MDM_METRIC_NAME_ETW_EVENTS_LOST` | `EtwEventsLost` |
| `MDM_METRIC_NAME_STORAGE_REQUEST` | `StorageRequests` |
| `MDM_METRIC_NAME_STORAGE_FAILURES` | `StorageFailures` |
| `MDM_METRIC_NAME_MA_CPU_USAGE` | `CpuUsage` |
| `MDM_METRIC_NAME_MA_MEMORY_USAGE` | `MemoryUsage` |
| `MDM_METRIC_NAME_EXTENSION_FAILURES` | `ExtensionFailures` |
| `MDM_METRIC_NAME_GCS_REQUEST` | `ServiceRequest` |

### Initialization

`AgentMetrics` must be initialized once before metrics can be emitted:

```cpp
IEvents1::SetupAgentMdmMetrics(
    PCWSTR Namespace,
    PCWSTR MdmAccountName,
    PCWSTR OnbehalfIdentity,
    std::unordered_map<std::wstring, std::wstring>& TenantRoleIdentitiesMap,
    std::unordered_map<std::wstring, std::wstring>& OtherIdentitiesMap,
    bool   UseFullIdentity,
    const std::wstring& EventNameRegexFilter);
```

### AgentMetrics API

```cpp
IEventStream1::LogNumericMetric(
    LPCSTR              MetricName,
    double              MetricValue,
    const std::wstring& EventName,
    std::unordered_map<std::string, std::wstring>& AdditionalDimensions);
```

Related variants: `LogNumericMetricForOnbehalf`, `LogAggregatedMetric`,
`LogAggregatedEventsLost`, `LogEventsDroppedMetric`, `LogStorageMetric`,
`LogAgentCostMetric`.

---

## 4. MaTelemetryEvents

`MaTelemetryEvents` is the structured telemetry table for Geneva common agent
telemetry with rich, schema-driven dimensions. Defined in
[src/shared/inc/IMonAgentTelemetry.h](src/shared/inc/IMonAgentTelemetry.h) and
implemented in
[src/shared/Telemetry/MonAgentTelemetry.cpp](src/shared/Telemetry/MonAgentTelemetry.cpp).

Each row consists of a fixed set of **common dimensions** plus an
**operation-specific** dimension set determined by the `Operation` value.

### Common Dimensions (every row)

| Field | Type | Description |
| --- | --- | --- |
| `Operation` | wstring | Operation name (e.g. `RowsSent`, `Errors`, `CpuUsage`) |
| `OperationValue` | int64 | Primary numeric value for the operation |
| `EventName` | wstring | Name of event being processed |
| `Namespace` | wstring | Customer namespace |
| `DataRegion` | wstring | Data center region |
| `ComponentRegion` | wstring | Component deployment region |
| `LogsAccount` | wstring | Logs storage account |
| `LogsEnvironment` | wstring | Environment (prod, dev, …) |
| `LogsMoniker` | wstring | Logs identifier |
| `ComponentName` | wstring | Name of the component |
| `ComponentTenant` | wstring | Tenant identifier |
| `ComponentCategory` | wstring | Category (e.g. `Collection`) |
| `OS` | wstring | Operating system |
| `ComponentVersion` | wstring | Agent version |
| `OnbehalfServiceIdentity` | wstring | OBO identity |
| `InputType` | wstring | Type of input (e.g. `Logs`) |
| `InputSubtype` | wstring | Subtype of input |
| `ConfigVersion` | wstring | Configuration version |
| `FinalDestination` | wstring | Final storage destination |
| `AssetIdentity` | wstring | Asset/machine identity |
| `OSVersion` | wstring | OS version |
| `ProcessorArchitecture` | wstring | CPU architecture |
| `ComponentIdentity` | wstring | Component GUID |
| `traceparent` | wstring | W3C trace context |
| `traceparentId` | wstring | Parent span ID |
| `Priority` | wstring | Task priority |
| `ComponentEnvironment` | wstring | Environment label |
| `ComponentRoleInstance` | wstring | Role instance |
| `LauncherType` | wstring | How agent was launched |
| `NodeType` | wstring | Node type |
| `FabricType` | wstring | Fabric type |
| `CloudType` | wstring | Cloud environment |

### Operation-Specific Dimensions

**`RowsSent`** — `c_SentDimensions`:
`DataStartTimeUtc`, `DataEndTimeUtc`, `OperationDurationMs`, `DataAgeMs`,
`NextDestinationValue`, `MinLevel`, `SchemaIDs`, `BlobSizeInBytes`,
`DataSizeInBytes`, `AddedDelayMs`, `ResourceId`, `NextDestinationType`, `Success`,
`HttpStatus`, `OnbehalfCustomerNamespace`, `RowsRead`,
`MemoryUseEstimateInBytes`, `NextDestinationReason`.

**`RowsReceived`** — `c_ReceivedDimensions`:
`TotalRowsInCache`, `TotalCacheSizeInBytes`, `AvgRowsReceivedSizeInBytes`,
`MaxRowsReceivedSizeInBytes`.

**`RowsDropped`** — `c_DroppedDimensions`:
`ErrorType`, `ErrorSubtype`, `OnbehalfCustomerNamespace`.

**`Errors`** — `c_ErrorsDimensions`:
`ErrorType`, `ErrorSubtype`, `HealthStatus`.

**`CpuUsage`** — `c_ResourceCpuDimensions`:
`ProcessorCount`, `LimitRate`, `LimitWeight`, `LimitParentRate`,
`LimitObjectActiveProcesses`, `LimitObjectParentActiveProcesses`,
`LimitEffectiveRate`, `IntervalSeconds`, `Description`, `ProcessorAffinity`,
`ConfiguredCpuThreshold`, `ComponentUptimeSeconds`.

**`MemoryUsage`** — `c_ResourceMemoryDimensions`:
`LimitProcessMemory`, `LimitObjectMemory`, `ProcessMemoryPeakUsed`,
`LimitObjectMemoryPeakUsed`, `Description`, `HandleCount`,
`ConfiguredMemoryAllowance`.

**`DiskUsage`** — `c_ResourceDiskDimensions`:
`ComponentCacheSizeInBytes`, `ComponentCachePeakSizeInBytes`,
`ComponentCacheQuotaInBytes`, `FreeAvailableInBytes`, `TotalAvailableInBytes`,
`DirectoryName`.

**`ServiceRequest`** — `c_GcsDimensions`:
`ResponseSizeInBytes`, `Success`, `HttpStatus`, `FailureClass`, `ApiName`,
`Url`, `AuthType`. (`McsDimensions` for MCS variants.)

**`ComponentStart`** — `c_StartDimensions`:
`Success`, `Reason`.

**`Heartbeat`** — `c_FeatureUsageDimensions`:
`EnabledFeatures`.

### Per-Operation Data Struct (`TelemetryDataQos`)

For data-flow operations (`RowsSent`, etc.), the agent collects a
`TelemetryDataQos` struct populated with:

- `DataSentCount`, `DataStartTimeUtc`, `DataEndTimeUtc` (`LONGLONG`)
- `BlobSizeInBytes`, `DataSizeInBytes`, `DurationMs`, `AddedDelayMs`,
  `DataAgeMs`, `RowsRead`, `OperationMemoryInBytes` (`LONGLONG`)
- `StatusCode`, `MinLevel` (`LONG`)
- `Destination` (`TelemetryQosDest`), `DestReason` (`TelemetryQosDestReason`)
- `EventName`, `DestObject`, `SchemaIDs`, `ResourceId`, `LogsMoniker`,
  `OboNamespace`, `OptTraceparent` (`std::wstring`)
- `IsSuccess` (`bool`)

---

## Initialization Flow

1. Create the events object:

   ```cpp
   PIEvents1 events = CreateIEvents2(TablePath, MA_ALL_EVENT_TABLE,
                                     SecAttributes, ToConsole, TableLockSidString);
   ```

2. Create a per-component event stream:

   ```cpp
   PIEventStream1 stream = nullptr;
   events->CreateEventStream(MaCatInfrastructure, L"DataFlowLib", 0, stream);
   ```

3. Set up QoS / counter tables:

   ```cpp
   events->SetupQosTables(MA_QOS_EVENT_TABLE, MA_COUNTER_EVENT_TABLE);
   ```

4. Initialize MDM `AgentMetrics` (if used):

   ```cpp
   events->SetupAgentMdmMetrics(...);
   ```

5. `MaTelemetryEvents` is set up via `CreateIMonAgentTelemetry()` in
   [src/shared/Telemetry/MonAgentTelemetry.cpp](src/shared/Telemetry/MonAgentTelemetry.cpp).

All four sinks are accessed through `IEventStream1` (per-component) or `IEvents1`
(per-process) and are thread-safe.

---

## TelemetryListener Design Implications

The `TelemetryListener` layer between the df_engine library and the Geneva Agent should
expose the following capabilities, each routed to the appropriate sink:

| Listener Capability | Sink |
| --- | --- |
| Diagnostic / log message (level, message, source) | `MaEventTable` via `ExternalMessage` |
| Operation timing & throughput (op, success, duration, bytes, retries) | `MaQosTable` via `LogQosEvent` |
| Aggregated counter / gauge with dimensions | `AgentMetrics` via `LogNumericMetric` |
| Structured per-data-flow operation telemetry (rows sent/received/dropped, resource usage) | `MaTelemetryEvents` via the `MonAgentTelemetry` accumulator APIs |

Each listener call should:

- Carry an **activity / trace ID** so events can be correlated across sinks
  (`SetThreadActivityId`, `GetThreadTraceContexId`).
- Provide enough source-location context for `MaEventTable`.
- Use canonical metric / operation names from `IEvents.h` where possible so existing
  dashboards continue to work.
- Be safe to call from any thread.

---

## What the df_engine library Must Provide vs. What the Agent Already Knows

A large fraction of the fields enumerated above are **environmental/identity context**
that the Geneva Agent already discovers from its configuration, IMDS / Azure metadata,
launcher arguments, and persistent state. The df_engine library does **not** need to
supply these — `TelemetryListener` should fill them in on the library's behalf, so the
library-facing API can stay small.

### Source legend

- **AGENT** – Already known to the Geneva Agent. The `TelemetryListener` injects it; the
  library never sees the field.
- **LIB** – Must come from the df_engine library. It describes *what the library did*
  and cannot be inferred by the host.
- **AUTO** – Computed by the platform/runtime (e.g. `GetCurrentProcessId`,
  `__WFILE__`, `QueryPerformanceCounter`). The listener fills these in at the call
  site; the library does not pass them.

### MaEventTable fields

| Field | Source | Notes |
| --- | --- | --- |
| Pid, Tid | AUTO | `GetCurrentProcessId`, `GetCurrentThreadId` |
| File, Function, Line | AUTO | Set via `__WFILE__`/`__WFUNCTION__`/`__LINE__` macros at the listener boundary |
| Stream, Instance | AGENT | Fixed when the listener creates its `IEventStream1` for the library |
| ActivityId | AGENT | Thread-local, managed via `SetThreadActivityId`/`GetThreadTraceContexId` |
| Category (`MaEventType`) | AGENT | Listener picks a single category for the library (e.g. `MaCatDataTransform` or `MaCatDataRouting`) |
| ErrorCodeMsg | AGENT/AUTO | Derived from `ErrorCode` |
| **Level** | **LIB** | Severity of the log line |
| **MDRESULT** | **LIB** (optional) | MA result code; default `MD_NO_ERROR` if not relevant |
| **ErrorCode** | **LIB** (optional) | OS / library error code; default 0 |
| **Message** | **LIB** | The log text |

> Net library input for a log entry: **(level, message, optional error code,
> optional MDRESULT)**.

### MaQosTable fields

| Field | Source | Notes |
| --- | --- | --- |
| **operation** | **LIB** | Library-defined operation name (or canonical one from `IEvents.h`) |
| **object** | **LIB** | Resource the operation acted on |
| **success** | **LIB** | |
| **result** | **LIB** | HTTP / library status code |
| **retries** | **LIB** | |
| **dataDelayInMilliseconds** | **LIB** | Data freshness; library-only knowledge |
| **durationInMilliseconds** | **LIB** | Or AUTO if the listener wraps the call |
| **dataSizeInBytes** | **LIB** | |
| **dataItemReadCount / WriteCount** | **LIB** | |
| **startTime / endTime** | LIB or AUTO | Listener can capture timestamps automatically; library only needs to provide them when its measurement window differs from the listener call |

> Net library input: essentially the entire QoS row. None of it is environmental.

### AgentMetrics (MDM) fields

| Field | Source | Notes |
| --- | --- | --- |
| Namespace, MdmAccountName, OnbehalfIdentity, TenantRoleIdentitiesMap, OtherIdentitiesMap, UseFullIdentity, EventNameRegexFilter | AGENT | All passed to `SetupAgentMdmMetrics` once at agent startup; the library never sees them |
| Common MDM dimensions (`Namespace`, `StorageType`, `AccountName`, `OnbehalfIdentity`, `OnbehalfCustomerNamespace`, `Container`, etc.) | AGENT | Filled in by the agent's MDM emitter |
| **MetricName** | **LIB** | Use canonical `MDM_METRIC_NAME_*` strings where applicable |
| **MetricValue** | **LIB** | |
| **EventName** | LIB (optional) | Only when the metric is per-event |
| **AdditionalDimensions** | LIB (optional) | Library-specific dimensions |

> Net library input: **(metric name, value, optional event name, optional extra
> dimensions)**.

### MaTelemetryEvents fields

The **entire** common-dimension block (≈ 32 fields) is populated by the agent —
`MonAgentTelemetry::SetTelemetryProperties`, `SetConfigProperties`,
`UpdateCommonFields`, `SetAzureMetadata`, and `SetCommonMetricDimensions` source
these from the config reader, `AzureMetadata`, `InfraInfo`, persistent hash, and
agent identity.

| Common dimension | Source | Provider |
| --- | --- | --- |
| `Namespace`, `ConfigVersion`, `OnbehalfServiceIdentity` | AGENT | Config reader |
| `LogsAccount`, `LogsEnvironment`, `LogsMoniker`, `FinalDestination` | AGENT | Config / GCS service info |
| `DataRegion`, `ComponentRegion`, `CloudType`, `AssetIdentity`, `ResourceId`, `NodeType`, `FabricType` | AGENT | `AzureMetadata`, `InfraInfo`, IMDS |
| `OS`, `OSVersion`, `ProcessorArchitecture` | AGENT | OS lookup at startup |
| `ComponentVersion`, `ComponentIdentity`, `ComponentName`, `ComponentTenant`, `ComponentEnvironment`, `ComponentRoleInstance`, `ComponentCategory` | AGENT | Agent self-identity / config |
| `LauncherType` | AGENT | Command line at startup |
| `Priority`, `InputType`, `InputSubtype` | AGENT | Per-stream config (listener fills based on registration) |
| `traceparent`, `traceparentId` | AUTO | From thread-local trace context |
| **`Operation`** | **LIB** | Selects the operation-specific dimension set |
| **`OperationValue`** | **LIB** | The primary numeric for the operation |
| **`EventName`** | **LIB** (often) | Event name being processed; sometimes filled by listener if there is only one |

For the operation-specific dimension sets, only a small subset is library-only data;
the rest is agent or platform supplied:

| Operation | LIB-only fields | AGENT/AUTO fields |
| --- | --- | --- |
| `RowsSent` (`c_SentDimensions`) | `DataStartTimeUtc`, `DataEndTimeUtc`, `OperationDurationMs`, `DataAgeMs`, `BlobSizeInBytes`, `DataSizeInBytes`, `RowsRead`, `AddedDelayMs`, `MinLevel`, `SchemaIDs`, `Success`, `HttpStatus`, `MemoryUseEstimateInBytes` | `NextDestinationValue`, `NextDestinationType`, `NextDestinationReason`, `ResourceId`, `OnbehalfCustomerNamespace` |
| `RowsReceived` (`c_ReceivedDimensions`) | `TotalRowsInCache`, `TotalCacheSizeInBytes`, `AvgRowsReceivedSizeInBytes`, `MaxRowsReceivedSizeInBytes` | — |
| `RowsDropped` (`c_DroppedDimensions`) | `ErrorType`, `ErrorSubtype` | `OnbehalfCustomerNamespace` |
| `Errors` (`c_ErrorsDimensions`) | `ErrorType`, `ErrorSubtype`, `HealthStatus` | — |
| `CpuUsage`, `MemoryUsage`, `DiskUsage` | (none — library should not emit) | All AGENT — collected by `MonAgentTelemetry`'s own resource sampler |
| `ServiceRequest` (`c_GcsDimensions`/`c_McsDimensions`) | `ResponseSizeInBytes`, `Success`, `HttpStatus`, `FailureClass`, `ApiName`, `Url`, `AuthType`, `Endpoint`, `Query`, `ServiceType` | — (only library knows what call it made) |
| `ComponentStart` (`c_StartDimensions`) | `Success`, `Reason` | — |
| `Heartbeat` (`c_FeatureUsageDimensions`) | `EnabledFeatures` | — |

### Implications for the TelemetryListener interface

Because the agent already owns the entire identity/environment block, the
listener interface exposed to the df_engine library can be **dramatically smaller**
than the union of fields above. A practical surface is roughly:

```cpp
class ITelemetryListener
{
public:
    // -> MaEventTable
    virtual void Log(
        Level level,
        std::wstring_view message,
        int errorCode = 0,
        long mdResult = 0) = 0;

    // -> MaQosTable
    virtual void RecordQos(
        std::wstring_view operation,
        std::wstring_view object,
        bool   success,
        int    resultCode,
        int    retries,
        int64_t dataDelayMs,
        int64_t durationMs,
        int64_t dataSizeBytes,
        int64_t itemsRead,
        int64_t itemsWritten) = 0;

    // -> AgentMetrics (MDM)
    virtual void EmitMetric(
        std::string_view  name,
        double            value,
        std::wstring_view eventName = {},
        std::span<const std::pair<std::string, std::wstring>> extraDims = {}) = 0;

    // -> MaTelemetryEvents (RowsSent / RowsReceived / RowsDropped / Errors / ServiceRequest)
    virtual void RecordRowsSent(std::wstring_view eventName, const RowsSentInfo&) = 0;
    virtual void RecordRowsReceived(std::wstring_view eventName, const RowsReceivedInfo&) = 0;
    virtual void RecordRowsDropped(std::wstring_view eventName,
                                   std::wstring_view errorType,
                                   std::wstring_view errorSubtype,
                                   int64_t count) = 0;
    virtual void RecordError(std::wstring_view errorType,
                             std::wstring_view errorSubtype,
                             std::wstring_view healthStatus) = 0;
    virtual void RecordServiceRequest(const ServiceRequestInfo&) = 0;
};
```

Everything else — process / thread IDs, source location, stream name, category,
activity ID, namespace, account, region, OS, agent version, asset identity,
trace context, MDM dimensions for tenant/role/identity, and all `MaTelemetryEvents`
common dimensions — is filled in by the listener implementation from the existing
agent state. The df_engine library therefore only needs to know **what it just did**,
not **where it is running**.
