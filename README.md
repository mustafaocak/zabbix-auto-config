# About

Zabbix-auto-config is an utility that aims to automatically configure hosts, host groups and templates in the monitoring software [Zabbix](https://www.zabbix.com/).

Note: This is only tested with Zabbix 4.0 LTS.

# Quick start

This is a crash course in how to quickly get this application up and running in a local test environment:

## Containers

Use podman/docker.

```bash
podman run -d -p 80:80 zabbix/zabbix-appliance:ubuntu-4.0-latest
podman run -d --name postgres -p 5432:5432 -e POSTGRES_USER=username -e POSTGRES_DB=mydatabase -e POSTGRES_PASSWORD=secret library/postgres:latest
```

## Zabbix web

It is currently assumed that you have the following hostgroups in Zabbix. You should logon to Zabbix and create them:

* All-auto-disabled-hosts
* All-hosts

For automatic linking in templates you could create the templates:

* Template-barry
* Template-pizza

## Database

```bash
PGPASSWORD=secret psql -h localhost -U postgres -p 5432 -U username mydatabase << EOF
CREATE TABLE hosts (
    data jsonb
);
CREATE TABLE hosts_source (
    data jsonb
);
EOF
```

## Application

```bash
python3 -m venv venv
. venv/bin/activate
pip install -e .
cat config.sample.ini | perl -pe 's/^dryrun = true$/dryrun = false/g' > config.ini
mkdir -p /path/to/source_collector_dir/ /path/to/host_modifier_dir/ /path/to/map_dir/
cat > /path/to/source_collector_dir/mysource.py << EOF
HOSTS = [
    {
        "hostname": "foo.example.com",
    },
    {
        "hostname": "bar.example.com",
    },
]

def collect(*args, **kwargs):
    for host in HOSTS:
        host["enabled"] = True
        host["siteadmins"] = ["bob@example.com"]
        host["properties"] = ["pizza"]
        source = kwargs.get("source")
        if source:
            host["properties"].append(source)

    return HOSTS
EOF
cat > /path/to/host_modifier_dir/mod.py << EOF
def modify(host):
    if host["hostname"] == "bar.example.com":
        host["properties"].append("barry")
        host["properties"] = sorted(host["properties"])
    return host
EOF
cat > /path/to/map_dir/property_template_map.txt << EOF
pizza:Template-pizza
barry:Template-barry
EOF
cat > /path/to/map_dir/property_hostgroup_map.txt << EOF
other:Hostgroup-other-hosts
EOF
cat > /path/to/map_dir/siteadmin_hostgroup_map.txt << EOF
bob@example.com:Hostgroup-bob-hosts
EOF
```

Run the application:

```bash
zac
```

## Systemd unit

You could run this as a systemd service:

```ini
[Unit]
Description=Zabbix auto config
After=network.target

[Service]
User=zabbix
Group=zabbix
WorkingDirectory=/home/zabbix/zabbix-auto-config
Environment=PATH=/home/zabbix/zabbix-auto-config/venv/bin
ExecStart=/home/zabbix/zabbix-auto-config/venv/bin/zac
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```
