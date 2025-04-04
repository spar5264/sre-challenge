Note: I have basic experience working with MySQL (mostly non-administration, like working with database tables). Never had a chance to work with Prometheus so the conclusion is based solely off the warning message information and [Prometheus documentation](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)

```sh
  - alert: MysqlTransactionDeadlock
    expr: increase(mysql_global_status_innodb_row_lock_waits[2m]) > 0
    for: 3m
    labels:
      severity: warning
    annotations:
      dashboard: database-metrics
      summary: 'Mysql Transaction Waits'
    description: 'There are `{{ $value | humanize }}` MySQL connections waiting for a stale transaction to release.'
```

## Description

| Line                                                                                       | Explanation                                                                                                                                                                                                                                  |
|--------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| alert:                                                                                     | The name of the alert. Must be a valid label value. |
| MysqlTransactionDeadlock  | type of alert about MySQL encountering a deadlock during transaction execution. Commonly happens when two or more transactions are waiting for each other to release lock on resources, which causes a loop when none can proceed. |
| expr: increase(mysql_global_status_innodb_row_lock_waits[2m]) > 0                          | this must be the event circumstances which trigger the alert. If I understand it correctly, it means that innodb row lock waits has increased in the last 2 minutes and the nuber of blocked transactions is greater than 0.                 |
| for: 3m                                                                                    | alerts are considered firing once they have been returned for this long. Alerts which have not yet fired for long enough are considered pending.                                                                           |
| labels:                                                                                    | some keywords or key-phrases usually assigned here to organize the alerts                                                                                                                                                                    |
| severity: warning                                                                          | specifies the severity of the issue (alert), this one is not critical, only warning.                                                                                                                                                         |
| annotations:                                                                               | readable context shown in dashboards and alerts |
| dashboard: database-metrics                                                                | should refer to the name or ID of the related Grafana dashboard |
| summary: 'Mysql Transaction Waits'                                                         | a title used for notifications or alerts |
| description: 'There are `{{ $value / humanize }}` MySQL connections waiting for a stale transaction to release.' | provides humanized description of the issue with an additional data to help uderstand the issue quickly (changed to / intentionally to not mess with the git table structure) |

Possible causes of the issue: 
1. Two or more transactions are modifying the same rows and waiting for each other to finish
2. Poorly designed queries or missing indexes can cause excessive row or table lock
3. Higher isolation levels like `SERIALIZE`      
4. Long transactions can also make conflicts 

## Investigation: 

1. Run `SHOW ENGINE INNODB STATUS;` to get details about locks and transactions causing deadlocks.
2. Use `SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;` to list running transactions.
3. Query `INFORMATION_SCHEMA.INNODB_LOCKS` and `INFORMATION_SCHEMA.INNODB_LOCK_WAITS` for details on locking conflicts.
4. Check slow queries using `SHOW PROCESSLIST;` and analyze queries with `EXPLAIN ANALYZE`.
5. Look for inefficient indexing or long-running transactions.
6. Review application logs to detect transactions that are not being committed or rolled back properly.
7. Use Prometheus and Grafana dashboards to check `mysql_global_status_innodb_row_lock_time_avg` or `mysql_global_status_innodb_row_lock_current` for lock trends.

## Improvements
1. It possibly can be modified to increase convenience and speed of understanding, investigation and fixing the issue.
```sh
description: 'There are {{ $value | humanize }} `MySQL transactions waiting due to row locks. Investigate long-running transactions and potential deadlocks.`'
```
1.1. If we have a guide for such investigations and the alert is clickable (never interacted with one of those, so just an assumption) we can also add a link to it. Also, it may be usable for other alerts, not only this one.
```sh   
description: 'There are {{ $value | humanize }} MySQL transactions waiting due to row locks. [Investigate](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html#:~:text=To%20view%20the%20last%20deadlock,to%20the%20mysqld%20error%20log.) long-running transactions and potential deadlocks.
```
2. Reduces unnecessary alerts caused by minor or short-lived lock waits:
```sh   
   expr: increase(mysql_global_status_innodb_row_lock_waits[5m]) > 5
for: 5m
```
