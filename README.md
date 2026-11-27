# ao-ea-provision

Ansible content for **weekly teardown + reprovision** of an early-access
Automation Orchestrator instance on OpenShift, while
keeping the PostgreSQL project and PVC intact.

Stable AO (or a local `ansible-playbook` run) drives these playbooks; AAP
Job Templates are the intended production path. Stitching guides:

- [docs/aap-setup.md](docs/aap-setup.md) — Job Templates, credentials, EE
- [docs/ao-workflow.md](docs/ao-workflow.md) — stable AO workflow node order
- [docs/local-run.md](docs/local-run.md) — run playbooks from your laptop

## Four provision modes

| Scenario | Playbook | Channel |
|---|---|---|
| Update existing (keep DB + CR) | `playbooks/update_release.yml` | `-e ao_channel=stable` or `early-access` |
| Clean install (new cluster) | `playbooks/clean_install.yml` | `-e ao_channel=stable` or `early-access` |

```bash
EX="-e @secrets/local.extra_vars.yml"
ansible-playbook playbooks/update_release.yml $EX -e ao_channel=stable
ansible-playbook playbooks/update_release.yml $EX -e ao_channel=early-access
ansible-playbook playbooks/clean_install.yml $EX -e ao_channel=stable
ansible-playbook playbooks/clean_install.yml $EX -e ao_channel=early-access
```

Optional weekly app wipe (keep DB): `teardown_app.yml` then `update_release.yml`.

## Granular playbooks (AAP JT / step-by-step)

| # | Playbook | Purpose |
|---|---|---|
| 1 | `playbooks/preflight.yml` | API / catalog / `ao_channel` available |
| 2 | `playbooks/teardown_app.yml` | Delete AO CR in `app_ns`; **never** touch DB ns/PVC |
| — | `playbooks/teardown_full.yml` | **Destructive:** AO + BYO PG (needs `-e destroy_confirm=DESTROY-AO-AND-DB`) |
| 3 | `playbooks/ensure_db.yml` | Ensure PG StatefulSet + init; never delete PVC |
| 4 | `playbooks/ensure_secrets.yml` | Recreate app-ns PG Secrets from vars |
| 5 | `playbooks/ensure_operator.yml` | Install or reconcile Subscription to `ao_channel`; wait latest CSV |
| 6 | `playbooks/apply_ao_cr.yml` | Apply `AutomationOrchestrator` CR |
| 7 | `playbooks/postconfig.yml` | Script nodes via Execution Plane (or skip) |
| 8 | `playbooks/health_check.yml` | Ready + Route + HTTPS `/health` |
| 9 | `playbooks/sync_admin_password.yml` | Sync `{ao}-initial-admin-password` into the DB (after Ready) |

`playbooks/safety_assert.yml` is a local guard helper, not a JT.
`clean_install.yml` and `update_release.yml` run steps 1–9 (health then
admin sync) so a fresh install and a teardown+recreate against a persistent
`orchestrator` DB both leave Secret and login password aligned.

Defaults (non-secret) live in `inventory/group_vars/all.yml`:

| Var | Default |
|---|---|
| `app_ns` | `syntarogation-ao` |
| `db_ns` | `syntarogation-ao-db` |
| `ao_channel` | `early-access` |
| `ao_script_python_image_reference` | `registry.redhat.io/ansible-automation-platform/ao-python-rhel9:2026.8` |
| `ao_script_bash_image_reference` | `registry.redhat.io/ansible-automation-platform/ao-bash-rhel9:2026.8` |

`ensure_operator` places the script image references in the OLM Subscription's
`spec.config.env`, then waits for the Operator manager rollout and confirms
both values reached its Deployment before the AO CR can trigger DB seeding.
This avoids the Operator's Quay `:devel` defaults. Override either reference
with Ansible extra-vars or inventory when a different image release is needed.

## Secrets policy

**No credentials belong in this repository.**

| Allowed in Git | Never in Git |
|---|---|
| Namespace names, API URL hostnames, image names | OpenShift API tokens |
| Secret *names* / key *names* | PostgreSQL passwords |
| Var names (`pg_orchestrator_password`, `k8s_api_key`) | Initial admin password values |
| Empty mocks in offline tests | Kubeconfigs, PEM/key files, vault passwords |

Inject secrets at runtime:

- **AAP:** Controller credentials → extra-vars / env
- **Local:** `secrets/local.extra_vars.yml` (gitignored) or env vars — see [docs/local-run.md](docs/local-run.md)

`.gitignore` excludes `secrets/`, `*.kubeconfig`, `*.pem`, `*.key`, vault
password files, and `collections/` (Galaxy install output).

### Credential scan (repo hygiene)

A scan of tracked content finds **no committed secret values** — only:

- Variable *names* and Jinja references (`{{ pg_*_password }}`, `k8s_api_key`)
- Documentation of credential *types*
- Offline tests that assert presence of key names / forbid hardcoded passwords

Re-check before every push:

```bash
# From repo root — should print nothing suspicious as VALUES
rg -n --glob '!.git/**' --glob '!collections/**' \
  -e '-----BEGIN' \
  -e 'oc login --token=' \
  -e 'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+' \
  -e 'password:\s*["'\''][^"'\''${][^"'\'']{4,}' \
  .
```

## Quick start (offline)

```bash
cd ao-ea-provision
python3 -m venv .venv && source .venv/bin/activate
pip install 'ansible-core>=2.15' kubernetes
ansible-galaxy collection install -r requirements.yml

# Syntax-check orchestration + JT playbooks
for p in playbooks/{clean_install,update_release,preflight,teardown_app,ensure_db,ensure_secrets,ensure_operator,apply_ao_cr,postconfig,health_check}.yml; do
  ansible-playbook --syntax-check "$p" || exit 1
done

# Offline guard / contract tests (no cluster)
ansible-playbook tests/teardown_guards.yml
ansible-playbook tests/preflight_channel.yml
# …or run the whole tests/ tree:
for t in tests/*.yml; do ansible-playbook "$t" || exit 1; done
```

`redhat.openshift` is optional here (playbooks use `kubernetes.core`); on
AAP, install it from Automation Hub in the EE if you need it later.

## Quick start

See **[docs/local-run.md](docs/local-run.md)** for the full guide. Short form:

```bash
# 1. Put secrets in a gitignored file (never commit)
cp docs/examples/local.extra_vars.example.yml secrets/local.extra_vars.yml
# edit secrets/local.extra_vars.yml — token + PG passwords only

# 2. Prefer a mode wrapper (needs pg_execution_plane_password when script nodes on)
ansible-playbook playbooks/clean_install.yml -e @secrets/local.extra_vars.yml \
  -e ao_channel=early-access
# or: playbooks/update_release.yml … -e ao_channel=stable
```

## Hard rules

1. Never delete `db_ns`, PVC `pg_pvc_name`, or StatefulSet data in the weekly
   path (`teardown_app`). Full wipe is only `teardown_full.yml` with
   `-e destroy_confirm=DESTROY-AO-AND-DB`.
2. Never create a second Operator Subscription; reconcile channel on the existing one.
3. Do not log tokens, PG passwords, or admin password plaintext.
4. Prefer AAP for production; local runs are for debug and first bring-up.
5. On `update_release` (`provision_mode=update`), `ensure_db` may recreate **only**
   the `execution_plane` database (`ep_reset_on_update=true`) so early-access EP
   migrations are not blocked by a stale alembic stamp. Force anytime with
   `-e ep_force_reset=true`. Never drop `orchestrator` / `temporal` that way.
6. After Ready, sync the admin login password from
   `{ao}-initial-admin-password` into the DB (`admin_password_sync=true`).
   Skip with `-e admin_password_sync=false`. Never print the value.

## Layout

```text
playbooks/          JT entrypoints
roles/              ao_ea_* implementation
inventory/          hosts + group_vars (non-secret defaults)
tests/              offline ansible-playbook contracts
docs/               AAP, AO workflow, local run
secrets/            gitignored — your local extra-vars only
requirements.yml    collections (kubernetes.core; redhat.openshift Hub)
```
