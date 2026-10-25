# dns (Bind9) Ansible Role

This role installs and configures **Bind9** as a DNS server and supports **Primary** and **Secondary** operation.

## Configuration

### Variables

- `dns_listen_on_v4`: IPv4 addresses/ports to listen on (default: `["any"]`)
- `dns_listen_on_v6`: IPv6 addresses to listen on (default: `["any"]`)
- `dns_allow_query`: Networks/hosts allowed to query the DNS server (default: `["any"]`)
- `dns_allow_recursion`: Networks/hosts allowed to perform recursive queries
- `dns_forwarders`: List of upstream DNS servers for forwarding queries

### Zone Configuration

For each managed zone, a YAML file with the following structure must be created in `defaults/zones`:

```yaml
name: "example.internal"
ttl: 3600
soa:
  mname: "ns1.example.internal."
  rname: "hostmaster.example.internal."
  refresh: 3600
  retry: 900
  expire: 604800
  error_ttl: 86400
ns:
  - "ns1.example.internal."
  - "ns2.example.internal."
records:
  - { name: "ns1", class: "IN", type: "A", value: "192.168.0.1" }
  - { name: "ns2", class: "IN", type: "A", value: "192.168.0.2" }
```

> [!NOTE]
> To convert the records of an existing bind9 zone database, the records can be piped through awk:
> 
> ```bash
> awk '{ print "- { name: \"""$1"\", class: \""$2"\", type: \""$3"\", value: \""$4"\" }" }'
> ```
>

## Templates

The configs are located as Jinja2 templates under `config/` and are synchronized to `/etc/bind/`:

- `config/named.conf.options.j2` → `/etc/bind/named.conf.options`
- `config/named.conf.local.j2` → `/etc/bind/named.conf.local`
- `config/zone.db.j2` → `/etc/bind/zones/db.<zone>`

