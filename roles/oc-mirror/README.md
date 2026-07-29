# oc-mirror

Ansible role to mirror OpenShift content with **oc-mirror v2** for connected and disconnected environments.

## Purpose

The role supports a two-phase workflow:

1. **Connected** (`tasks/connected-mirror.yaml`) — On a host with internet access, download OpenShift release content to a local workspace and pack it into a transfer archive.
2. **Disconnected** (`tasks/quay-mirror.yaml` + `tasks/disconnected-mirror.yaml`) — Install the Quay mirror registry (if needed), restore the workspace from disk or archive, and push content into Quay with `oc-mirror` diskToMirror.

Playbooks:

| Playbook | Inventory group | Task files |
|----------|-----------------|------------|
| `playbooks/oc-mirror-create-connected.yaml` | `connected_mirror` | `connected-mirror` |
| `playbooks/oc-mirror-create-disconnected.yaml` | `disconnected_mirror` | `quay-mirror` (if Quay down), then `disconnected-mirror` |

Both groups are children of `oc_mirror` so they share `group_vars/oc_mirror.yml`.

After the disconnected mirror is populated, generate an Agent ISO with the
[`openshift-agent-iso`](../openshift-agent-iso/README.md) role
(`playbooks/openshift-agent-create-iso.yaml`).

## Requirements

- Ansible 2.14 or later
- `community.general` (archive/unarchive)
- Target host packages installed by the role as needed: `podman`, `jq`, `openssl`
- Outbound HTTPS on the **connected** host (OpenShift clients + image pulls)
- Outbound HTTPS on the **disconnected** host only to download `mirror-registry` (unless already cached under `/var/tmp`)
- Vault file with Red Hat / Quay credentials (see [Secrets](#secrets))

## Role layout

```
roles/oc-mirror/
├── defaults/main.yml
├── vars/main.yaml                 # openshift_version / archive defaults
├── tasks/
│   ├── connected-mirror.yaml      # mirrorToDisk + tarball
│   ├── disconnected-mirror.yaml   # restore workspace + diskToMirror
│   ├── quay-mirror.yaml           # install mirror-registry / Quay
│   └── install-oc-binaries.yaml   # oc / oc-mirror clients
└── templates/
    └── ImageSetConfiguration.yaml.j2
```

## Variables

### Shared paths (`defaults/main.yml` / `group_vars/oc_mirror.yml`)

| Variable | Role default | This repo (`group_vars`) | Description |
|----------|--------------|--------------------------|-------------|
| `oc_mirror_workdir` | `/opt/oc-mirror` | same | Workspace for binaries, config, and disk mirror |
| `oc_mirror_archive_path` | `/var/tmp/oc-mirror.tar.gz` | `/var/tmp/oc-mirror.tar` | Transfer archive (connected → disconnected) |
| `quay_workdir` | `/opt/quay-mirror` | same | Quay / mirror-registry install root (`--quayRoot`) |
| `openshift_version` | `4.22.4` | same | Client and ImageSet channel pin |
| `mirror_set_config` | empty catalogs | operators + `additionalImages` in group_vars | ImageSetConfiguration content |

The connected playbook archives with `format: tar` (see
`connected-mirror.yaml`). Prefer an archive path ending in `.tar` when using
the group_vars override.

### Secrets (vault)

Provide via `-e @vault/quay_secrets.yaml` (or equivalent):

| Variable | Required | Description |
|----------|----------|-------------|
| `redhat_pull_secret` | Yes | Red Hat pull secret JSON used as an authfile |
| `quay_user` | Yes (disconnected) | Initial Quay user for `mirror-registry install` / `podman login` |
| `quay_password` | Yes (disconnected) | Initial Quay password |
| `quay_sslCert` | No | PEM certificate for Quay; if set, `quay_sslKey` must also be set |
| `quay_sslKey` | No | PEM private key for Quay |

When both `quay_sslCert` and `quay_sslKey` are set, they are written to
`{{ quay_workdir }}/ssl.cert` and `ssl.key` and passed to install as
`--sslCert` / `--sslKey`. If neither is set, mirror-registry generates
certificates. Setting only one fails the role.

## Workflows

### Connected (mirrorToDisk)

1. Install OpenShift client / oc-mirror binaries into `oc_mirror_workdir`.
2. Render `ImageSetConfiguration.yaml` from the template.
3. Write `redhat-auth.json` from `redhat_pull_secret`.
4. Run `oc-mirror --v2 … file://{{ oc_mirror_workdir }}` when the ImageSet
   config changes (or `force_oc_mirror=true`).
5. Archive the workspace to `oc_mirror_archive_path` for transfer.

Copy the archive to the disconnected host (for example with `scp`) when the
hosts differ.

### Disconnected (Quay + diskToMirror)

The playbook first checks `https://{{ ansible_fqdn }}:8443/health/instance`.
If Quay is not healthy, it runs `quay-mirror` (download/install mirror-registry).

Then `disconnected-mirror`:

1. If `{{ oc_mirror_workdir }}/ImageSetConfiguration.yaml` is missing and the
   archive exists, extract `oc_mirror_archive_path` into
   `{{ oc_mirror_workdir | dirname }}` (archive entries are `oc-mirror/…`).
2. Fail if neither a populated workspace nor the archive is available.
3. `podman login` to Quay (`--tls-verify=false`), merge Quay + Red Hat auth
   into `auth.json`.
4. Run `oc-mirror --v2 --from file://{{ oc_mirror_workdir }} … --dest-tls-verify=false docker://$(hostname -f):8443`.

Generated cluster resources (including `idms-oc-mirror.yaml`) land under
`{{ oc_mirror_workdir }}/working-dir/cluster-resources/` and are consumed by
the Agent ISO role.

Same-host runs (connected then disconnected on one machine) skip archive
restore when the workspace is already populated.

## Usage

Connected:

```bash
ansible-playbook playbooks/oc-mirror-create-connected.yaml \
  -i inventory.yaml \
  -l registry-ext.lab.naramajac.xyz \
  -e @vault/quay_secrets.yaml \
  --vault-password-file /path/to/.vault_password
```

Transfer archive (if needed; path matches `group_vars/oc_mirror.yml`):

```bash
scp -3 user@registry-ext:/var/tmp/oc-mirror.tar \
       user@registry-int:/var/tmp/oc-mirror.tar
```

Disconnected:

```bash
ansible-playbook playbooks/oc-mirror-create-disconnected.yaml \
  -i inventory.yaml \
  -l registry-int.lab.naramajac.xyz \
  -e @vault/quay_secrets.yaml \
  --vault-password-file /path/to/.vault_password
```

Ensure inventory hosts are listed under `connected_mirror` /
`disconnected_mirror` as appropriate (`inventory.yaml`).

## Notes

- Disconnected Quay health checks and `podman login` / `oc-mirror` use TLS
  verification disabled for lab self-signed (or custom) certificates.
- Connected mirror only re-runs `oc-mirror` when the ImageSet configuration
  task reports changed (or `force_oc_mirror` is set).
- Disk sizing: plan for workspace + archive + Quay storage (often well over
  200 GiB for a full release mirror).
- oc-mirror v2 registry paths used by the Agent ISO role:
  - release images → `…/openshift/release-images`
  - release payload → `…/openshift/release`

## Example timings

### Connected mirror

```
PLAYBOOK RECAP
Playbook run took 0 days, 1 hours, 21 minutes, 10 seconds

Mirror content to disk with oc-mirror v2 .............................. ~59m
Create a tarball for transfer ......................................... ~20m
```

### Archive transfer

```
scp -o BatchMode=yes -3 \
  registry-ext.lab.naramajac.xyz:/var/tmp/oc-mirror.tar \
  registry-int.lab.naramajac.xyz:/var/tmp/
# ~63GB at ~80 MB/s ≈ 13 minutes
```

### Disconnected mirror

```
PLAYBOOK RECAP
Playbook run took 0 days, 1 hours, 59 minutes, 3 seconds

Run oc-mirror (diskToMirror) .......................................... ~97m
Restore workspace from transfer archive ............................... ~11m
Install Quay Mirror Registry .......................................... ~4m
```

## License

GPL-2.0-or-later

## Author

Nathan Reilly
