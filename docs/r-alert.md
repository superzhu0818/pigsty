# Alerting

Alerts are critical for daily fault response and improving system availability.

Missing alerts will lead to reduced availability and false alerts will lead to reduced sensitivity, and it is necessary to design the alert rules prudently.

* Reasonable definition of alert levels and the corresponding processing flow
* Reasonable definition of alert indicators, removal of duplicate alert items, and replenishment of missing alert items
* Scientific configuration of alert thresholds based on historical monitoring data to reduce the false alert rate.
* Reasonably rationalize the special case rules to eliminate false alerts caused by maintenance work, ETL, and offline query.

## Implementation

There are two parallel alerting system in Pigsty

* [Prometheus](http://p.pigsty.cc/alerts) + [AlertManager](http://a.pigsty.cc/#/alerts) (primary)
* [Grafana](http://demo.pigsty.cc/d/pgsql-alert) (standby)


The two systems are functionally equivalent, with different focus capabilities, and can be used simultaneously as a backup to complement each other.



## Taxonomy

**By Urgency**

* P0: CRIT: Incidents that have significant off-site impact and require emergency intervention to handle. For example, primary library down, replication outage. (Incident)
* P1: WARN: Incidents with minor off-site impact, or incidents with redundant handling, requiring response handling at the minute level. (WARN)
* P2: INFO: Impending impact, let loose may worsen at the hourly level, requires response at the hourly level. (Incident)
* Pigsty uses the `level` tag `{0, 1, 2}`, with the `severity` tag `{CRIT,WARN,INFO}` to identify the level of urgency of an alert.

**By Level**

* Infrastructure: Operating system, hardware resources, infrastructure software, load balancing and other alerts, usually handled by operations and maintenance staff.
* Database level: database cluster itself alerts, DBA focus on. Generated by PG, PGB, Exporter's own monitoring metrics.
* Application level: Application alerts are handled by the business side itself, but the DBA sets alerts for business metrics such as QPS, TPS, Rollback, Seasonality, etc.
* Pigsty uses the `category` tag `{infra,pgsql}` to identify the hierarchical classification of alerts: infrastructure vs. database clusters.

**By Category**

* Errors: PG Down, PGB Down, Exporter Down, Stream Replication Outage, Single Cluster Multi-Master
* Traffic: QPS, TPS, Rollback, Seasonality
* Latency: Average Response Time, Replication Latency
* Saturation: Connection Stacking, Number of Idle Transactions, CPU, Disk, Age (Transaction Number), Buffers



## Visualization

In various monitoring panels, Pigsty uses ** timeline status charts** to present alert information. 
The horizontal axis represents a time period and a color bar represents an alert event. 
Only alerts in the **Firing** state are displayed in the alert chart, alerts in the Pending state are usually hidden or grayed out.



## Alert Rules

alert rules can be roughly divided into four categories by type: error, delay, saturation, and traffic. Among them.

* Errors: mainly focus on the **survivability (Aliveness)** of each component, as well as network outages, brain fractures and other abnormalities, and the level is usually higher (P0|P1).
* Latency: mainly concerned with query response time, replication latency, slow queries, and long transactions.
* Saturation: mainly focus on CPU, disk (these two belong to system monitoring but very important for DB so incorporate), connection pool queue, number of database back-end connections, age (essentially the saturation of available thing numbers), SSD life, etc.
* Traffic: QPS, TPS, Rollback (traffic is usually related to business metrics belongs to business monitoring, but incorporated because it is important for DB), seasonality of QPS, burst of TPS.

## Prometheus alert rules

Alert rules are defined using Prometheus syntax, the full alert rules are detailed in.

* [Infrastructure alert rules](https://github.com/Vonng/pigsty/blob/master/roles/prometheus/files/rules/infra-alert.yml)
* [Database cluster alert rules](https://github.com/Vonng/pigsty/blob/master/roles/prometheus/files/rules/pgsql-alert.yml)



## Pigsty Alerts

### Error*

A database instance going down will immediately trigger a P0 alert.


```yaml
# database server down
- alert: PostgresDown
  expr: pg_up < 1
  for: 1m
  labels: { level: 0, severity: CRIT, category: pgsql }
  annotations:
    summary: "CRIT PostgresDown {{ $labels.ins }}@{{ $labels.instance }}"
    description: |
      pg_up[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value }} < 1
      http://g.pigsty/d/pgsql-instance?var-ins={{ $labels.ins }}
```

When using pigsty, Pgbouncer & Postgres are bonded 

When using PostgreSQL in a production environment, 
Pgbouncer instances and Postgres are a fate-sharing community, 
Pgbouncer failure is basically equivalent to Postgres failure,
and its severity is aligned with Postgres.


```yaml
# database connection pool down
- alert: PgbouncerDown
  expr: pgbouncer_up < 1
  for: 1m
  labels: { level: 0, severity: CRIT, category: pgsql }
  annotations:
    summary: "CRIT PostgresDown {{ $labels.ins }}@{{ $labels.instance }}"
    description: |
      pgbouncer_up[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value }} < 1
      http://g.pigsty/d/pgsql-instance?var-ins={{ $labels.ins }}
```

Monitoring agent Exporter downtime usually indicates a serious failure: 
HAProxy and Node Exporter downtime usually means that the load balancer and database nodes themselves are down and need to be focused on


```yaml
#==============================================================#
#                       Agent Aliveness                        #
#==============================================================#
# node & haproxy aliveness are determined directly by exporter aliveness
# including: node_exporter, pg_exporter, pgbouncer_exporter, haproxy_exporter
- alert: AgentDown
  expr: agent_up < 1
  for: 1m
  labels: { level: 0, severity: CRIT, category: infra }
  annotations:
    summary: 'CRIT AgentDown {{ $labels.ins }}@{{ $labels.instance }}'
    description: |
      agent_up[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value  | printf "%.2f" }} < 1
      http://g.pigsty/d/pgsql-alert?viewPanel=22
```

The duration threshold for all survivability tests is set to 1 minute, and a 15s collection period usually means 4 consecutive failures. Regular fast restart operations usually do not trigger survivability alerts.



**Split Brain**

A database cluster that should have and only have one instance of the master leader. That is, the cluster should have only one **partition** under normal circumstances. If the number of partitions in the cluster is not 1 (0 means no leader, greater than 1 means a cluster) it means that the cluster has entered an abnormal state: unwritable or brain cracked, which will immediately trigger a P0 alert. Because the detection threshold is 1 minute, so the regular Failover and Switchover usually do not easily trigger this alert.

```yaml
# cluster partition: split brain
- alert: PostgresPartition
  expr: pg:cls:partition != 1
  for: 1m
  labels: { level: 0, severity: CRIT, category: pgsql }
  annotations:
    summary: "CRIT PostgresPartition {{ $labels.cls }}@{{ $labels.job }} {{ $value }}"
    description: |
      pg:cls:partition[cls={{ $labels.cls }}, job={{ $labels.job }}] = {{ $value }} != 1
```





### Lag

There are two alarms related to replication latency: replication outage, and high replication latency, graded as P1 warning.

* where **replication break** is an error that is determined using the indicator: `pg_downstream_count{state="streaming"}`. A break alert is triggered if there is a negative change in the number of slave libraries in the current `streaming` state. `walsender` will determine the state of replication, the slave will break directly, and the buffer backlog will go from `streaming` to `catchup` state, which will also trigger this alert. Replication interruptions can cause clients to read stale data, which has some off-site impact and is rated as P1.

* Replication latency can be determined using either latency time or latency bytes. The number of delay bytes is the authoritative indicator. Under normal conditions, replication latency time in the order of 100 milliseconds and replication latency bytes in the order of 100 KB are normal. Based on historical experience data, the time alert threshold of 1MB and 1s is currently used.


```yaml
#==============================================================#
#                         Replication                          #
#==============================================================#
# replication break for 1m triggers a P1 alert (WARN: heal in 5m)
- alert: PostgresReplicationBreak
  expr: changes(pg_downstream_count{state="streaming"}[5m]) > 0
  # for: 1m
  labels: { level: 1, severity: WARN, category: pgsql }
  annotations:
    summary: "WARN PostgresReplicationBreak: {{ $labels.ins }}@{{ $labels.instance }}"
    description: |
      changes(pg_downstream_count{ins={{ $labels.ins }}, instance={{ $labels.instance }}, state="streaming"}[5m]) > 0


# replication lag bytes > 1MiB or lag seconds > 1s
- alert: PostgresReplicationLag
  expr: pg:ins:lag_bytes > 1048576 or pg:ins:lag_seconds > 1
  for: 1m
  labels: { level: 1, severity: WARN, category: pgsql }
  annotations:
    summary: "WARN PostgresReplicationLag: {{ $labels.ins }}@{{ $labels.instance }}"
    description: |
      pg:ins:lag_bytes[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value | printf "%.0f" }} > 1048576 or
      pg:ins:lag_seconds[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value | printf "%.2f" }} > 1


```


In addition, there are corresponding alert rules for query latency and disk latency.

For example, the average disk read/write response time lasts more than 32ms for one minute, or the average query RT in Pgbouncer exceeds 16ms will trigger a P1 alert.

```yaml

# read latency > 32ms (typical on pci-e ssd: 100µs)
- alert: NodeDiskSlow
  expr: node:dev:disk_read_rt_1m > 0.032 or node:dev:disk_write_rt_1m > 0.032
  for: 1m
  labels: { level: 1, severity: WARN, category: node }
  annotations:
    summary: 'WARN NodeReadSlow {{ $labels.ins }}@{{ $labels.instance }} {{ $value  | printf "%.6f" }}'
    description: |
      node:dev:disk_read_rt_1m[ins={{ $labels.ins }}] = {{ $value  | printf "%.6f" }} > 32ms


# pgbouncer avg response time > 16ms (database level)
- alert: PgbouncerQuerySlow
  expr: pgbouncer:db:query_rt_1m > 0.016
  for: 3m
  labels: { level: 1, severity: WARN, category: pgsql }
  annotations:
    summary: "WARN PgbouncerQuerySlow: {{ $labels.ins }}@{{ $labels.instance }} [{{ $labels.datname }}]"
    description: |
      pgbouncer:db:query_rt_1m[ins={{ $labels.ins }}, instance={{ $labels.instance }}, datname={{ $labels.datname }}] = {{ $value | printf "%.3f" }} > 0.016

```



### Saturation alerts

Saturation metrics main resources, including many system-level monitoring metrics. Mainly include: CPU, disk (these two belong to system monitoring but very important for DB so incorporate), connection pool queue, number of database back-end connections, age (essentially saturation of available thing numbers), SSD life, etc.

**Database Stress**

Database stress is the combined maximum (percentage, but can exceed 100% when overloaded) of: machine CPU usage, Pgbouncer time utilization, Postgres time utilization (14 introduced). Stress is the most important metric in Pigsty, concentrating on the load water level of the database instances and clusters.


```yaml
#==============================================================#
#                        Saturation                            #
#==============================================================#
# instance pressure higher than 70% for 1m triggers a P1 alert
- alert: PostgresPressureHigh
  expr: ins:pressure1 > 0.70
  for: 1m
  labels: { level: 1, severity: WARN, category: pgsql }
  annotations:
    summary: "WARN PostgresPressureHigh: {{ $labels.ins }}@{{ $labels.instance }}"
    description: |
      ins:pressure1[ins={{ $labels.ins }}, instance={{ $labels.instance }}] = {{ $value | printf "%.3f" }} > 0.70

```



**Queuing Detection**

Heap contains two main types of metrics, the number of back-end connections and active connections of the PG itself on the one hand, and the queuing of the connection pool on the other.

PGB queuing is the decisive indicator, it represents that perceptible blocking has occurred on the user side, so the presence of queuing lasting 1 minute triggers a P0 alert.

When using **Session Pooling** mode, this alert indicator can be relaxed appropriately.

```yaml
# pgbouncer client queue exists
- alert: PgbouncerClientQueue
  expr: pgbouncer:db:waiting_clients > 1
  for: 1m
  labels: { level: 0, severity: CRIT, category: pgsql }
  annotations:
    summary: "CRIT PgbouncerClientQueue: {{ $labels.ins }}@{{ $labels.instance }} [{{ $labels.datname }}]"
    description: |
      pgbouncer:db:waiting_clients[ins={{ $labels.ins }}, instance={{ $labels.instance }}, datname={{ $labels.datname }}] = {{ $value | printf "%.0f" }} > 1

```

The number of back-end connections is an important warning indicator, and if the back-end connections consistently reach the maximum number of connections, it often means an avalanche as well. The number of queued connections in the connection pool also reflects this situation, but does not cover the case where the application is directly connected to the database.

Currently, Pigsty uses **connection utilization** as an alerting metric, i.e. the percentage of available database connections that have been used, with a P1 alert departing if it exceeds 70% for 3 minutes.


```yaml
# database connection usage > 70%
- alert: PostgresConnUsageHigh
  expr: pg:db:conn_usage > 0.70
  for: 3m
  labels: { level: 1, severity: WARN, category: pgsql }
  annotations:
    summary: "WARN PostgresConnUsageHigh: {{ $labels.ins }}@{{ $labels.instance }} [{{ $labels.datname }}]"
    description: |
      pg:db:conn_usage[ins={{ $labels.ins }}, instance={{ $labels.instance }}, datname={{ $labels.datname }}] = {{ $value | printf "%.3f" }} > 0.70

```

**Idle in Transaction**

That is, the number of connections with `Idle in Transaction` status in the database, more than 2 for 3 minutes is the departure of P1 alert.

```yaml
# database connection usage > 70%
- alert: PostgresIdleInXact
  expr: pg:db:ixact_backends > 1
  for: 3m
  labels: { level: 2, severity: INFO, category: pgsql }
  annotations:
    summary: "Info PostgresIdleInXact: {{ $labels.ins }}@{{ $labels.instance }} [{{ $labels.datname }}]"
    description: |
      pg:db:ixact_backends[ins={{ $labels.ins }}, instance={{ $labels.instance }}, datname={{ $labels.datname }}] = {{ $value | printf "%.0f" }} > 1

```


**Resource alert**

Age (XID) usage exceeds 80% departure P0 alert, which means the system is about to run out of transaction number resources and enter XID Wraparound state


```yaml
# database age saturation > 80%
- alert: PostgresXidWarpAround
  expr: pg:db:age > 0.80
  for: 1m
  labels: { level: 0, severity: CRIT, category: pgsql }
  annotations:
    summary: "CRIT PostgresXidWarpAround: {{ $labels.ins }}@{{ $labels.instance }} [{{ $labels.datname }}]"
    description: |
      pg:db:age[ins={{ $labels.ins }}, instance={{ $labels.instance }}, datname={{ $labels.datname }}] = {{ $value | printf "%.0f" }} > 80%

```


