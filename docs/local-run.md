# Local run guide

Run the provision playbooks from your laptop against OpenShift instance (or a
compatible cluster) without AAP. Use this for debugging one stage at a
time. Production weekly runs should still go through AAP Job Templates
and the stable AO workflow ([aap-setup.md](aap-setup.md),
[ao-workflow.md](ao-workflow.md)).

**Never commit** tokens, passwords, kubeconfigs, or vault files.

## Prerequisites

| Tool | Notes |
|---|---|
| Python 3.11+ | venv recommended |
| `ansible-core` ≥ 2.15 | `pip install 'ansible-core>=2.15'` |
| `kubernetes` Python package | required by `kubernetes.core` |
| `ansible-galaxy` | install `requirements.yml` |
| Network to `cluster_api` | and a token with cluster-admin (or equivalent) |
| Optional: `oc` | handy for post-run checks; not required by playbooks |

```bash
cd /path/to/ao-ea-provision
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible-core>=2.15' kubernetes
ansible-galaxy collection install -r requirements.yml
```

`redhat.openshift` may fail from public Galaxy — that is expected. These
playbooks use `kubernetes.core` only.

## Secrets file (gitignored)

Create `secrets/` (ignored by Git) and copy the example:

```bash
mkdir -p secrets
cp docs/examples/local.extra_vars.example.yml secrets/local.extra_vars.yml
chmod 600 secrets/local.extra_vars.yml
```

Edit `secrets/local.extra_vars.yml` and fill in **only** secret values.
Non-secret defaults already come from `inventory/group_vars/all.yml`.

Confirm Git will not track it:

```bash
git check-ignore -v secrets/local.extra_vars.yml
# expected: .gitignore match for secrets/
git status --short secrets/
# expected: empty / ignored
```

Auth options (pick one):

1. **`k8s_api_key`** — bearer token for `cluster_api` (typical for ROSA)
2. **`k8s_kubeconfig`** — path to a kubeconfig file (also gitignored via `*.kubeconfig`)

Do not put the token on the shell history if you can avoid it:

```bash
# OK
ansible-playbook playbooks/preflight.yml -e @secrets/local.extra_vars.yml

# Avoid (leaks into shell history / process list)
# ansible-playbook … -e k8s_api_key=sha256~…
```

## Offline checks (no cluster)

Run these before any live mutation:

```bash
source .venv/bin/activate

for p in playbooks/{clean_install,update_release,preflight,teardown_app,ensure_db,ensure_secrets,ensure_operator,apply_ao_cr,postconfig,health_check}.yml; do
  ansible-playbook --syntax-check "$p"
done

for t in tests/*.yml; do
  echo "=== $t ==="
  ansible-playbook "$t"
done
```

## Four modes (preferred)

Include `pg_execution_plane_password` when `script_nodes_enabled=true`
(default). Channel is `early-access` or `stable`.

```bash
EX="-e @secrets/local.extra_vars.yml"

# Update existing install (keep DB PVC + AO CR; reconcile operator channel)
ansible-playbook playbooks/update_release.yml $EX -e ao_channel=stable
ansible-playbook playbooks/update_release.yml $EX -e ao_channel=early-access

# Clean install on a new / empty cluster
ansible-playbook playbooks/clean_install.yml $EX -e ao_channel=stable
ansible-playbook playbooks/clean_install.yml $EX -e ao_channel=early-access

# Optional weekly app wipe (keep DB): teardown then update
ansible-playbook playbooks/teardown_app.yml $EX
ansible-playbook playbooks/update_release.yml $EX -e ao_channel=early-access
```

## Live order (OpenShift instance) — step-by-step

Run **one granular playbook at a time** when debugging. Stop on failure;
fix; resume from that stage. Teardown keeps the DB project. App project
is `app_ns` from inventory.

Recommended path **with** script nodes (`script_nodes_enabled=true`).
Include `pg_execution_plane_password` in `secrets/local.extra_vars.yml`.

```bash
EX="-e @secrets/local.extra_vars.yml"

ansible-playbook playbooks/preflight.yml $EX

ansible-playbook playbooks/teardown_app.yml $EX

ansible-playbook playbooks/ensure_db.yml $EX
# creates execution_plane role + database when script_nodes_enabled=true

ansible-playbook playbooks/ensure_secrets.yml $EX
# applies {ao}-execution-plane-database Secret
# If PG connect check flakes on SCLORG pg_hba:
# ansible-playbook playbooks/ensure_secrets.yml $EX -e secrets_verify_connect=false

ansible-playbook playbooks/ensure_operator.yml $EX
# Override channel only if Subscription already matches:
# ansible-playbook playbooks/ensure_operator.yml $EX -e ao_channel=stable

ansible-playbook playbooks/apply_ao_cr.yml $EX

ansible-playbook playbooks/postconfig.yml $EX
# ENABLE_EXECUTION_PLANE + wait for APP_SCRIPT_NODES_ENABLED

# Long wait — Ready + probe (often 10–20+ minutes)
ansible-playbook playbooks/health_check.yml $EX
```

Skip EP entirely with `-e script_nodes_enabled=false` on ensure_db,
ensure_secrets, and postconfig. Details:
[roles/ao_ea_postconfig/README.md](../roles/ao_ea_postconfig/README.md).

## Useful one-offs

```bash
# Guards only (no cluster API)
ansible-playbook playbooks/safety_assert.yml \
  -e teardown_target_ns=syntarogation-ao \
  -e teardown_delete_namespaces=[] \
  -e teardown_delete_pvcs=[]

# Re-run health after a timeout without tearing down again
ansible-playbook playbooks/health_check.yml -e @secrets/local.extra_vars.yml

# TLS to API with a custom CA or lab cluster
ansible-playbook playbooks/preflight.yml -e @secrets/local.extra_vars.yml \
  -e k8s_validate_certs=false
```

## Post-run checks (`oc`)

Use a separate login session; do not paste tokens into the repo.

```bash
oc whoami --show-server
oc get automationorchestrator my-orchestrator -n syntarogation-ao
oc get pvc -n syntarogation-ao-db
# Confirm the same PVC name / volume still Bound after teardown+provision

oc get route -n syntarogation-ao
# Admin password — presence only unless you intentionally extract locally:
oc get secret my-orchestrator-initial-admin-password -n syntarogation-ao \
  -o jsonpath='{.data}' | jq 'keys'
```

## Troubleshooting

| Symptom | What to try |
|---|---|
| `Failed to import kubernetes` | `pip install kubernetes` in the same venv as ansible |
| 401 / 403 from API | Refresh token; confirm `cluster_api` matches the token’s server |
| `ensure-operator` channel / CSV wait | Confirm CatalogSource has the channel; `operator_manage_channel=true` reconciles channel then waits `AtLatestKnown` |
| `postconfig` fails closed | Re-run `ensure_db` + `ensure_secrets` with `pg_execution_plane_password`; or `-e script_nodes_enabled=false` |
| missing EP password | Add `pg_execution_plane_password` to `secrets/local.extra_vars.yml` |
| `ensure-secrets` password mismatch | Same `pg_*_password` values as `ensure-db`; or `-e secrets_verify_connect=false` for connect-only flake |
| Health timeout | Re-run `health_check` only; check pods/events in `app_ns` |

## Security checklist before `git push`

```bash
git status
git check-ignore -v secrets/local.extra_vars.yml
git diff --cached | rg -i 'token|password|BEGIN |kubeconfig' || true
```

If anything under `secrets/` or a kubeconfig appears as a staged file,
**unstage and stop**.
