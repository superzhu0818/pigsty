# environ (Ansible role)

Setup meta node admin user environment

## Create directories
  * `/etc/pigsty` : pigsty resource dir
  * `/etc/pigsty/playbooks` : additional ansible playbooks
  * `/etc/pigsty/dashboards` : grafana dashboards
  * `/etc/pigsty/datasources` : grafana datasources
    
## Setup postgres metadb credential

`~/.pgpass`:

```bash
*:*:*:{{ pg_replication_username }}:{{ pg_replication_password }}
*:*:*:{{ pg_monitor_username }}:{{ pg_monitor_password }}
*:*:*:{{ pg_admin_username }}:{{ pg_admin_password }}
```

`~/.pg_service`:

```ini
[meta]
host=127.0.0.1
port=5432
dbname=meta
user={{ pg_admin_username }}
password={{ pg_admin_password }}
```

## Setup admin user environment

`~/.pigsty` : 

```bash
# set default database for postgres
export PGUSER={{ pg_admin_username }}
export PGSERVICE="meta"
export METADB_URL="service=meta"

# passing environ for application
export GRAFANA_ENDPOINT={{ grafana_endpoint }}
export GRAFANA_USERNAME={{ grafana_admin_username }}
export GRAFANA_PASSWORD={{ grafana_admin_password }}

# export PIGSTY_HOME if exists on home dir
if [[ -d ~/pigsty ]]; then
  export PIGSTY_HOME="${HOME}/pigsty"
fi
if [[ -d /etc/pigsty ]]; then
  export PIGSTY_DASHBOARD_DIR=/etc/pigsty/dashboards
  export PIGSTY_DATASOURCE_DIR=/etc/pigsty/datasource
  export PIGSTY_PLAYBOOK_DIR=/etc/pigsty/playbooks
fi
```

`~/.bashrc` :

```bash
[ -f ~/.pigsty ] && . ~/.pigsty
```

## Setup cmdb




### Tasks

[tasks/main.yml](tasks/main.yml)

```yaml
Create pigsty application dir on admin home	TAGS: [environ, environ_dirs, infra]
Create pigsty resource dirs on /etc/pigsty	TAGS: [environ, environ_dirs, infra]
Get current username	TAGS: [environ, environ_ssh, infra]
Create admin user ssh key pair if not exists	TAGS: [environ, environ_ssh, infra]
Write default user credential to pgpass	TAGS: [environ, environ_pgpass, infra]
Write default meta service to pg_service	TAGS: [environ, environ_pgpass, infra]
Set environment for admin user	TAGS: [environ, environ_vars, infra]
Enable environment for admin user	TAGS: [environ, environ_vars, infra]
```

### Default variables

no default variables (references only)