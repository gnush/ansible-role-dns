# DNS (Bind9) Ansible Role

Installs and configures [Bind9](https://www.isc.org/bind/) as a primary or secondary DNS server.

## Requirements

- Ubuntu >= 22.04  
  Debian >= 12
- Ansible >= 2.11
- Root privileges (`become: true`)
- Inventory group `dns_nodes` with `dns_role` set per host; zone transfers and notifies are derived from it

## Role Variables

Defined in `defaults/main.yml`:

| Variable                    | Default                     | Description                                         |
|-----------------------------|-----------------------------|-----------------------------------------------------|
| `dns_role`                  | `primary`                   | `primary` or `secondary`                            |
| `dns_zones_dir`             | `{{ inventory_dir }}/zones` | Directory with one `*.yml` per zone                 |
| `dns_listen_on_v4`          | `["any"]`                   | IPv4 addresses to listen on                         |
| `dns_listen_on_v6`          | `["any"]`                   | IPv6 addresses to listen on                         |
| `dns_allow_query`           | `["any"]`                   | Clients allowed to query                            |
| `dns_allow_recursion`       | `localnets`, `localhost`    | Clients allowed to query recursively                |
| `dns_forwarders`            | `[]`                        | Upstream resolvers                                  |
| `dns_split_horizon_enabled` | `false`                     | Enable per-network answers, see below               |
| `dns_split_records`         | `[]`                        | Records with per-network answers                    |

## Example Setup

```
inventory/
├── hosts.yml
├── group_vars/
│   └── dns_nodes.yml        # role variables, e.g. dns_split_records
└── zones/                   # dns_zones_dir
    ├── example.internal.yml
    └── 0.168.192.in-addr.arpa.yml
site.yml
```

`site.yml`:

```yaml
- hosts: dns_nodes
  become: true
  roles:
    - dns
```

`inventory/hosts.yml`:

```yaml
dns_nodes:
  hosts:
    dns1:
      ansible_host: 192.168.0.8
      dns_role: primary
    dns2:
      ansible_host: 192.168.0.9
      dns_role: secondary
```

## Zones

Every `*.yml` in `dns_zones_dir` becomes one zone. Seed from `examples/zones/`.

`inventory/zones/example.internal.yml`:

```yaml
name: "example.internal"
ttl: 3600
soa:
  mname: "ns1.example.internal."
  rname: "hostmaster.example.internal."
ns:
  - "ns1.example.internal."
  - "ns2.example.internal."
records:
  - { name: "ns1", class: "IN", type: "A", value: "192.168.0.8" }
```

Records of an existing zone file can be converted with:

```bash
awk '{ print "- { name: \"""$1"\", class: \""$2"\", type: \""$3"\", value: \""$4"\" }" }'
```

## Split-Horizon

Answers a record differently depending on the client's network, e.g. a service reachable
at a different address in each segment. Clients outside all listed networks get the
regular record.

`inventory/group_vars/dns_nodes.yml`:

```yaml
dns_split_horizon_enabled: true
dns_split_records:
  - name: s3.example.internal
    answers:
      "192.168.1.0/24": 192.168.1.100
      "192.168.2.0/24": [192.168.2.100, 192.168.2.101]
```

Each network becomes a BIND view. The override zones are not transferred, so deploy to
primary and secondary. An override replaces all record types at that name.
