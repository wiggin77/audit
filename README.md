# Audit, logging, and alerting

Who, what, where, when

## Objectives

* Improve existing audit mechanism to provide comprehensive information about actions performed and the resulting auditable events
* Ensure audit emission cannot be bypassed
* Alert when auditing thresholds are exceeded, such as frequency of failures across all APIs, or per user, or per IP address
* Emit audit records asynchronously to reduce latency to the caller
* Allow flexibility in where audit records get stored

## High level design

~~[TODO: request comments regarding replacement of AuditStore with enhanced logging engine]~~

Auditing will be done with a dedicated API but utilize the logging engine for storage.

Logging engine enhancements will be made to facilitate audit goals, but also improve logging in general. Logging will become an asynchronous operation, to the extent possible, by queuing the data to be logged quickly, then formatting and storing using background goroutines owned by the logging engine.

Logging asynchronously presents some caveats. For example, when logging a struct asynchronously care must be taken to ensure the contents of the data being logged does not change while the log record is being created and formatted. To achieve this ~~one of two methods will be employed (TBD after benchmarking)~~:

~~1. types of arguments will be inspected and copies made for mutable types before enqueuing~~
2. all formatting will happen before enqueuing

Another issue with asynchronous logging is deciding what to do when the queue is full. To keep things simple the caller will be blocked up to a configurable timeout waiting for enqueue to succeed. If enqueue succeeds before the timeout then a counter representing logging contention will be incremented and the total emitted periodically. If enqueue times out then an alert is emitted.

~~[TODO: determine queue approach: buffered channel, ring buffer, ...]~~

Queuing will be done via a buffered channel.

(phase 2) Queue(s) will be monitored for percent full and metrics provided via Prometheus.

Asynchronous logging also requires explicit call to logging shutdown method to ensure queues are flushed. All application exit paths must be covered.

~~[TODO: determine if audit API needs to extend deeper into app pkg or 100% of auditable events will come through server REST API]~~
~~[TODO: research which auditable events currently bypass server REST API]~~

Auditing at the Rest API and CLI layers will provide coverage.

(phase 2) The logging engine will be enhanced to allow discrete logging "levels". Currently an application wide logging level is configured and any log records matching that level *or lower* will be emitted. These logging levels will remain, but support for zero or more discrete logging "level"'s will be added, meaning only records matching the current log level or one of the discrete "level"s are emitted. Within the logging engine, any level below 10 (trace through critical/fatal, plus reserved) will behave as it does currently, but any "level" above 10 will be considered discrete. Audit records will have a level above 10.

(phase 2) Logging engine enhancements will be implemented via the mlog package and will be compatible with existing usage.

(phase 2) Logging engine will allow logging levels and discrete levels to different targets (files, databases, etc) via configuration. Zap logger will continue to be used internally.

## Data model

A single audit record is emitted for each event (add, delete, login, ...). Multiple auditable events may be emitted for a single API call.

| name       | type                   | description     |
| ---------- | ---------------------- | --------------- |
| ID         | string                 | audit record id |
| CreateAt   | int64                  | timestamp of record creation, UTC |
| Level      | string                 | e.g. audit-rest, audit-app, audit-model |
| APIPath    | string                 | rest endpoint  |
| Event      | string                 | e.g. add, delete, login, ... |
| Status     | string                 | e.g. attempt, success, fail, ... |
| UserId     | string                 | id of user calling the API |
| SessionId  | string                 | id of session used to call the API |
| Client     | string                 | e.g. webapp, mmctl, user-agent |
| IPAddress  | string                 | ip address of Client |
| Meta       | map[string]interface{} | API specific info; e.g. user id being deleted |

```go
type AuditRecord struct {
  ID          string
  //CreateAt    int64  -- added by logger
  Level       string
  APIPath     string
  Event       string
  Status      string
  UserId      string
  SessionId   string
  Client      string
  IPAddress   string
  Meta        map[string]interface{}
}
```

## Audit API

### Web/Rest Layer

From web.Context, audit API will be replaced with a `defer` based API where a partially populated audit record is generated and additional information is added by the Rest API. All calls will need to be reviewed to ensure required fields are added. Conversion of name value strings to Meta map will happen asynchronously.

Context will continue to be used to auto-populate some of the audit record fields.
~~[TODO: request comments regarding security/reliability of data within Context]~~

Example:

```go
func sampleRestAPI(c *Context, w http.ResponseWriter, r *http.Request) {
  // check inputs ...

  // auditRec is pre-populated with data from Context
  // Fail by default
  auditRec := c.MakeAuditRecord("sampleAPI", audit.Fail)
  // Log the audit record regardless of how method is exited,
  // including panic
  defer c.LogAuditRec(auditRec)

  // ... do whatever this rest API is supposed to do
  created_id, err := c.App.CreateSomething(...)
  if err != nil {
    c.Err = err
    return
  }

  // add any additional fields specific to this rest API
  auditRec.AddMeta("created_id", created_id)

  // mark audit record as successful
  auditRec.Success()
}
```

### App layer

~~New APIs will be added in the app layer to capture all auditable events. This is necessary because not all auditable events are triggered via the Rest API. This means certain code paths will emit multiple audit records for the same event. To reduce noise, the `APILayer` field will be used to filter records such that only the record emitted closest to the caller is kept. For example, a caller using the Rest API will have any app layer records filtered leaving only the Rest layer record.~~
~~[TODO: is filtering really necessary?]~~

~~[TODO: api examples, code snippets]~~

After further investigation, the app layer is not an ideal place for generating audit records. Too many fields needed for useful audit records are missing at this layer and would either need to be passed in or looked up.  Instead auditing will be done at the Rest API layer and CLI layer where all the needed calling context exists.

## Storage

(phase 2) Storage options will be administrator configurable via logging engine configuration. When storing to file, typically the audit records will go to a separate file from general logging.

In the case of error writing to the target storage several strategies can be employed. The strategy(s) chosen are specific to the target type (file, database, email, ...).

1. Retries based on count, interval and/or space left in target queue
2. Alert via different target (e.g. email)
3. Spool to local disk, with size cap, then write to target when it becomes available

(phase 2) To ensure audit logs cannot be unknowingly tampered with or corrupted it will be possible to configure the logging engine to sign log files for specific targets. When an audit store cannot be made secure, audit logs should be stored in multiple places (e.g. file and database) so they can reconciled if needed.

## Alerting

(phase 2) Alerting will be achieved via a plugin logger target, and configured using a discrete log level. Destination(s) for alerts can be email, database, Mattermost channel post, or other.

## Reporting

[TODO]

## Configuration

Phase 2

## Best Practices

[TODO]
