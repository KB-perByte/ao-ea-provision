# ao-ea-provision Execution Environment

EE for [`ao-ea-provision`](../../ao-ea-provision): OpenShift playbooks that
teardown/reprovision early-access Automation Orchestrator (DB PVC kept).

Built with [ansible-builder](https://docs.ansible.com/projects/builder/en/latest/)
**schema v3**, following:

- [EE definition](https://docs.ansible.com/projects/builder/en/latest/definition/)
- [Galaxy Hub env / build-arg auth](https://docs.ansible.com/projects/builder/en/latest/scenario_guides/scenario_custom/)
- [Passing secrets](https://docs.ansible.com/projects/builder/en/latest/scenario_guides/scenario_secret_passing/) (optional stronger pattern)

## Standards applied (vs older personal EEs)

| Practice | Why |
|---|---|
| `version: 3` | Required for builder 3.x schema |
| AAP `ee-minimal` base | Already has `ansible-core` + `ansible-runner` — do not re-pip them |
| `package_manager_path: /usr/bin/microdnf` | Downstream AAP images |
| Hub token as **build-arg only** | Never commit / hardcode tokens in YAML (unlike older `ENV …=mytoken` patterns) |
| ARG name = `ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN` | Matches Galaxy client env (builder docs) |
| `ansible.cfg` for Hub/Galaxy **URLs only** | Non-secret server config via `additional_build_files` |
| `append_final` import + collection checks | Fail the build if EE is incomplete |
| No cluster/PG secrets in the image | Inject at job runtime via AAP credentials |

## Files

| File | Purpose |
|---|---|
| `execution-environment.yml` | ansible-builder v3 definition |
| `ansible-collections.yml` | `kubernetes.core`, `redhat.openshift` |
| `python-packages.txt` | Python deps for `kubernetes.core` |
| `ansible.cfg` | Automation Hub + Galaxy server list |

## Build

```bash
cd /home/sagpaul/Work/nexus/sagpaul-ee/ao-ea-provision

podman login registry.redhat.io

# Export Hub token in the shell (do not commit it), then:
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN

ansible-builder build -v3 \
  --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN \
  --container-runtime podman
```

Default tag comes from `options.tags` (`quay.io/sagpaul/ao-ea-provision:latest`).
Override with `--tag` if needed.

### Stronger: Hub token as a build secret (optional)

Avoids putting the token in build-arg metadata ([secret passing](https://docs.ansible.com/projects/builder/en/latest/scenario_guides/scenario_secret_passing/)):

```bash
printf '%s' "$ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN" > /tmp/ah_token
chmod 600 /tmp/ah_token

# You would adjust prepend_galaxy to use:
#   RUN --mount=type=secret,id=ah_token …
# See ansible-builder "Passing Secrets" scenario if you adopt this.
```

For most Hub collection installs, the official **build-arg** pattern above is enough.

## Push

```bash
podman login quay.io
podman push quay.io/sagpaul/ao-ea-provision:latest
```

## Verify

```bash
podman run --rm quay.io/sagpaul/ao-ea-provision:latest \
  ansible-galaxy collection list | grep -E 'kubernetes.core|redhat.openshift'

podman run --rm quay.io/sagpaul/ao-ea-provision:latest \
  python3 -c 'import kubernetes, yaml, requests, jsonpatch; print("ok")'
```

## AAP wiring

1. Register the image as an Execution Environment in Controller.
2. Project SCM → `ao-ea-provision` playbook repo.
3. Inventory → localhost.
4. Credentials → OpenShift/K8s + PG password creds (runtime, not baked in).
5. JT playbooks → `playbooks/*.yml`.
6. `ea-health-check` timeout **> 25m**.

Details: `ao-ea-provision/docs/aap-setup.md`.
