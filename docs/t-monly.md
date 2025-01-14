# Monitoring System Deployment

> How to use Pigsty to monitor existing PostgreSQL instances?

For instances created by Pigsty, all monitoring staff are automatically configured. However, for existing Pigsty instances not created by Pigsty, some manual config is required if you want to monitor them using the Pigsty monitoring system alone.



## TL; DR

1. Setup monitor objects (user,hba,schema,extensions): [Monitor Setup](#monitor-setup)

2. Declare the cluster in the inventory (pigsty.yml or cmdb).

   ```yaml
   pg-test:
     hosts: # Assign unique local ports to each instance
       10.10.10.11: { pg_seq: 1, pg_role: primary , pg_exporter_port: 20001}
       10.10.10.12: { pg_seq: 2, pg_role: replica , pg_exporter_port: 20002}
       10.10.10.13: { pg_seq: 3, pg_role: offline , pg_exporter_port: 20003}
     vars:
       pg_cluster: pg-test # Fill in the cluster name
       pg_version: 14 # Fill in the database major version
       pg_databases: [{ name: test }] # fill in the list of databases (each database object as an array element)
       
   # Provide monitor user & password @ global/cluster/instance leve:  pg_monitor_username/pg_monitor_password
   ```

3. Execute the playbook on that cluster: `. /pgsql-monly.yml -l pg-test`




## Monitoring Deployment Overview

If you want to use only the **monitoring system** part of Pigsty, for example, if you want to use the Pigsty monitoring system to monitor existing PostgreSQL instances, then you can use the **monitor only deployment** mode. In monitor only mode, you can use Pigsty to manage and monitor other PostgreSQL instances (currently 10+ versions are supported by default, older versions can be supported by manually modifying the pg_exporter configuration file)

First, you need to complete the standard Pigsty installation process on one (or more) management nodes, and then you can add more database instances to the monitoring. Depending on the access rights of the target database node, there are two further cases.



**If the target node can be managed**

**If target DB nodes can be managed by Pigsty** (ssh accessible, sudo available), then you can use the `monitor` task in the [`pgsql.yml`](p-pgsql.md) script to deploy the monitoring components on the target node in the same way: PG Exporter, Node Exporter. You can also use the other tasks of the script to deploy additional components and their monitoring on the existing instance nodes: connection pool Pgbouncer and load balancer HAProxy, log collection Agent, to get the exact same experience as the native Pigsty database instance.

Since the target database cluster already exists, you need to refer to this section to manually [create monitoring users, schemas and extensions] on the target database cluster (# Monitoring Objects Configuration). The rest of the process is no different from a full deployment.



**If only database connection string**

If you **can only access the target database by means of a PGURL** (database connection string), consider using **Monitor Only mode/Lite mode** (Monitor Only: Monly) to monitor the target database. In this mode, all monitoring components are deployed on the management node where Pigsty is installed. The monitoring system does not have metrics related to nodes, connection pools, or load balancers, but the database itself, as well as real-time status information in the Data Catalog, remains available.

To perform a streamlined monitoring deployment, you will also need to refer to this section to manually [create monitoring users, schema and extensions] on the target database cluster (#monitoring object configuration) and ensure that you can access the target database from the management node using monitoring users. After that, execute the `pgsql-monly.yml` script against the target cluster to complete the deployment.

**This article focuses on this monitoring deployment model**.



![](../_media/monly.svg)

> Figure: Schematic diagram of the monitoring-only mode architecture, with multiple PG Exporter deployed locally on the management machine to monitor multiple remote database instances.





## Difference between lean and standard deployments

The Pigsty monitoring system consists of three core modules.

* PGCAT: Information presentation based on database data dictionary, presenting current database status information, but unable to retrace historical data.
* PGSQL: monitoring based on metrics data, which is the main part of Pigsty monitoring with database/connection pool/machine node/load balancer related metrics collected and presented by Prometheus.
* PGLOG: Database logging, providing real-time log access capability, with additional monitoring metrics. Based on Grafana Loki / Promtail, optional item.

| Matters \ Levels | L1 | L2 | L3 |
| :------------: | :-------------------: | :--------------------: | :--------------------: |
| name | basic | managed | full |
| English | basic | managed | full |
| Scenario | Only connection string | DB already exists, node is manageable | Instance created by Pigsty |
| PGCAT | ✅ Full available | ✅ Full available | ✅ Full available |
| PGSQL | ✅ Limited to PG metrics | ✅ Limited to PG and node metrics | ✅ Full functionality |
| Pgbouncer Metrics | ❌ Not Available | ⚠️ Optional | ✅ Pre-installed items |
| HAProxy Metrics | ❌ Not available | ⚠️ Optional | ✅ Pre-installed items |
| PGLOG | ❌ Not available | ⚠️ Optional | ⚠️ Optional |
| PG Exporter | ⚠️ Deploy on meta | ✅ Deploy on DB Node | ✅ Deployono DB Node |
| Node Exporter | ❌ Not deployed | ✅ Deploy on DB Node | ✅ Deployono DB Node |
| Intrusion into DB node | ✅ intact | ⚠️ install exporter | ⚠️ managed by Pigsty |
| Monitor existing instances | ✅ OK | ✅ OK | ❌ Only for Pigsty-Managed instances |
| Monitor users and views | Manually created | Manually created | Automatically created by Pigsty |
| Deployment Usage Script | `pgsql-monly.yml` | `pgsql.yml -t monitor` | `pgsql.yml -t monitor` |
| Required Permissions | Manage node reachable PGURLs | ssh & sudo | ssh & sudo |
| Features Overview | Basic Features: PGCAT+PGSQL | Most Features | Full Features |









## Monitoring Existing Instances: Monly Mode

Deploying monitoring system for a database instance is divided into three steps: 

* [Ensure Accessibility](#ensure-accessibility)
* [Edit-Config](#edit-config)
* [Execute Playbook](#execute-playbook)



### Ensure Accessibility

In order to include an external existing PostgreSQL instance in monitoring, you need to have a connection string that can be used to access that instance/cluster. Any reachable connection string (business user, superuser) can be used, but we recommend using a dedicated monitoring user to avoid privilege leaks.

- [ ] Monitor user: The default user name used is `dbuser_monitor`, which needs to belong to the `pg_monitor` role group or ensure that it has access to the relevant view.
- [ ] Monitor Authentication: Default is password access, you need to make sure the HBA policy allows the monitor user to access the database locally from the management machine or DB node.
- [ ] Monitor Mode: Fixed using the name `monitor` for installing additional **monitor views** with extended plugins, not mandatory but highly recommended to create.
- [ ] Monitoring extensions: It is highly recommended to enable the monitoring extension `pg_stat_statements` that comes with PG

For details on the preparation of monitoring objects, please refer to the section after the article: [Monitor Setup](#monitor-setup).




### Edit Config

As with deploying a brand new Pigsty instance, you need to declare this target cluster in the configuration manifest (configuration file or CMDB). For example, specify [identity] for the cluster and instance (c-config.md#identity parameter). The difference is that you also need to manually assign a unique local port number (`pg_exporter_port`) to each instance at the **instance level**.

The following is a sample database cluster declaration.

```yaml
pg-test:
  hosts: # Assign a unique local port to each instance
    10.10.10.11: { pg_seq: 1, pg_role: primary , pg_exporter_port: 20001}
    10.10.10.12: { pg_seq: 2, pg_role: replica , pg_exporter_port: 20002}
    10.10.10.13: { pg_seq: 3, pg_role: offline , pg_exporter_port: 20003}
  vars:
    pg_cluster: pg-test # Fill in the cluster name
    pg_version: 14 # Fill in the database major version
    pg_databases: [{ name: test }] # fill in the list of databases (each database object as an array element)
    
# Provide monitor user password in global/cluster/instance configuration pg_monitor_username/pg_monitor_password
```

> Note that even if you access the database via a domain name, you still need to declare the database cluster by filling in the actual IP address.

To enable the PGCAT feature, you need to explicitly list the list of database names of the target cluster in `pg_databases`. The databases in this list will be registered as Grafana's data source and you can access the Catalog data of this instance directly through Grafana. If you do not want to use PGCAT-related functions, you can leave this variable unset or set it to an empty array.



#### Connection Information

Note: Pigsty will generate the monitored connection string by default using the following rules. However, if the parameter `pg_exporter_url` is present, it will override the spliced connection string directly.

```bash
postgres://{{ pg_monitor_username }}:{{ pg_monitor_password }}@{{ inventory_hostname }}:{{ pg_port }}/postgres?sslmode=disable
```

You can use a uniform monitor user/password setting globally, or configure the following connection parameters on-demand at the **cluster level** or **instance level** as appropriate

```yaml
pg_monitor_username: dbuser_monitor # Monitor username, not required if using global configuration
pg_monitor_password: DBUser.Monitor # Monitor the user's password, you don't need to configure it here if you use the global configuration
pg_port: 5432 # If you use a non-standard database port, change it here
```

<details><summary>Example: Specifying connection information at the instance level</summary

```yaml
pg-test:
  hosts: # Specify access URLs directly for the instance
    10.10.10.11: 
      pg_seq: 1
      pg_role: primary
      pg_exporter_port: 20001
      pg_monitor_username: monitor_user1
      pg_monitor_password: monitor_pass1
    10.10.10.12: 
      pg_seq: 2
      pg_role: replica
      pg_exporter_port: 20002 # Specify pg_exporter_url directly
      pg_exporter_url: 'postgres://someuser:pass@rds.pg.hongkong.xxx:5432/postgres?sslmode=disable''
    10.10.10.13: 
      pg_seq: 3
      pg_role: offline
      pg_exporter_port: 20003
      pg_monitor_username: monitor_user3
      pg_monitor_password: monitor_pass3
  vars:
    pg_cluster: pg-test # Fill in the cluster name
    pg_version: 14 # Fill in the database major version
    pg_databases: [{ name: test }] # fill in the list of databases (each database object as an array element)
```

</details>





### Execute Playbook

Once the cluster declaration is complete, incorporating it into monitoring is as simple as using the script ``pgsql-monitor.yml`` on the management node for the target cluster

```bash
. /pgsql-monitor.yml -l <cluster> 
```





---------------------





## Monitor object configuration

### Monitor user

Using the default monitoring user ``dbuser_monitor`` used by Pigsty as an example, create the following user on the target database cluster.

```sql
CREATE USER dbuser_monitor;
GRANT pg_monitor TO dbuser_monitor;
COMMENT ON ROLE dbuser_monitor IS 'system monitor user';
ALTER USER dbuser_monitor SET log_min_duration_statement = 1000;
ALTER USER dbuser_monitor PASSWORD 'DBUser.Password'; -- Change monitor user password as needed
```

Configure the database `pg_hba.conf` file by adding the following rules to allow monitoring users to access the database from local, and administrative machines using passwords.

```ini
# allow local role monitor with password
local all dbuser_monitor md5
host all dbuser_monitor 127.0.0.1/32 md5
host all dbuser_monitor <management machine IP address>/32 md5
```

### Monitor mode

Monitoring mode and extensions are **optional** and the body of the Pigsty monitoring system will work fine even without them, but we strongly recommend creating monitoring mode and enabling at least the `pg_stat_statements` that comes with the official PG, which provides important data about query performance. Note: This extension must be included in the database parameter `shared_preload_libraries` to take effect, and modifying this parameter requires a database restart.

To create the extension schema.

```sql
CREATE SCHEMA IF NOT EXISTS monitor; -- Create a monitor-specific schema
GRANT USAGE ON SCHEMA monitor TO dbuser_monitor; -- Allow monitor users to use
```

### Monitor extensions

Create extension plugins.

```sql
-- It is highly recommended to enable the pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements" WITH SCHEMA "monitor"; 

-- optional other extensions
CREATE EXTENSION IF NOT EXISTS "pgstattuple" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pg_qualstats" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pg_buffercache" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pageinspect" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pg_prewarm" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pg_visibility" WITH SCHEMA "monitor";
CREATE EXTENSION IF NOT EXISTS "pg_freespacemap" WITH SCHEMA "monitor";
```

### Monitoring Views

The monitoring view provides several common pre-processing results and wraps permissions for certain monitoring metrics that require high privileges (e.g. shared memory allocation) for easy query and use. It is highly recommended to create



<details><summary>Monitoring Views</summary>

```sql
--==================================================================--
--                            Monitor Schema                        --
--==================================================================--

----------------------------------------------------------------------
-- cleanse
----------------------------------------------------------------------
CREATE SCHEMA IF NOT EXISTS monitor;
GRANT USAGE ON SCHEMA monitor TO dbuser_monitor;
GRANT USAGE ON SCHEMA monitor TO "{{ pg_admin_username }}";
GRANT USAGE ON SCHEMA monitor TO "{{ pg_replication_username }}";

--==================================================================--
--                            Monitor Views                         --
--==================================================================--

----------------------------------------------------------------------
-- Table bloat estimate : monitor.pg_table_bloat
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_table_bloat CASCADE;
CREATE OR REPLACE VIEW monitor.pg_table_bloat AS
SELECT CURRENT_CATALOG AS datname, nspname, relname , tblid , bs * tblpages AS size,
       CASE WHEN tblpages - est_tblpages_ff > 0 THEN (tblpages - est_tblpages_ff)/tblpages::FLOAT ELSE 0 END AS ratio
FROM (
         SELECT ceil( reltuples / ( (bs-page_hdr)*fillfactor/(tpl_size*100) ) ) + ceil( toasttuples / 4 ) AS est_tblpages_ff,
                tblpages, fillfactor, bs, tblid, nspname, relname, is_na
         FROM (
                  SELECT
                      ( 4 + tpl_hdr_size + tpl_data_size + (2 * ma)
                          - CASE WHEN tpl_hdr_size % ma = 0 THEN ma ELSE tpl_hdr_size % ma END
                          - CASE WHEN ceil(tpl_data_size)::INT % ma = 0 THEN ma ELSE ceil(tpl_data_size)::INT % ma END
                          ) AS tpl_size, (heappages + toastpages) AS tblpages, heappages,
                      toastpages, reltuples, toasttuples, bs, page_hdr, tblid, nspname, relname, fillfactor, is_na
                  FROM (
                           SELECT
                               tbl.oid AS tblid, ns.nspname , tbl.relname, tbl.reltuples,
                               tbl.relpages AS heappages, coalesce(toast.relpages, 0) AS toastpages,
                               coalesce(toast.reltuples, 0) AS toasttuples,
                               coalesce(substring(array_to_string(tbl.reloptions, ' ') FROM 'fillfactor=([0-9]+)')::smallint, 100) AS fillfactor,
                               current_setting('block_size')::numeric AS bs,
                               CASE WHEN version()~'mingw32' OR version()~'64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END AS ma,
                               24 AS page_hdr,
                               23 + CASE WHEN MAX(coalesce(s.null_frac,0)) > 0 THEN ( 7 + count(s.attname) ) / 8 ELSE 0::int END
                                   + CASE WHEN bool_or(att.attname = 'oid' and att.attnum < 0) THEN 4 ELSE 0 END AS tpl_hdr_size,
                               sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 0) ) AS tpl_data_size,
                               bool_or(att.atttypid = 'pg_catalog.name'::regtype)
                                   OR sum(CASE WHEN att.attnum > 0 THEN 1 ELSE 0 END) <> count(s.attname) AS is_na
                           FROM pg_attribute AS att
                                    JOIN pg_class AS tbl ON att.attrelid = tbl.oid
                                    JOIN pg_namespace AS ns ON ns.oid = tbl.relnamespace
                                    LEFT JOIN pg_stats AS s ON s.schemaname=ns.nspname AND s.tablename = tbl.relname AND s.inherited=false AND s.attname=att.attname
                                    LEFT JOIN pg_class AS toast ON tbl.reltoastrelid = toast.oid
                           WHERE NOT att.attisdropped AND tbl.relkind = 'r' AND nspname NOT IN ('pg_catalog','information_schema')
                           GROUP BY 1,2,3,4,5,6,7,8,9,10
                       ) AS s
              ) AS s2
     ) AS s3
WHERE NOT is_na;
COMMENT ON VIEW monitor.pg_table_bloat IS 'postgres table bloat estimate';

----------------------------------------------------------------------
-- Index bloat estimate : monitor.pg_index_bloat
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_index_bloat CASCADE;
CREATE OR REPLACE VIEW monitor.pg_index_bloat AS
SELECT CURRENT_CATALOG AS datname, nspname, idxname AS relname, tblid, idxid, relpages::BIGINT * bs AS size,
       COALESCE((relpages - ( reltuples * (6 + ma - (CASE WHEN index_tuple_hdr % ma = 0 THEN ma ELSE index_tuple_hdr % ma END)
                                               + nulldatawidth + ma - (CASE WHEN nulldatawidth % ma = 0 THEN ma ELSE nulldatawidth % ma END))
                                  / (bs - pagehdr)::FLOAT  + 1 )), 0) / relpages::FLOAT AS ratio
FROM (
         SELECT nspname,idxname,indrelid AS tblid,indexrelid AS idxid,
                reltuples,relpages,
                current_setting('block_size')::INTEGER                                                               AS bs,
                (CASE WHEN version() ~ 'mingw32' OR version() ~ '64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END)  AS ma,
                24                                                                                                   AS pagehdr,
                (CASE WHEN max(COALESCE(pg_stats.null_frac, 0)) = 0 THEN 2 ELSE 6 END)                               AS index_tuple_hdr,
                sum((1.0 - COALESCE(pg_stats.null_frac, 0.0)) *
                    COALESCE(pg_stats.avg_width, 1024))::INTEGER                                                     AS nulldatawidth
         FROM pg_attribute
                  JOIN (
             SELECT pg_namespace.nspname,
                    ic.relname                                                   AS idxname,
                    ic.reltuples,
                    ic.relpages,
                    pg_index.indrelid,
                    pg_index.indexrelid,
                    tc.relname                                                   AS tablename,
                    regexp_split_to_table(pg_index.indkey::TEXT, ' ') :: INTEGER AS attnum,
                    pg_index.indexrelid                                          AS index_oid
             FROM pg_index
                      JOIN pg_class ic ON pg_index.indexrelid = ic.oid
                      JOIN pg_class tc ON pg_index.indrelid = tc.oid
                      JOIN pg_namespace ON pg_namespace.oid = ic.relnamespace
                      JOIN pg_am ON ic.relam = pg_am.oid
             WHERE pg_am.amname = 'btree' AND ic.relpages > 0 AND nspname NOT IN ('pg_catalog', 'information_schema')
         ) ind_atts ON pg_attribute.attrelid = ind_atts.indexrelid AND pg_attribute.attnum = ind_atts.attnum
                  JOIN pg_stats ON pg_stats.schemaname = ind_atts.nspname
             AND ((pg_stats.tablename = ind_atts.tablename AND pg_stats.attname = pg_get_indexdef(pg_attribute.attrelid, pg_attribute.attnum, TRUE))
                 OR (pg_stats.tablename = ind_atts.idxname AND pg_stats.attname = pg_attribute.attname))
         WHERE pg_attribute.attnum > 0
         GROUP BY 1, 2, 3, 4, 5, 6
     ) est;
COMMENT ON VIEW monitor.pg_index_bloat IS 'postgres index bloat estimate (btree-only)';


----------------------------------------------------------------------
-- Relation Bloat : monitor.pg_bloat
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_bloat CASCADE;
CREATE OR REPLACE VIEW monitor.pg_bloat AS
SELECT coalesce(ib.datname, tb.datname)                                                   AS datname,
       coalesce(ib.nspname, tb.nspname)                                                   AS nspname,
       coalesce(ib.tblid, tb.tblid)                                                       AS tblid,
       coalesce(tb.nspname || '.' || tb.relname, ib.nspname || '.' || ib.tblid::RegClass) AS tblname,
       tb.size                                                                            AS tbl_size,
       CASE WHEN tb.ratio < 0 THEN 0 ELSE round(tb.ratio::NUMERIC, 6) END                 AS tbl_ratio,
       (tb.size * (CASE WHEN tb.ratio < 0 THEN 0 ELSE tb.ratio::NUMERIC END)) ::BIGINT    AS tbl_wasted,
       ib.idxid,
       ib.nspname || '.' || ib.relname                                                    AS idxname,
       ib.size                                                                            AS idx_size,
       CASE WHEN ib.ratio < 0 THEN 0 ELSE round(ib.ratio::NUMERIC, 5) END                 AS idx_ratio,
       (ib.size * (CASE WHEN ib.ratio < 0 THEN 0 ELSE ib.ratio::NUMERIC END)) ::BIGINT    AS idx_wasted
FROM monitor.pg_index_bloat ib
         FULL OUTER JOIN monitor.pg_table_bloat tb ON ib.tblid = tb.tblid;

COMMENT ON VIEW monitor.pg_bloat IS 'postgres relation bloat detail';


----------------------------------------------------------------------
-- monitor.pg_index_bloat_human
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_index_bloat_human CASCADE;
CREATE OR REPLACE VIEW monitor.pg_index_bloat_human AS
SELECT idxname                            AS name,
       tblname,
       idx_wasted                         AS wasted,
       pg_size_pretty(idx_size)           AS idx_size,
       round(100 * idx_ratio::NUMERIC, 2) AS idx_ratio,
       pg_size_pretty(idx_wasted)         AS idx_wasted,
       pg_size_pretty(tbl_size)           AS tbl_size,
       round(100 * tbl_ratio::NUMERIC, 2) AS tbl_ratio,
       pg_size_pretty(tbl_wasted)         AS tbl_wasted
FROM monitor.pg_bloat
WHERE idxname IS NOT NULL;
COMMENT ON VIEW monitor.pg_index_bloat_human IS 'postgres index bloat info in human-readable format';

----------------------------------------------------------------------
-- monitor.pg_table_bloat_human
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_table_bloat_human CASCADE;
CREATE OR REPLACE VIEW monitor.pg_table_bloat_human AS
SELECT tblname                                          AS name,
       idx_wasted + tbl_wasted                          AS wasted,
       pg_size_pretty(idx_wasted + tbl_wasted)          AS all_wasted,
       pg_size_pretty(tbl_wasted)                       AS tbl_wasted,
       pg_size_pretty(tbl_size)                         AS tbl_size,
       tbl_ratio,
       pg_size_pretty(idx_wasted)                       AS idx_wasted,
       pg_size_pretty(idx_size)                         AS idx_size,
       round(idx_wasted::NUMERIC * 100.0 / idx_size, 2) AS idx_ratio
FROM (SELECT datname,
             nspname,
             tblname,
             coalesce(max(tbl_wasted), 0)                         AS tbl_wasted,
             coalesce(max(tbl_size), 1)                           AS tbl_size,
             round(100 * coalesce(max(tbl_ratio), 0)::NUMERIC, 2) AS tbl_ratio,
             coalesce(sum(idx_wasted), 0)                         AS idx_wasted,
             coalesce(sum(idx_size), 1)                           AS idx_size
      FROM monitor.pg_bloat
      WHERE tblname IS NOT NULL
      GROUP BY 1, 2, 3
     ) d;
COMMENT ON VIEW monitor.pg_table_bloat_human IS 'postgres table bloat info in human-readable format';



----------------------------------------------------------------------
-- Activity Overview: monitor.pg_session
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_session CASCADE;
CREATE OR REPLACE VIEW monitor.pg_session AS
SELECT coalesce(datname, 'all') AS datname, numbackends, active, idle, ixact, max_duration, max_tx_duration, max_conn_duration
FROM (
         SELECT datname,
                count(*)                                         AS numbackends,
                count(*) FILTER ( WHERE state = 'active' )       AS active,
                count(*) FILTER ( WHERE state = 'idle' )         AS idle,
                count(*) FILTER ( WHERE state = 'idle in transaction'
                    OR state = 'idle in transaction (aborted)' ) AS ixact,
                max(extract(epoch from now() - state_change))
                FILTER ( WHERE state = 'active' )                AS max_duration,
                max(extract(epoch from now() - xact_start))      AS max_tx_duration,
                max(extract(epoch from now() - backend_start))   AS max_conn_duration
         FROM pg_stat_activity
         WHERE backend_type = 'client backend'
           AND pid <> pg_backend_pid()
         GROUP BY ROLLUP (1)
         ORDER BY 1 NULLS FIRST
     ) t;
COMMENT ON VIEW monitor.pg_session IS 'postgres activity group by session';


----------------------------------------------------------------------
-- Sequential Scan: monitor.pg_seq_scan
----------------------------------------------------------------------
DROP VIEW IF EXISTS monitor.pg_seq_scan CASCADE;
CREATE OR REPLACE VIEW monitor.pg_seq_scan AS
    SELECT schemaname                                                        AS nspname,
           relname,
           seq_scan,
           seq_tup_read,
           seq_tup_read / seq_scan                                           AS seq_tup_avg,
           idx_scan,
           n_live_tup + n_dead_tup                                           AS tuples,
           round(n_live_tup * 100.0::NUMERIC / (n_live_tup + n_dead_tup), 2) AS live_ratio
    FROM pg_stat_user_tables
    WHERE seq_scan > 0
      and (n_live_tup + n_dead_tup) > 0
    ORDER BY seq_scan DESC;
COMMENT ON VIEW monitor.pg_seq_scan IS 'table that have seq scan';

```



if PostgreSQL Major version >= 13 , this view can be created to inspect shared memory usage

```sql
DROP FUNCTION IF EXISTS monitor.pg_shmem() CASCADE;
CREATE OR REPLACE FUNCTION monitor.pg_shmem() RETURNS SETOF
    pg_shmem_allocations AS $$ SELECT * FROM pg_shmem_allocations;$$ LANGUAGE SQL SECURITY DEFINER;
COMMENT ON FUNCTION monitor.pg_shmem() IS 'security wrapper for pg_shmem';
```



</details>