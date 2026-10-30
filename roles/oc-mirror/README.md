# oc-mirror

Ansible role to mirror OpenShift content with **oc-mirror v2** for connected and
disconnected environments, using a **CNCF Distribution** registry on the
disconnected host.

## Purpose

The role supports a two-phase workflow:

1. **Connected** (`tasks/connected-mirror.yaml`) — On a host with internet
   access, download OpenShift release content to a local workspace, save the
   Distribution registry image for air-gapped transfer, and pack a transfer
   archive.
2. **Disconnected** (`tasks/registry.yaml` + `tasks/disconnected-mirror.yaml`) —
   Install the Distribution registry as a Podman Quadlet / systemd unit if
   needed, restore the workspace from disk or archive, and push content with
   `oc-mirror` diskToMirror.

Playbooks:

| Playbook | Inventory group | Task files |
|----------|-----------------|------------|
| `playbooks/oc-mirror-create-connected.yaml` | `connected_mirror` | `connected-mirror` |
| `playbooks/oc-mirror-create-disconnected.yaml` | `disconnected_mirror` | `registry` (if down), then `disconnected-mirror` |

Both groups are children of `oc_mirror` so they share `group_vars/oc_mirror.yml`.

After the disconnected mirror is populated, generate an Agent ISO with the
[`openshift-agent-iso`](../openshift-agent-iso/README.md) role
(`playbooks/openshift-agent-create-iso.yaml`).

## Requirements

- Ansible 2.14 or later
- Collections: `community.general`, `containers.podman`, `ansible.posix`
- Target host packages installed by the role as needed: `podman`, `httpd-tools`,
  `openssl`, `jq`
- Outbound HTTPS on the **connected** host (OpenShift clients + image pulls +
  `docker.io/library/registry:3`)
- Disconnected host can load the registry image from the transfer archive
  (no outbound pull required when the archive includes `registry.tar`)
- Vault file with Red Hat / registry credentials (see [Secrets](#secrets))

## Role layout

```
roles/oc-mirror/
├── defaults/main.yml
├── vars/main.yaml                 # openshift_version / archive defaults
├── tasks/
│   ├── connected-mirror.yaml      # mirrorToDisk + registry image + tarball
│   ├── disconnected-mirror.yaml   # restore workspace + diskToMirror
│   ├── registry.yaml              # CNCF Distribution as Podman systemd unit
│   ├── registry-save-image.yaml   # pull/save registry image on connected host
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
| `registry_workdir` | `/opt/registry` | same | Distribution install root (config, certs, auth, data) |
| `registry_port` | `8443` | same | Host port published for the registry |
| `registry_listen_port` | `5000` | role default | Port inside the container |
| `registry_image` | `docker.io/library/registry:3` | role default | CNCF Distribution image |
| `registry_unit` | `registry` | role default | Quadlet / systemd unit basename |
| `registry_image_archive` | `{{ oc_mirror_workdir }}/registry.tar` | role default | Saved image carried in the transfer archive |
| `openshift_version` | `4.22.4` | same | Client and ImageSet channel pin |
| `mirror_set_config` | empty catalogs | operators + `additionalImages` in group_vars | ImageSetConfiguration content |

The connected playbook archives with `format: tar` (see
`connected-mirror.yaml`). Prefer an archive path ending in `.tar` when using
the group_vars override.

### Secrets (vault)

Provide via `--extra-vars @vault/…` for every file under `vault/`
(or equivalent):

| Variable | Required | Description |
|----------|----------|-------------|
| `redhat_pull_secret` | Yes | Red Hat pull secret JSON used as an authfile |
| `registry_user` | Yes (disconnected) | htpasswd user for registry install / `podman login` |
| `registry_password` | Yes (disconnected) | htpasswd password |
| `registry_ssl_cert` | No | PEM certificate for the registry; if set, `registry_ssl_key` must also be set |
| `registry_ssl_key` | No | PEM private key for the registry |

When both `registry_ssl_cert` and `registry_ssl_key` are set, they are written
to `{{ registry_workdir }}/certs/domain.crt` and `domain.key`. If neither is
set, the role generates a self-signed certificate for the host FQDN. Setting
only one fails the role.

## Workflows

### Connected (mirrorToDisk)

1. Install OpenShift client / oc-mirror binaries into `oc_mirror_workdir`.
2. Render `ImageSetConfiguration.yaml` from the template.
3. Write `redhat-auth.json` from `redhat_pull_secret`.
4. Run `oc-mirror --v2 … file://{{ oc_mirror_workdir }}` when the ImageSet
   config changes (or `force_oc_mirror=true`).
5. Pull and save the Distribution image to `registry_image_archive`.
6. Archive the workspace to `oc_mirror_archive_path` for transfer.

Copy the archive to the disconnected host (for example with `scp`) when the
hosts differ. On a shared virtiofs mount, point both hosts at the same
`oc_mirror_archive_path` instead.

### Disconnected (registry + diskToMirror)

The playbook first checks `https://{{ ansible_fqdn }}:{{ registry_port }}/v2/`.
A live registry with htpasswd returns **401**. Any other status runs
`registry` (load or pull the image, Quadlet unit, TLS, htpasswd).

Then `disconnected-mirror`:

1. If `{{ oc_mirror_workdir }}/ImageSetConfiguration.yaml` is missing and the
   archive exists, extract `oc_mirror_archive_path` into
   `{{ oc_mirror_workdir | dirname }}` (archive entries are `oc-mirror/…`).
2. Fail if neither a populated workspace nor the archive is available.
3. `podman login` to the registry (`--tls-verify=false`).
4. Run `oc-mirror --v2 --from file://{{ oc_mirror_workdir }} …
   --dest-tls-verify=false docker://$(hostname -f):{{ registry_port }}`.

Generated cluster resources (including `idms-oc-mirror.yaml`) land under
`{{ oc_mirror_workdir }}/working-dir/cluster-resources/` and are consumed by
the Agent ISO role.

Same-host runs (connected then disconnected on one machine) skip archive
restore when the workspace is already populated.

## Usage

Connected:

```bash
ansible-playbook playbooks/oc-mirror-create-connected.yaml \
  --inventory inventory.yaml \
  --limit registry-ext.lab.naramajac.xyz \
  --extra-vars @vault/redhat_secrets.yaml \
  --extra-vars @vault/registry_secrets.yaml \
  --extra-vars @vault/ingress_tls_secrets.yaml \
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
  --inventory inventory.yaml \
  --limit registry-int.lab.naramajac.xyz \
  --extra-vars @vault/redhat_secrets.yaml \
  --extra-vars @vault/registry_secrets.yaml \
  --extra-vars @vault/ingress_tls_secrets.yaml \
  --vault-password-file /path/to/.vault_password
```

Ensure inventory hosts are listed under `connected_mirror` /
`disconnected_mirror` as appropriate (`inventory.yaml`).

## Notes

- Registry API: `https://<fqdn>:{{ registry_port }}/v2/` (401 with auth).
  There is no web UI.
- `podman login` / `oc-mirror` use TLS verification disabled for lab
  self-signed (or custom) certificates.
- Connected mirror only re-runs `oc-mirror` when the ImageSet configuration
  task reports changed (or `force_oc_mirror` is set). The transfer archive is
  also rebuilt when `registry.tar` is newer than the archive.
- Disk sizing: plan for workspace + archive + registry storage (often well
  over 200 GiB for a full release mirror).
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
Install registry ...................................................... ~4m
```

## License

GPL-2.0-or-later

## Author

Nathan Reilly
