# service (Ansible role)


### Tasks

[tasks/main.yml](tasks/main.yml)

```yaml
Make sure haproxy is installed	TAGS: [haproxy, haproxy_install, pgsql-init, service]
Create haproxy config directory	TAGS: [haproxy, haproxy_install, pgsql-init, service]
Create haproxy environment file	TAGS: [haproxy, haproxy_install, pgsql-init, service]
Copy haproxy systemd service file	TAGS: [haproxy, haproxy_install, pgsql-init, service]
Fetch postgres cluster memberships	TAGS: [haproxy, haproxy_config, pgsql-init, service]
Templating /etc/haproxy/haproxy.cfg	TAGS: [haproxy, haproxy_config, pgsql-init, service]
Reload Haproxy	TAGS: [haproxy, haproxy_config, haproxy_reload, pgsql-init, service]
Launch haproxy load balancer service	TAGS: [haproxy, haproxy_launch, pgsql-init, service]
Wait for haproxy load balancer online	TAGS: [haproxy, haproxy_launch, pgsql-init, service]
Make sure vip-manager is installed	TAGS: [pgsql-init, service, vip, vip2, vip2_install]
Copy vip-manager systemd service file	TAGS: [pgsql-init, service, vip, vip2, vip2_install]
create vip-manager systemd drop-in dir	TAGS: [pgsql-init, service, vip, vip2, vip2_install]
create vip-manager systemd drop-in file	TAGS: [pgsql-init, service, vip, vip2, vip2_install]
Templating /etc/default/vip-manager.yml	TAGS: [pgsql-init, service, vip, vip2, vip2_config]
Reload vip-manager config	TAGS: [pgsql-init, service, vip, vip2, vip2_reload]
```

### Default variables

[defaults/main.yml](defaults/main.yml)

```yaml
#-----------------------------------------------------------------
# PG_SERVICE
#-----------------------------------------------------------------
pg_services:                     # how to expose postgres service in cluster?
  - name: primary                # service name {{ pg_cluster }}-primary
    src_ip: "*"
    src_port: 5433
    dst_port: pgbouncer          # 5433 route to pgbouncer
    check_url: /primary          # primary health check, success when instance is primary
    selector: "[]"               # select all instance as primary service candidate
  - name: replica                # service name {{ pg_cluster }}-replica
    src_ip: "*"
    src_port: 5434
    dst_port: pgbouncer
    check_url: /read-only        # read-only health check. (including primary)
    selector: "[]"               # select all instance as replica service candidate
    selector_backup: "[? pg_role == `primary` || pg_role == `offline` ]"
  - name: default                # service's actual name is {{ pg_cluster }}-default
    src_ip: "*"                  # service bind ip address, * for all, vip for cluster virtual ip address
    src_port: 5436               # bind port, mandatory
    dst_port: postgres           # target port: postgres|pgbouncer|port_number , pgbouncer(6432) by default
    check_method: http           # health check method: only http is available for now
    check_port: patroni          # health check port:  patroni|pg_exporter|port_number , patroni by default
    check_url: /primary          # health check url path, / as default
    check_code: 200              # health check http code, 200 as default
    selector: "[]"               # instance selector
    haproxy:                     # haproxy specific fields
      maxconn: 3000              # default front-end connection
      balance: roundrobin        # load balance algorithm (roundrobin by default)
      default_server_options: 'inter 3s fastinter 1s downinter 5s rise 3 fall 3 on-marked-down shutdown-sessions slowstart 30s maxconn 3000 maxqueue 128 weight 100'
  - name: offline                # service name {{ pg_cluster }}-offline
    src_ip: "*"
    src_port: 5438
    dst_port: postgres
    check_url: /replica          # offline MUST be a replica
    selector: "[? pg_role == `offline` || pg_offline_query ]"         # instances with pg_role == 'offline' or instance marked with 'pg_offline_query == true'
    selector_backup: "[? pg_role == `replica` && !pg_offline_query]"  # replica are used as backup server in offline service

haproxy_enabled: true             # enable haproxy on this node?
haproxy_reload: true              # reload haproxy after config?
haproxy_admin_auth_enabled: false # enable authentication for haproxy admin?
haproxy_admin_username: admin     # default haproxy admin username
haproxy_admin_password: pigsty    # default haproxy admin password
haproxy_exporter_port: 9101       # default admin/exporter port
haproxy_client_timeout: 24h       # client side connection timeout
haproxy_server_timeout: 24h       # server side connection timeout
vip_mode: none                    # none | l2 | l4 , l4 not implemented yet
vip_reload: true                  # reload vip after config?
vip_address: 127.0.0.1            # virtual ip address ip (l2 or l4)
vip_cidrmask: 24                  # virtual ip address cidr mask (l2 only)
vip_interface: eth0               # virtual ip network interface (l2 only)
dns_mode: vip                     # vip|all|selector: how to resolve cluster DNS?
dns_selector: '[]'                # if dns_mode == vip, filter instances been resolved


#-----------------------------------------------------------------
# PG_IDENTITY (REFERENCE)
#-----------------------------------------------------------------
pg_weight: 100              # default load balance weight (instance level)

#-----------------------------------------------------------------
# DCS (Reference)
#-----------------------------------------------------------------
service_registry: consul              # none | consul | etcd | both
dcs_type: consul                      # none | consul | etcd

#-----------------------------------------------------------------
# PG_BOOTSTRAP (Reference)
#-----------------------------------------------------------------
pg_namespace: /pg
pg_port: 5432
pgbouncer_port: 6432
patroni_port: 8008

#-----------------------------------------------------------------
# EXPORTER (Reference)
#-----------------------------------------------------------------
exporter_metrics_path: /metrics

#-----------------------------------------------------------------
# PG EXPORTER (Reference)
#-----------------------------------------------------------------
pg_exporter_port: 9630
```