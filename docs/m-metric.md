# Metrics

Metrics lies in the core part of pigsty.


## Form

**Metrics** are atomic logical units of measure that can be updated and statistically aggregated over time periods.

Metrics typically exist as **time series with dimension labels**. As an example,
the `pg:ins:xact_commit_rate1m` metric in the Pigsty sandbox shows the number of transaction commits for the last 1 minute for all instances.

```json
pg:ins:xact_commit_rate1m{cls="pg-meta", ins="pg-meta-1", ip="10.10.10.10", role="primary"} 0
pg:ins:xact_commit_rate1m{cls="pg-test", ins="pg-test-1", ip="10.10.10.11", role="primary"} 327.6
pg:ins:xact_commit_rate1m{cls="pg-test", ins="pg-test-2", ip="10.10.10.12", role="replica"} 517.0
pg:ins:xact_commit_rate1m{cls="pg-test", ins="pg-test-3", ip="10.10.10.13", role="replica"} 0
```


You can calculate based on metrics: sum, drive, avg, percentile, etc.... 

```sql
$ sum(pg:ins:xact_commit_rate1m) by (cls)        -- sum by cluster
{cls="pg-meta"} 0
{cls="pg-test"} 844.6

$ avg(pg:ins:xact_commit_rate1m) by (cls)        -- average by cluster
{cls="pg-meta"} 0
{cls="pg-test"} 280

$ avg_over_time(pg:ins:qps_realtime[30m])        -- average of last 30 minute for each instance
pg:ins:qps_realtime{cls="pg-meta", ins="pg-meta-1", ip="10.10.10.10", role="primary"} 0
pg:ins:qps_realtime{cls="pg-test", ins="pg-test-1", ip="10.10.10.11", role="primary"} 130
pg:ins:qps_realtime{cls="pg-test", ins="pg-test-2", ip="10.10.10.12", role="replica"} 100
pg:ins:qps_realtime{cls="pg-test", ins="pg-test-3", ip="10.10.10.13", role="replica"} 0
```



## Model

Each **Metric** is a **Class** of data that usually instantiate to multiple **time series**. 
Different time series corresponding to the same metric are distinguished by **labels**.

You can identify a time series by Metric + Labels. 
Each **time series** is an array of (timestamp, fetch) binaries.

Pigsty uses Prometheus's metrics model, whose logical concept can be represented by the following SQL DDL.


```sql
-- Metrics Table,  Metric:TimeSeries = 1:n
CREATE TABLE metrics (
    id   INT PRIMARY KEY,         -- Metrics ID
    name TEXT UNIQUE              -- Metrics Name，[...and other meta data such as type]
);

-- Time Series Table
CREATE TABLE series (
    id        BIGINT PRIMARY KEY,               -- Time Series ID 
    metric_id INTEGER REFERENCES metrics (id),  -- MetricID which the time series belonged, refer metrics(id)
    dimension JSONB DEFAULT '{}'                -- Dimension information in the form of k-v pair
);

-- Time Series Data 
-- Holds sampled data point, a logical representation
CREATE TABLE series_data (
    series_id BIGINT REFERENCES series(id),     -- Time Series ID, refer series(id)
    ts        TIMESTAMP,                        -- Timestamp of the data point
    value     FLOAT,                            -- value of the data point
    PRIMARY KEY (series_id, ts)                 -- each data point can be identified by time series id and timestamp
);
```

Take `pg:ins:qps` as an example：

```sql
INSERT INTO metrics VALUES(1, 'pg:ins:qps');  -- It's a metric named pg:ins:qps, type GAUGE
INSERT INTO series VALUES                     -- The metrics contains 4 time-series, distinguished by dimension labels
(1001, 1, '{"cls": "pg-meta", "ins": "pg-meta-1", "role": "primary", "other": "..."}'),
(1002, 1, '{"cls": "pg-test", "ins": "pg-test-1", "role": "primary", "other": "..."}'),
(1003, 1, '{"cls": "pg-test", "ins": "pg-test-2", "role": "replica", "other": "..."}'),
(1004, 1, '{"cls": "pg-test", "ins": "pg-test-3", "role": "replica", "other": "..."}');
INSERT INTO series_data VALUES                 -- The underneath sampling data point
(1001, now(), 1000),                           -- instance pg-meta-1 qps 1000 at this moment
(1002, now(), 1000),                           -- instance pg-test-1 qps 1000 at this moment
(1003, now(), 5000),                           -- instance pg-test-2 qps 5000 at this moment
(1004, now(), 5001);                           -- instance pg-test-3 qps 5000 at this moment
```

* `pg_up` is a metrics with 4 time series , which represent aliveness status of all instance in the sandbox environment.
* `pg_up{ins": "pg-test-1", ...}` is a time series which represent aliveness of the specific instance `pg-test-1`.





## Sources

Pigsty has four main sources of monitoring data: **database**, **connection pool**, **operating system**, **load balancer**. Exposed to the public via the corresponding exporter.

![](_media/metrics_source.png)

Full sources include.

* PostgreSQL's own monitoring metrics
* Statistical metrics from the PostgreSQL logs
* PostgreSQL system directory information
* Metrics from Pgbouncer connection pool median price
* PgExporter metrics
* Metrics of the database working node Node
* Load balancer Haproxy metrics
* DCS (Consul) working metrics
* Monitoring system itself working metrics: Grafana, Prometheus, Nginx
* Blackbox probing metrics (TBD)

For a full list of available metrics, please refer to [Reference - Metrics List]() section



## Numbers

So, how many metrics does Pigsty contain in total? Here is a pie chart of the percentage of each metric source. We can see that the blue-green-yellow part on the right corresponds to the metrics exposed by the database and database-related components, while the red-orange part on the bottom left corresponds to the machine node-related metrics. The purple part on the top left is the metrics related to load balancers.

![](_media/metrics_ratio.png)

Among the database metrics, there are about 230 raw metrics related to postgres itself, about 50 raw metrics related to middleware, and based on these raw metrics, Pigsty carefully designed about 350 DB-related derived metrics through hierarchical aggregation and precomputation.

Thus, for each database cluster, there are 621 monitoring metrics purely for the database and its attachments. And there are 281 machine original metrics and 83 derived metrics for a total of 364. Adding the 170 metrics for the load balancer, we have a total of close to 1200 classes of metrics.

Note that here we must distinguish between metric and time-series.
Here we use the term class rather than individual. This is because a metric may correspond to multiple time-series. For example, if there are 20 tables in a database, then a metric like `pg_table_index_scan` will have 20 corresponding time series.

![](_media/metrics_compare.png)

As of 2021, Pigsty's metrics coverage is one of the best among all open source/commercial monitoring systems known to the authors, see [**Cross-Sectional Comparison**] for details).



## Metrics Hierarchy

Pigsty also produces **[Derived Metrics]() based on existing metrics**.

For example, metrics can be aggregated at different levels

![](_media/label-naming.png)

From the raw monitoring time series data to the final chart, there are several processing steps in between.

Here is an example of the derivation process for TPS metrics.

There are four instances in the cluster, and each instance has two databases, so there are a total of 8 DB-level TPS metrics for one instance.

The following chart, on the other hand, is a cross-sectional comparison of QPS for each instance in the entire cluster, so here we use predefined rules to first derive the original transaction counters to obtain 8 DB-level TPS metrics, then aggregate the 8 DB-level time series into 4 instance-level TPS metrics, and finally aggregate these four instance-level TPS metrics into cluster-level TPS metrics at the cluster level.

![](_media/derived-metrics.png)

Pigsty defines a total of 360 classes of derived aggregated metrics, with more to come. The rules for defining derived metrics are described in [**Reference-Derived-Metrics**]()



## Summary

After understanding Pigsty metrics, 
it is useful to understand how Pigsty's [**alerting**](r-alert.md) system uses this metrics data for practical production purposes.

