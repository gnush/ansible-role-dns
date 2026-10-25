# dns (Bind9) Ansible Role

This role installs and configures **Bind9** as a DNS server and supports **Primary** and **Secondary** operation.

## Configuration

### Variables

- `dns_listen_on_v4`: IPv4 addresses/ports to listen on (default: `["any"]`)
- `dns_listen_on_v6`: IPv6 addresses to listen on (default: `["any"]`)
- `dns_allow_query`: Networks/hosts allowed to query the DNS server (default: `["any"]`)
- `dns_allow_recursion`: Networks/hosts allowed to perform recursive queries
- `dns_forwarders`: List of upstream DNS servers for forwarding queries

- `dns_split_horizon_enabled`: enable split-horizon answers via BIND views (default: `false`)
- `dns_split_records`: records whose answer depends on the client's source network (default: `[]`)

### Split-Horizon

Some records need a different answer depending on where the query comes from, e.g. an
S3 endpoint that is only reachable via its address in the client's own segment:

```yaml
dns_split_horizon_enabled: true
dns_split_records:
  - name: s3.example.internal
    type: A          # default: A
    ttl: 300         # default: 300
    answers:
      "192.168.1.0/24": 192.168.1.100
      "192.168.2.0/24": [192.168.2.100, 192.168.2.101]   # multiple values allowed
```

When enabled, the role:

- creates one view per distinct network in `answers` (named `seg-<cidr>`, most specific
  first) plus a catch-all `default` view. Clients outside all listed networks get the regular
  record from `defaults/zones/`;
- serves each split record as a small override zone in the matching views
  (`/etc/bind/zones/views/<view>/db.<name>`). NS/SOA are taken from the parent zone;
- defines the regular zones in the first view and shares them with all others via `in-view`;
- manages `/etc/bind/named.conf` (with views, the default zones must be inside every view).

Notes:

- Enabled with an empty `dns_split_records` is a noop: the flat config is rendered as if disabled.
- Override zones are rendered as **master on primary and secondary** and are not
  transferred (transfers would need a TSIG key per view). Always deploy to both DNS nodes.
- The override replaces *all* record types at that name within the segment views.
- Every view has its own resolver cache; fine for tens of networks, not hundreds.
- Networks should not overlap; if they do, the more specific one wins.

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
- `config/named.conf.j2` → `/etc/bind/named.conf`
- `config/split.db.j2` → `/etc/bind/zones/views/<view>/db.<name>` (split-horizon only)

