# openshift-agent-iso

Generate an OpenShift **Agent-based Installer** ISO from a release image on a
disconnected mirror (extract `openshift-install`, render configs, create image).

## Purpose

1. Log in to the disconnected mirror and build a **mirror-only** pull secret for
   `install-config.yaml`.
2. Resolve `imageDigestSources` (from oc-mirror IDMS) and
   `additionalTrustBundle` (mirror TLS cert).
3. Extract `openshift-install` (and `oc`) from the mirrored release image with
   `oc adm release extract --idms-file=...`.
4. Render `install-config.yaml` and `agent-config.yaml` (NMState networking,
   root device hints, bonds/VLANs/routes).
5. Run `openshift-install agent create image`.

Playbook:

| Playbook | Inventory group | Task file |
|----------|-----------------|-----------|
| `playbooks/openshift-agent-create-iso.yaml` | `openshift_agent_iso` | `create-iso` |

Typical target: `registry-int.lab.naramajac.xyz` (has Quay + oc-mirror workspace).

## Requirements

- Ansible 2.14 or later
- Collections: `community.crypto` (mirror TLS), `ansible.utils` (CIDR helpers)
- On the target host: `podman`, `jq`, and `oc` (`openshift_oc_bin`, often
  `/opt/oc-mirror/oc`)
- Vault / extra-vars: `redhat_pull_secret`, `quay_user`, `quay_password`
- Network access from the target to `openshift_mirror_registry`
- Populated oc-mirror workspace (for IDMS under
  `…/working-dir/cluster-resources/`) when using auto IDMS load

## Role layout

```
roles/openshift-agent-iso/
├── defaults/main.yml
├── vars/main.yaml                  # topology → replicas / platform
├── tasks/
│   ├── create-iso.yaml               # validate + orchestrate
│   ├── derive-machine-network.yaml   # machineNetwork from role: node
│   ├── prepare-mirror-config.yaml    # IDMS + mirror CA for install-config
│   ├── extract-installer.yaml        # oc adm release extract --idms-file
│   ├── render-configs.yaml           # install-config + agent-config
│   └── create-image.yaml             # openshift-install agent create image
└── templates/
    ├── install-config.yaml.j2
    ├── agent-config.yaml.j2
    └── idms-release-extract.yaml.j2  # fallback IDMS if none from oc-mirror
```

## Quick start

```bash
ansible-playbook playbooks/openshift-agent-create-iso.yaml \
  -i inventory.yaml \
  -l registry-int.lab.naramajac.xyz \
  -e @vault/quay_secrets.yaml \
  -e openshift_ssh_key="$(cat ~/.ssh/id_ed25519.pub)" \
  --vault-password-file /path/to/.vault_password
```

Cluster inputs live in `group_vars/openshift_agent_iso.yml` (hosts, VIPs,
networks, mirror registry).

### Outputs

| Path | Contents |
|------|----------|
| `{{ openshift_agent_iso_workdir }}/` | Persistent `install-config.yaml`, `agent-config.yaml`, extracted binaries, `mirror-ca.crt` |
| `{{ openshift_agent_iso_install_dir }}/` | Config copies consumed by the installer + generated `agent*.iso` |

Default workdir: `/opt/openshift-agent-iso`  
Default install dir: `/opt/openshift-agent-iso/install`

`openshift-install agent create image --dir=…/install` **consumes** the config
copies under `install/` (they are removed after a successful run). Keep using
the persistent copies in the workdir root for edits/re-runs.

## Variables

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `openshift_agent_iso_workdir` | `/opt/openshift-agent-iso` | Binary, auth, persistent configs |
| `openshift_agent_iso_install_dir` | `…/install` | Consumed config copies + generated ISO |
| `openshift_version` | `4.22.4` | Used in default release image tag |
| `openshift_mirror_registry` | `registry-int.lab.naramajac.xyz:8443` | Internal Quay host:port (no scheme) |
| `openshift_release_image` | `{{ mirror }}/openshift/release-images:{{ version }}-x86_64` | oc-mirror v2 release image for extract |
| `openshift_oc_bin` | `oc` | Path to `oc` (group_vars often sets `/opt/oc-mirror/oc`) |
| `openshift_topology` | auto | Detected from `openshift_hosts` count — do not set manually |
| `openshift_cluster_name` | `sno` | Cluster name in install-config |
| `openshift_base_domain` | `lab.naramajac.xyz` | baseDomain |
| `openshift_pull_secret` | `{{ redhat_pull_secret }}` | Seed for assert; install-config uses mirror-only auth from `podman login` |
| `openshift_ssh_key` | `""` | Public SSH key (required) |
| `openshift_machine_network` | derived | install-config `machineNetwork` (from `role: node`) |
| `openshift_machine_network_from_hosts` | `true` | Derive machineNetwork from host address roles |
| `openshift_api_vips` | `[]` | Required for compact/ha; must be on node network |
| `openshift_ingress_vips` | `[]` | Required for compact/ha; must be on node network |
| `openshift_hosts` | example SNO host | Host list for agent-config |
| `openshift_rendezvous_ip` | `""` | Defaults to first host IPv4; must be on node network |
| `openshift_oc_mirror_workdir` | `/opt/oc-mirror` | oc-mirror workspace |
| `openshift_oc_mirror_cluster_resources` | `…/working-dir/cluster-resources` | IDMS source dir |
| `openshift_image_digest_sources` | auto / defaults | Written to install-config |
| `openshift_additional_trust_bundle` | auto from TLS port | Mirror cert PEM (optional override) |
| `force_openshift_install_extract` | `false` | Re-extract installer / `oc` binaries |

### Disconnected mirror (required)

`install-config.yaml` always includes `imageDigestSources` and
`additionalTrustBundle`:

1. **imageDigestSources** — loaded from oc-mirror `idms*.yaml` under
   `openshift_oc_mirror_cluster_resources` when present; otherwise
   `openshift_image_digest_sources_default` (oc-mirror v2 paths):
   - `quay.io/…/ocp-release` → `{{ mirror }}/openshift/release-images`
   - `quay.io/…/ocp-v4.0-art-dev` → `{{ mirror }}/openshift/release`
   - `registry.redhat.io` → `{{ mirror }}`
2. **additionalTrustBundle** — from `openshift_additional_trust_bundle` if set;
   else fetched from `openshift_mirror_registry` with
   `community.crypto.get_certificate` (leaf + chain when available).

Set `openshift_image_digest_sources_force: true` to skip IDMS auto-load and use
an explicit `openshift_image_digest_sources` list.

### Extracting `openshift-install`

`oc adm release extract` does **not** use `/etc/containers/registries.conf` for
payload pulls. The role passes `--idms-file` (oc-mirror IDMS when present, else
a rendered fallback). Manual equivalent:

```bash
/opt/oc-mirror/oc adm release extract \
  --registry-config="$HOME/.dockercfg" \
  --insecure=true \
  --idms-file=/opt/oc-mirror/working-dir/cluster-resources/idms-oc-mirror.yaml \
  --command=openshift-install \
  --to=/opt/openshift-agent-iso \
  --from=registry-int.lab.naramajac.xyz:8443/openshift/release-images:4.22.4-x86_64
```

### Pull secret for install-config

The role runs `podman login` to the mirror, then `jq -c . ~/.dockercfg` so
`pullSecret` is a **single-line JSON string** with double quotes. A Python-dict
repr (`{'auths': …}`) breaks YAML parsing in `openshift-install`.

### Topology detection

Topology is derived from `openshift_hosts | length`:

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
      interface: enp1s0           # interface (or bond/vlan) that gets the address
      address: 172.16.0.10
      prefix_length: 24
      # role: node                # optional when only one address
      # dhcp: false
    dns_resolver:
      - 172.16.0.254
    routes:
      - destination: 0.0.0.0/0
        next_hop_address: 172.16.0.1
        next_hop_interface: enp1s0
```

### machineNetwork from address `role`

`install-config.yaml` `machineNetwork` is a **single** CIDR for the cluster node /
API fabric. It is derived from host addresses:

| Rule | Behavior |
|------|----------|
| One address, no `role` | Defaults to `role: node` |
| Multiple addresses | Every address needs `role`; exactly one `node` per host |
| Across hosts | All `role: node` addresses must resolve to the same CIDR |
| VIPs / rendezvousIP | Must fall inside that node CIDR |

CIDR is computed with `ansible.utils.ipaddr('network/prefix')` from
`address` + `prefix_length` (e.g. `172.16.1.226/24` → `172.16.1.0/24`).

Additional fabrics (`role: storage`, etc.) stay in agent-config NMState only —
they are **not** written to `machineNetwork`.

To supply `openshift_machine_network` yourself instead:

```yaml
openshift_machine_network_from_hosts: false
openshift_machine_network:
  - cidr: 172.16.1.0/24
```

### Multi-NIC (IP on one interface)

List every NIC under `interfaces`. Set `ipv4.interface` to the one that should
receive the address; others are brought up with IPv4 disabled.

### Bonds and VLANs

Use `bonds` (list) and optional `vlans`. Put node IPs on a bond or VLAN via
`ipv4.interface` or `ipv4_addresses` (see `group_vars/openshift_agent_iso.yml`
for a full compact example with 4 NICs, 2 bonds, and `bond0.100`).

```yaml
network_config:
  ipv4_addresses:
    - interface: bond0.100
      address: 172.16.1.226
      prefix_length: 24
      role: node
    - interface: bond1
      address: 10.254.224.226
      prefix_length: 24
      role: storage
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

Three hosts → topology `compact` automatically. A complete dual-network compact
layout is in `group_vars/openshift_agent_iso.yml`. Minimal shape:

```yaml
openshift_cluster_name: compact
openshift_api_vips:
  - 172.16.1.240
openshift_ingress_vips:
  - 172.16.1.241
openshift_hosts:
  - hostname: master-0.lab.naramajac.xyz
    role: master
    root_device_name: /dev/sda
    interfaces:
      - name: eno1
        mac_address: "52:54:01:00:00:11"
    network_config:
      ipv4:
        interface: eno1
        address: 172.16.1.226
        prefix_length: 24
        # role: node  # optional for single-address hosts
      dns_resolver: [172.16.1.253]
      routes:
        - destination: 0.0.0.0/0
          next_hop_address: 172.16.1.254
          next_hop_interface: eno1
  # … master-1, master-2 …
```

## Secrets

Provide via vault, for example:

```bash
-e @vault/quay_secrets.yaml --vault-password-file …
```

| Variable | Used for |
|----------|----------|
| `redhat_pull_secret` | Satisfies role assert / seed auth |
| `quay_user` / `quay_password` | `podman login` → mirror-only `pullSecret` |

Pass `openshift_ssh_key` on the CLI or in group_vars (public key only).

## Example timings

```
PLAYBOOK RECAP
Playbook run took 0 days, 0 hours, 3 minutes, 4 seconds

Create Agent-based Installer ISO ...................................... ~124s
Extract openshift-install and oc CLI from release image ............... ~35s
```

## License

GPL-2.0-or-later

## Author

Nathan Reilly
