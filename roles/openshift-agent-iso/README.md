# openshift-agent-iso

Generate an OpenShift **Agent-based Installer** ISO from a release image on a
disconnected mirror (extract `openshift-install`, render configs, create image).

## Purpose

1. Extract `openshift-install` from the mirrored release image with
   `oc adm release extract`.
2. Render `install-config.yaml` (baseDomain, networks, VIPs, pullSecret, sshKey).
3. Render `agent-config.yaml` for SNO or Compact (NMState networking, root
   device hints, DNS, routes).
4. Run `openshift-install agent create image`.

Playbook:

| Playbook | Inventory group | Task file |
|----------|-----------------|-----------|
| `playbooks/openshift-agent-create-iso.yaml` | `openshift_agent_iso` | `create-iso` |

## Requirements

- Ansible 2.14 or later
- `community.crypto` collection (`get_certificate` for the mirror TLS trust bundle)
- `oc` on the target host (system path or `openshift_oc_bin`)
- Pull secret that can pull the mirrored release image
- Network access from the target host to the registry in
  `openshift_release_image` / `openshift_mirror_registry`

## Role layout

```
roles/openshift-agent-iso/
├── defaults/main.yml
├── vars/main.yaml              # topology → replicas / platform
├── tasks/
│   ├── create-iso.yaml             # validate + orchestrate
│   ├── extract-installer.yaml      # oc adm release extract
│   ├── prepare-mirror-config.yaml  # IDMS + mirror CA for install-config
│   ├── render-configs.yaml         # install-config + agent-config
│   └── create-image.yaml           # openshift-install agent create image
└── templates/
    ├── install-config.yaml.j2
    └── agent-config.yaml.j2
```

## Quick start

```bash
ansible-playbook playbooks/openshift-agent-create-iso.yaml \
  -i inventory.yaml \
  -e @vault/quay_secrets.yaml \
  -e openshift_ssh_key="$(cat ~/.ssh/id_ed25519.pub)" \
  --vault-password-file /home/nreilly/tmp/.vault/.vault_password
```

Persistent `install-config.yaml` / `agent-config.yaml` stay in
`{{ openshift_agent_iso_workdir }}/`. Copies are placed under `install/` for
`openshift-install agent create image --dir=.../install` (which consumes them).

## Variables

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `openshift_agent_iso_workdir` | `/opt/openshift-agent-iso` | Binary, auth, persistent configs |
| `openshift_agent_iso_install_dir` | `…/install` | Consumed config copies + generated ISO |
| `openshift_version` | `4.22.4` | Used in default release image tag |
| `openshift_release_image` | `registry-int…/ocp-release:{{ version }}-x86_64` | Source for extract |
| `openshift_oc_bin` | `oc` | Path to `oc` |
| `openshift_topology` | auto | Detected from `openshift_hosts` count |
| `openshift_cluster_name` | `sno` | Cluster name in install-config |
| `openshift_base_domain` | `lab.naramajac.xyz` | baseDomain |
| `openshift_pull_secret` | `{{ redhat_pull_secret }}` | Seed RH secret; install-config gets mirror-only auth |
| `openshift_ssh_key` | `""` | Public SSH key (required) |
| `openshift_machine_network` | `[{cidr: 172.16.0.0/24}]` | machineNetwork |
| `openshift_api_vips` | `[]` | Required for compact/ha |
| `openshift_ingress_vips` | `[]` | Required for compact/ha |
| `openshift_hosts` | `[]` | Host list for agent-config |
| `openshift_rendezvous_ip` | `""` | Defaults to first host IPv4 |
| `openshift_mirror_registry` | `registry-int…:8443` | Internal mirror host:port |
| `openshift_oc_mirror_cluster_resources` | `…/working-dir/cluster-resources` | oc-mirror IDMS source dir |
| `openshift_image_digest_sources` | auto / defaults | Written to install-config |
| `openshift_additional_trust_bundle` | auto from TLS port | Mirror cert PEM (optional override) |
| `force_openshift_install_extract` | `false` | Re-extract installer binary |

### Disconnected mirror (required)

`install-config.yaml` always includes `imageDigestSources` and
`additionalTrustBundle`:

1. **imageDigestSources** — loaded from oc-mirror `idms*.yaml` under
   `openshift_oc_mirror_cluster_resources` when present; otherwise
   `openshift_image_digest_sources_default` (release + `registry.redhat.io`
   → `openshift_mirror_registry`).
2. **additionalTrustBundle** — from `openshift_additional_trust_bundle` if set;
   else fetched from `openshift_mirror_registry` with
   `community.crypto.get_certificate` (leaf + chain when available).

Requires the `community.crypto` collection on the controller/target Python
environment. Set `openshift_image_digest_sources_force: true` to skip IDMS
auto-load and use an explicit `openshift_image_digest_sources` list.

### Topology detection

Topology is derived from `openshift_hosts | length` (do not set
`openshift_topology` manually):

| Hosts | Topology | Expected roles | Platform |
|-------|----------|----------------|----------|
| 1 | `sno` | 1 master | `none` |
| 3 | `compact` | 3 masters | `baremetal` |
| 5+ | `ha` | 3 masters + remaining workers | `baremetal` |
| 2 or 4 | **invalid** | — | role fails |

HA `compute` replicas are set from the number of `role: worker` hosts.

## Host networking (`openshift_hosts`)

Each host:

```yaml
- hostname: master-0.lab.naramajac.xyz
  role: master                    # master | worker
  root_device_name: /dev/vda      # or /dev/sda
  interfaces:
    - name: enp1s0
      mac_address: "52:54:00:00:00:10"
  network_config:                 # optional; NMState under the hood
    ipv4:
      interface: enp1s0           # interface (or bond) that gets the address
      address: 172.16.0.10
      prefix_length: 24
      # dhcp: false
    dns_resolver:
      - 172.16.0.254
    routes:
      - destination: 0.0.0.0/0
        next_hop_address: 172.16.0.1
        next_hop_interface: enp1s0
```

### Multi-NIC (IP on one interface)

List every NIC under `interfaces`. Set `ipv4.interface` to the one that should
receive the address; others are brought up with IPv4 disabled.

### Bonds and VLANs

Use `bonds` (list) and optional `vlans`. Put the node IP on a bond or VLAN via
`ipv4.interface` (see `group_vars/openshift_agent_iso.yml` for a full compact
example with 4 NICs, 2 bonds, and `bond0.100`).

```yaml
network_config:
  ipv4_addresses:
    - interface: bond0.100
      address: 172.16.1.226
      prefix_length: 24
    - interface: bond1
      address: 10.254.224.226
      prefix_length: 24
  bonds:
    - name: bond0
      mode: active-backup
      ports: [eno1, eno2]
      primary: eno1
    - name: bond1
      mode: 802.3ad
      ports: [eno3, eno4]
  vlans:
    - name: bond0.100
      id: 100
      base_iface: bond0
  dns_resolver:
    - 172.16.1.253
    - 10.254.224.253
  routes:
    - destination: "0.0.0.0/0"
      next_hop_address: 172.16.1.254
      next_hop_interface: bond0.100
      table_id: 100
    - destination: "0.0.0.0/0"
      next_hop_address: 10.254.224.254
      next_hop_interface: bond1
      table_id: 101
  route_rules:
    - ip_from: 172.16.1.226/32
      route_table: 100
      priority: 1000
    - ip_from: 10.254.224.226/32
      route_table: 101
      priority: 1001
```

Dual default routes need `table_id` + `route_rules` (policy routing) so
traffic sourced from each address egresses the matching interface. Without that,
NetworkManager keeps only one useful default in the main table.

Singular `bond:` is still accepted for older single-bond configs.

## Compact example

Three hosts → topology `compact` automatically:

```yaml
openshift_cluster_name: compact
openshift_api_vips:
  - 172.16.0.240
openshift_ingress_vips:
  - 172.16.0.241
openshift_hosts:
  - hostname: master-0.lab.naramajac.xyz
    role: master
    root_device_name: /dev/sda
    interfaces:
      - name: eno1
        mac_address: "52:54:00:00:00:20"
    network_config:
      ipv4:
        interface: eno1
        address: 172.16.0.20
        prefix_length: 24
      dns_resolver: [172.16.0.254]
      routes:
        - destination: 0.0.0.0/0
          next_hop_address: 172.16.0.1
  # … master-1, master-2 …
```

## Secrets

Provide `redhat_pull_secret` (or `openshift_pull_secret`) via vault, for example:

```bash
-e @vault/quay_secrets.yaml --vault-password-file …
```

Pass `openshift_ssh_key` on the CLI or in group_vars (public key only).
