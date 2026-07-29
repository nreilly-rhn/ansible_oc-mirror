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
├── vars/main.yaml                 # openshift_version
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

| Variable | Default | Description |
|----------|---------|-------------|
| `oc_mirror_workdir` | `/opt/oc-mirror` | Workspace for oc-mirror binaries, config, and disk mirror |
| `oc_mirror_archive_path` | `/var/tmp/oc-mirror.tar.gz` | Transfer archive produced by connected / consumed by disconnected |
| `quay_workdir` | `/opt/quay-mirror` | Quay / mirror-registry install root (`--quayRoot`) |
| `openshift_version` | `4.22.4` | Client and ImageSet channel pin (`vars/main.yaml`) |

### Secrets (vault)

Provide via `-e @vault/quay_secrets.yaml` (or equivalent):

| Variable | Required | Description |
|----------|----------|-------------|
| `redhat_pull_secret` | Yes | Red Hat pull secret JSON used as an authfile |
| `quay_user` | Yes (disconnected) | Initial Quay user for `mirror-registry install` / `podman login` |
| `quay_password` | Yes (disconnected) | Initial Quay password |
| `quay_sslCert` | No | PEM certificate for Quay; if set, `quay_sslKey` must also be set |
| `quay_sslKey` | No | PEM private key for Quay |

When both `quay_sslCert` and `quay_sslKey` are set, they are written to `{{ quay_workdir }}/ssl.cert` and `ssl.key` and passed to install as `--sslCert` / `--sslKey`. If neither is set, mirror-registry generates certificates. Setting only one fails the role.

## Workflows

### Connected (mirrorToDisk)

1. Install OpenShift client / oc-mirror binaries into `oc_mirror_workdir`.
2. Render `ImageSetConfiguration.yaml` from the template.
3. Write `redhat-auth.json` from `redhat_pull_secret`.
4. Run `oc-mirror --v2 … file://{{ oc_mirror_workdir }}` when the ImageSet config changes.
5. Archive the workspace to `oc_mirror_archive_path` for transfer.

Copy the archive to the disconnected host (for example with `scp`) when the hosts differ.

### Disconnected (Quay + diskToMirror)

The playbook first checks `https://{{ ansible_fqdn }}:8443/health/instance`. If Quay is not healthy, it runs `quay-mirror` (download/install mirror-registry).

Then `disconnected-mirror`:

1. If `{{ oc_mirror_workdir }}/ImageSetConfiguration.yaml` is missing and the archive exists, extract `oc_mirror_archive_path` into `{{ oc_mirror_workdir | dirname }}` (archive entries are `oc-mirror/…`).
2. Fail if neither a populated workspace nor the archive is available.
3. `podman login` to Quay (`--tls-verify=false`), merge Quay + Red Hat auth into `auth.json`.
4. Run `oc-mirror --v2 --from file://{{ oc_mirror_workdir }} … --dest-tls-verify=false docker://$(hostname -f):8443`.

Same-host runs (connected then disconnected on one machine) skip archive restore when the workspace is already populated.

## Usage

Connected:

```bash
ansible-playbook playbooks/oc-mirror-create-connected.yaml \
  -l registry-ext.lab.naramajac.xyz \
  -e @vault/quay_secrets.yaml \
  --vault-password-file /path/to/.vault_password
```

Transfer archive (if needed):

```bash
scp -3 user@registry-ext:/var/tmp/oc-mirror.tar.gz \
       user@registry-int:/var/tmp/oc-mirror.tar.gz
```

Disconnected:

```bash
ansible-playbook playbooks/oc-mirror-create-disconnected.yaml \
  -l registry-int.lab.naramajac.xyz \
  -e @vault/quay_secrets.yaml \
  --vault-password-file /path/to/.vault_password
```

Ensure inventory hosts are listed under `connected_mirror` / `disconnected_mirror` as appropriate.

## Notes

- Disconnected Quay health checks and `podman login` / `oc-mirror` use TLS verification disabled for lab self-signed (or custom) certificates.
- Connected mirror only re-runs `oc-mirror` when the ImageSet configuration task reports changed.
- Disk sizing: plan for workspace + archive + Quay storage (often well over 200 GiB for a full release mirror).

## License

GPL-2.0-or-later

## Author

Nathan Reilly


## Timings

### Connected mirror
```
PLAYBOOK RECAP ****************************************************************************************************************************************************************************************************************************************************************************
Playbook run took 0 days, 1 hours, 21 minutes, 10 seconds


TASKS RECAP *************************************************************************************************
Wednesday 29 July 2026  09:21:04 +1000 (0:20:08.958)       1:21:10.760 ******** 
=============================================================================== 
../roles/oc-mirror : Mirror content to disk with oc-mirror v2 ----------------------------------------------- 3541.94s
../roles/oc-mirror : Create a tarball for transfer ---------------------------------------------------------- 1208.96s
../roles/oc-mirror : Download OpenShift client archives ----------------------------------------------------- 43.39s
../roles/oc-mirror : Extract binaries ----------------------------------------------------------------------- 37.80s
../roles/oc-mirror : Download the mirror-registry binary ---------------------------------------------------- 14.65s
../roles/oc-mirror : Cleanup the tarballs ------------------------------------------------------------------- 9.10s
../roles/oc-mirror : Copy ImageSetConfiguration ------------------------------------------------------------- 4.96s
../roles/oc-mirror : Create Red Hat pull secret ------------------------------------------------------------- 3.32s
Gathering Facts --------------------------------------------------------------------------------------------- 3.28s
../roles/oc-mirror : Create oc-mirror workspace ------------------------------------------------------------- 1.08s
../roles/oc-mirror : Check for oc-mirror binary ------------------------------------------------------------- 0.97s
../roles/oc-mirror : Ensure OpenShift client is correct ----------------------------------------------------- 0.97s
../roles/oc-mirror : Remove oc-mirror binary if OpenShift client is incorrect ------------------------------- 0.12s
Populate Mirror --------------------------------------------------------------------------------------------- 0.09s
../roles/oc-mirror : Install OpenShift binaries ------------------------------------------------------------- 0.06s

ROLES RECAP *************************************************************************************************
Wednesday 29 July 2026  09:21:04 +1000 (0:20:08.959)       1:21:10.760 ******** 
=============================================================================== 
../roles/oc-mirror ------------------------------------------------------------- 4867.32s
gather_facts ------------------------------------------------------------------- 3.28s
ansible.builtin.include_role --------------------------------------------------- 0.09s
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
total -------------------------------------------------------------------------- 4870.70s
```

### ssh archive
```
scp -o BatchMode=yes -3 registry-ext.lab.naramajac.xyz:/var/tmp/oc-mirror.tar registry-int.lab.naramajac.xyz:/var/tmp/
oc-mirror.tar                                                                              100%   63GB  80.7MB/s   13:25
```
### Disconnected mirror
```
PLAYBOOK RECAP **********************************************************************************************
Playbook run took 0 days, 1 hours, 59 minutes, 3 seconds


TASKS RECAP *************************************************************************************************
Wednesday 29 July 2026  11:35:32 +1000 (1:36:41.311)       1:59:03.798 ******** 
=============================================================================== 
../roles/oc-mirror : Run oc-mirror -------------------------------------------------------------------------- 5801.31s
../roles/oc-mirror : Restore workspace from transfer archive ------------------------------------------------ 673.45s
../roles/oc-mirror : Check for transfer archive ------------------------------------------------------------- 339.44s
../roles/oc-mirror : Install Quay Mirror Registry ----------------------------------------------------------- 258.29s
../roles/oc-mirror : Extract the Quay Mirror Registry ------------------------------------------------------- 57.59s
Gathering Facts --------------------------------------------------------------------------------------------- 4.77s
../roles/oc-mirror : Login to the Mirror -------------------------------------------------------------------- 2.67s
../roles/oc-mirror : Create Directories --------------------------------------------------------------------- 1.91s
Check if mirror is up --------------------------------------------------------------------------------------- 1.67s
../roles/oc-mirror : Check whether oc-mirror workspace is populated ----------------------------------------- 0.95s
../roles/oc-mirror : Check whether oc-mirror workspace is populated ----------------------------------------- 0.95s
../roles/oc-mirror : Build mirror-registry install arguments ------------------------------------------------ 0.14s
../roles/oc-mirror : Validate custom TLS variables are paired ----------------------------------------------- 0.09s
Install Quay ------------------------------------------------------------------------------------------------ 0.09s
../roles/oc-mirror : Copy Quay TLS private key -------------------------------------------------------------- 0.08s
../roles/oc-mirror : Copy Quay TLS certificate -------------------------------------------------------------- 0.08s
Populate Mirror --------------------------------------------------------------------------------------------- 0.07s
../roles/oc-mirror : Print all available facts -------------------------------------------------------------- 0.05s
../roles/oc-mirror : Fail when workspace and archive are both missing --------------------------------------- 0.05s
../roles/oc-mirror : Restore workspace from transfer archive ------------------------------------------------ 0.05s

ROLES RECAP *************************************************************************************************
Wednesday 29 July 2026  11:35:32 +1000 (1:36:41.312)       1:59:03.799 ******** 
=============================================================================== 
../roles/oc-mirror --------------------------------------------------- 7137.18s
gather_facts ------------------------------------------------------------ 4.77s
ansible.builtin.uri ----------------------------------------------------- 1.67s
ansible.builtin.include_role -------------------------------------------- 0.16s
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
total ---------------------------------------------------------------- 7143.78s
```
