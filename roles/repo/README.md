# Repo (ansible role)

This role will bootstrap a local yum repo and download rpm packages


Tasks:

```yaml
Create local repo directory	TAGS: [infra-boot, repo, repo_dir]
Backup & remove existing repos	TAGS: [infra-boot, repo, repo_upstream]
Add required upstream repos	TAGS: [infra-boot, repo, repo_upstream]
Check repo pkgs cache exists	TAGS: [infra-boot, repo, repo_prepare]
Set fact whether repo_exists	TAGS: [infra-boot, repo, repo_prepare]
Move upstream repo to backup	TAGS: [infra-boot, repo, repo_prepare]
Add local file system repos	TAGS: [infra-boot, repo, repo_prepare]
Remake yum cache if not exists	TAGS: [infra-boot, repo, repo_prepare]
Install repo bootstrap packages	TAGS: [infra-boot, repo, repo_boot]
Render repo nginx server files	TAGS: [infra-boot, repo, repo_nginx]
Disable selinux for repo server	TAGS: [infra-boot, repo, repo_nginx]
Launch repo nginx server	TAGS: [infra-boot, repo, repo_nginx]
Waits repo server online	TAGS: [infra-boot, repo, repo_nginx]
Download web url packages	TAGS: [infra-boot, repo, repo_download]
Download repo packages	TAGS: [infra-boot, repo, repo_download]
Download repo pkg dependencies	TAGS: [infra-boot, repo, repo_download]
Create local repo index	TAGS: [infra-boot, repo, repo_download]
Mark repo cache as valid	TAGS: [infra-boot, repo, repo_download]
```

Related variables:

```yaml
#-----------------------------------------------------------------
# REPO
#-----------------------------------------------------------------
repo_enabled: true               # build local repo on this node
repo_name: pigsty                # repo name, pigsty by default
repo_address: pigsty             # external address to this repo (ip:port or url)
repo_port: 80                    # repo listen address, must same as repo_address
repo_home: /www                  # default repo home dir
repo_rebuild: false              # force re-download packages
repo_remove: true                # remove existing upstream repo
repo_upstreams:                  # where to download packages?
  - name: base
    description: CentOS-$releasever - Base
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/ # tuna
      - http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
      - http://mirrors.aliyuncs.com/centos/$releasever/os/$basearch/
      - http://mirrors.cloud.aliyuncs.com/centos/$releasever/os/$basearch/    # aliyun
      - http://mirror.centos.org/centos/$releasever/os/$basearch/             # official
  - name: updates
    description: CentOS-$releasever - Updates
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/ # tuna
      - http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
      - http://mirrors.aliyuncs.com/centos/$releasever/updates/$basearch/
      - http://mirrors.cloud.aliyuncs.com/centos/$releasever/updates/$basearch/    # aliyun
      - http://mirror.centos.org/centos/$releasever/updates/$basearch/             # official
  - name: extras
    description: CentOS-$releasever - Extras
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/ # tuna
      - http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
      - http://mirrors.aliyuncs.com/centos/$releasever/extras/$basearch/
      - http://mirrors.cloud.aliyuncs.com/centos/$releasever/extras/$basearch/    # aliyun
      - http://mirror.centos.org/centos/$releasever/extras/$basearch/             # official
    gpgcheck: no
  - name: epel
    description: CentOS $releasever - epel
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch   # tuna
      - http://mirrors.aliyun.com/epel/$releasever/$basearch              # aliyun
      - http://download.fedoraproject.org/pub/epel/$releasever/$basearch  # official
  - name: grafana
    description: Grafana
    enabled: yes
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/grafana/yum/rpm    # tuna mirror
      - https://packages.grafana.com/oss/rpm                    # official
  - name: prometheus
    description: Prometheus and exporters
    gpgcheck: no
    baseurl: https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch # no other mirrors, quite slow
  - name: pgdg-common
    description: PostgreSQL common RPMs for RHEL/CentOS $releasever - $basearch
    gpgcheck: no
    baseurl:
      - http://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/common/redhat/rhel-$releasever-$basearch  # tuna
      - https://download.postgresql.org/pub/repos/yum/common/redhat/rhel-$releasever-$basearch             # official
  - name: pgdg13
    description: PostgreSQL 13 for RHEL/CentOS $releasever - $basearch
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/13/redhat/rhel-$releasever-$basearch    # tuna
      - https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-$releasever-$basearch                # official
  - name: pgdg14
    description: PostgreSQL 14 for RHEL/CentOS $releasever - $basearch
    gpgcheck: no
    baseurl:
      - https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/14/redhat/rhel-$releasever-$basearch    # tuna
      - https://download.postgresql.org/pub/repos/yum/14/redhat/rhel-$releasever-$basearch                # official
  - name: timescaledb
    description: TimescaleDB for RHEL/CentOS $releasever - $basearch
    gpgcheck: no
    baseurl:
      - https://packagecloud.io/timescale/timescaledb/el/7/$basearch
  - name: centos-sclo
    description: CentOS-$releasever - SCLo
    gpgcheck: no
    baseurl: # mirrorlist: http://mirrorlist.centos.org?arch=$basearch&release=$releasever&repo=sclo-sclo
      - http://mirrors.aliyun.com/centos/$releasever/sclo/$basearch/sclo/
      - http://repo.virtualhosting.hk/centos/$releasever/sclo/$basearch/sclo/
  - name: centos-sclo-rh
    description: CentOS-$releasever - SCLo rh
    gpgcheck: no
    baseurl: # mirrorlist: http://mirrorlist.centos.org?arch=$basearch&release=7&repo=sclo-rh
      - http://mirrors.aliyun.com/centos/$releasever/sclo/$basearch/rh/
      - http://repo.virtualhosting.hk/centos/$releasever/sclo/$basearch/rh/
  - name: nginx
    description: Nginx Official Yum Repo
    skip_if_unavailable: true
    gpgcheck: no
    baseurl: http://nginx.org/packages/centos/$releasever/$basearch/
  - name: harbottle              # for latest consul & kubernetes
    description: Copr repo for main owned by harbottle
    skip_if_unavailable: true
    gpgcheck: no
    baseurl: https://download.copr.fedorainfracloud.org/results/harbottle/main/epel-$releasever-$basearch/

repo_packages:                   # which packages to be included                                                                                  #  what to download #
  - epel-release nginx wget yum-utils yum createrepo sshpass zip unzip                                              # ----  boot   ---- #
  - ntp chrony uuid lz4 bzip2 nc pv jq vim-enhanced make patch bash lsof wget git tuned perf ftp lrzsz rsync        # ----  node   ---- #
  - numactl grubby sysstat dstat iotop bind-utils net-tools tcpdump socat ipvsadm telnet ca-certificates keepalived # ----- utils ----- #
  - readline zlib openssl openssh-clients libyaml libxml2 libxslt libevent perl perl-devel perl-ExtUtils*           # ---  deps:pg  --- #
  - readline-devel zlib-devel uuid-devel libuuid-devel libxml2-devel libxslt-devel openssl-devel libicu-devel       # --- deps:devel -- #
  - ed mlocate parted krb5-devel apr apr-util audit                                                                 # --- deps:gpsql -- #
  - grafana prometheus2 pushgateway alertmanager consul consul_exporter consul-template etcd dnsmasq                # -----  meta ----- #
  - node_exporter postgres_exporter nginx_exporter blackbox_exporter redis_exporter                                 # ---- exporter --- #
  - ansible python python-pip python-psycopg2                                                                       # - ansible & py3 - #
  - python3 python3-psycopg2 python36-requests python3-etcd python3-consul python36-urllib3 python36-idna python36-pyOpenSSL python36-cryptography
  - patroni patroni-consul patroni-etcd pgbouncer pg_cli pgbadger pg_activity tail_n_mail                           # -- pgsql common - #
  - pgcenter boxinfo check_postgres emaj pgbconsole pg_bloat_check pgquarrel barman barman-cli pgloader pgFormatter pitrery pspg pgxnclient PyGreSQL pgadmin4
  - postgresql14* postgis32_14* citus_14* pglogical_14* timescaledb-2-postgresql-14 pg_repack_14 wal2json_14        # -- pg14 packages -#
  - pg_qualstats_14 pg_stat_kcache_14 pg_stat_monitor_14 pg_top_14 pg_track_settings_14 pg_wait_sampling_14
  - pg_statement_rollback_14 system_stats_14 plproxy_14 plsh_14 pldebugger_14 plpgsql_check_14 pgmemcache_14 # plr_14
  - mysql_fdw_14 ogr_fdw_14 tds_fdw_14 sqlite_fdw_14 firebird_fdw_14 hdfs_fdw_14 mongo_fdw_14 osm_fdw_14 pgbouncer_fdw_14
  - hypopg_14 geoip_14 rum_14 hll_14 ip4r_14 prefix_14 pguri_14 tdigest_14 topn_14 periods_14
  - bgw_replstatus_14 count_distinct_14 credcheck_14 ddlx_14 extra_window_functions_14 logerrors_14 mysqlcompat_14 orafce_14
  - repmgr_14 pg_auth_mon_14 pg_auto_failover_14 pg_background_14 pg_bulkload_14 pg_catcheck_14 pg_comparator_14
  - pg_cron_14 pg_fkpart_14 pg_jobmon_14 pg_partman_14 pg_permissions_14 pg_prioritize_14 pgagent_14
  - pgaudit16_14 pgauditlogtofile_14 pgcryptokey_14 pgexportdoc_14 pgfincore_14 pgimportdoc_14 powa_14 pgmp_14 pgq_14
  - pgquarrel-0.7.0-1 pgsql_tweaks_14 pgtap_14 pgtt_14 postgresql-unit_14 postgresql_anonymizer_14 postgresql_faker_14
  - safeupdate_14 semver_14 set_user_14 sslutils_14 table_version_14 # pgrouting_14 osm2pgrouting_14
  - clang coreutils diffutils rpm-build rpm-devel rpmlint rpmdevtools bison flex # gcc gcc-c++                      # - build utils - #

repo_url_packages: [ ]

```