# Audit, logging, and alerting

Who, what, where, when

## Objectives

* Improve existing audit mechanism to provide comprehensive information about actions performed
* Ensure audit emission cannot be bypassed
* Alert when auditing thresholds are exceeded, such as frequency of failures across all APIs, or per user, or per IP address
* Emit audit records asynchronously to reduce latency to the caller
* Allow flexibility in where audit records get stored

## High level design

[TODO: request comments regarding replacement of AuditStore with enhanced logging engine]

Auditing will be done with a dedicated API but utilize the logging engine for storage.

Logging engine enhancements will be made to facilitate audit goals, but also improve logging in general. Logging will become an asynchronous operation, to the extent possible, by queuing the data to be logged quickly, then formatting and storing using background goroutines owned by the logging engine.

Logging asynchronously presents some caveats. For example, when logging a struct asynchronously care must be taken to ensure the contents of the data being logged does not change while the log record is being created and formatted. To achieve this one of two methods will be employed (TBD after benchmarking):

1. types of arguments will be inspected and copies made for mutable types before enqueuing
2. all formatting will happen before enqueuing

Another issue with asynchronous logging is deciding what to do when the queue is full. To keep things simple the caller will be blocked up to a configurable timeout waiting for enqueue to succeed. If enqueue succeeds before the timeout then a counter representing logging contention will be incremented and the total emitted periodically. If enqueue times out then an alert is emitted.

[TODO: determine queue approach: buffered channel, ring buffer, ...]

Queue(s) will be monitored for percent full and periodically reported.

Asynchronous logging also requires explicit call to logging shutdown method to ensure queues are flushed. All application exit paths must be covered.

[TODO: determine if audit API needs to extend deeper into app pkg or 100% of auditable events will come through server REST API]
[TODO: research which auditable events currently bypass server REST API]

The logging engine will be enhanced to allow discrete logging "levels". Currently an application wide logging level is configured and any log records matching that level *or lower* will be emitted. These logging levels will remain, but support for zero or more discrete logging "level"'s will be added, meaning only records matching the current log level or one of the discrete "level"s are emitted. Within the logging engine, any level below 10 (trace through critical/fatal, plus reserved) will behave as it does currently, but any "level" above 10 will be considered discrete. Audit records will have a level above 10.

Logging engine enhancements will be implemented via the mlog package and will be compatible with existing usage.

Logging engine will allow logging levels and discrete levels to different targets (files, databases, etc) via configuration. Zap logger will continue to be used internally.

## Data model

A single audit record is emitted for each event (add, delete, login, ...). Multiple auditable events may be emitted for a single API call.

| name       | type              | description     |
| ---------- | ----------------- | -----------     |
| Id         | string            | audit record id |
| CreateAt   | int64             | timestamp of record creation, UTC |
| API        | string            | endpoint path |
| Event      | string            | e.g. add, delete, login, ... |
| UserId     | string            | id of user calling the API |
| SessionId  | string            | id of session used to call the API |
| Client     | string            | e.g. webapp, mmctl, user-agent |
| IPAddress  | string            | ip address of Client |
| Meta       | map[string]string | API specific info; e.g. user id being deleted |

```go
type AuditRecord struct {
  Id          string
  CreateAt    int64
  API         string
  Event       string
  UserId      string
  SessionId   string
  Client      string
  IPAddress   string
  Meta        map[string]string
}
```

## Audit API

From web.Context, audit API will be modified to have variadic parameters, but otherwise be unchanged. However all calls will need to be reviewed to ensure required fields are added. Conversion of name value strings to Meta map will happen asynchronously.

Context will continue to be used to auto-populate some of the audit record fields.
[TODO: request comments regarding security/reliability of data within Context]

[TODO: api examples, code snippets]

## Storage

Storage options will be administrator configurable via logging engine configuration. When storing to file, typically the audit records will go to a separate file from general logging.

## Alerting

Alerting will be achieved via a plugin logger target, and configured used a discrete log level. Destination(s) for alerts can be email, database, Mattermost channel post, or other.

## Reporting

## Configuration

## Best Practices
