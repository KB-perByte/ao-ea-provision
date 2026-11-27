# Stable AO workflow `ea-ao-weekly-provision`

Human-owned. Create this workflow on the **stable** Automation
Orchestrator (script nodes enabled there), pointing at the AAP
Controller Job Templates in [aap-setup.md](aap-setup.md).

Do not check a workflow JSON with secrets into this repo. Do not log
OCP tokens, PostgreSQL passwords, or the EA initial-admin password in
script-node output, AO run summaries, or AAP job logs you paste into
tickets.

## Prerequisites

1. AAP Project `ao-ea-provision` synced; eight JTs registered
   ([aap-setup.md](aap-setup.md)).
2. Stable AO Integration → that AAP Controller (URL + auth). Record
   the Integration ID for `aap_job_template` nodes.
3. Stable AO worker has `APP_SCRIPT_NODES_ENABLED=true` (Execution
   Plane enabled, or the supported admin path for that install).
4. OpenShift instance Kubernetes credential and PG password credentials attached
   to every JT.

## Node order

Pass the **same extra_vars map** into every `aap_job_template` node.

| # | Node | Target | Role |
|---|---|---|---|
| 1 | Script | — | Normalize inputs. Set `ao_channel` to the workflow input or `"early-access"`. Pass through the fixed OpenShift vars. No secrets. |
| 2 | `aap_job_template` | `ea-preflight` | API, CatalogSource, channel available |
| 3 | `aap_job_template` | `ea-teardown-app` | Delete EA CR; keep DB ns / PVC / StatefulSet |
| 4 | `aap_job_template` | `ea-ensure-db` | Ensure PG; never delete the PVC |
| 5 | `aap_job_template` | `ea-ensure-secrets` | Recreate app-ns PG Secrets from AAP creds |
| 6 | `aap_job_template` | `ea-ensure-operator` | Install operator if missing; else assert `channel == ao_channel` |
| 7 | `aap_job_template` | `ea-apply-ao-cr` | Apply `AutomationOrchestrator` (BYO PG + Route) |
| 8 | `aap_job_template` | `ea-postconfig` | Script nodes on EA, or skip (see below) |
| 9 | `aap_job_template` | `ea-health-check` | Ready, Deployments, Route, HTTPS `/health` |
| 10 | Script | — | Summary: `ao_channel`, `route_url`, ready / probe flags. **No secrets.** |

Nodes 2–9 are sequential. Do not parallelize teardown with ensure-db.
`ea-health-check` is read-only; resume from it after a health timeout
without re-running teardown.

## extra_vars map

Non-secret defaults (same as `inventory/group_vars/all.yml`). Copy this
into the workflow (or each JT extra_vars) and override only
`ao_channel` / `script_nodes_enabled` per run:

```yaml
cluster_api: "https://api.OpenShift-instance.rnil.p3.openshiftapps.com:443"
app_ns: syntarogation-ao
db_ns: syntarogation-ao-db
ao_name: my-orchestrator
pg_service_host: ao-postgres.syntarogation-ao-db.svc
pg_service_name: ao-postgres
route_host: syntarogation-ao.apps.rosa.OpenShift-instance.rnil.p3.openshiftapps.com
ao_channel: early-access
catalog_source: redhat-operators
catalog_ns: openshift-marketplace
operator_ns: automation-orchestrator-operator-system
operator_subscription_name: automation-orchestrator-operator
script_nodes_enabled: true
pg_pvc_name: data-ao-postgres-0
```

PG passwords and the OpenShift token stay on AAP credentials. Do **not**
put `pg_*`, `vault_pg_*`, `k8s_api_key`, or kubeconfig contents in this
map or in script-node variables.

### `ao_channel`

| | |
|---|---|
| Default | `early-access` (inventory + script node 1) |
| Override | Workflow input / extra-var `ao_channel` (`early-access` or `stable`) |
| Behaviour | `ea-ensure-operator` reconciles the Subscription to `ao_channel` (`operator_manage_channel=true`) and waits for `AtLatestKnown`. For full local/orchestration runs prefer `playbooks/update_release.yml` or `playbooks/clean_install.yml` with the same `-e ao_channel=…`. |

### Local / AAP full-run recipes

| Goal | Command |
|---|---|
| Update stable (keep DB + CR) | `update_release.yml -e ao_channel=stable` |
| Update early-access (keep DB + CR) | `update_release.yml -e ao_channel=early-access` |
| Clean install stable | `clean_install.yml -e ao_channel=stable` |
| Clean install early-access | `clean_install.yml -e ao_channel=early-access` |
| Weekly wipe app, keep DB | `teardown_app.yml` then `update_release.yml` |
| Repair stale EP alembic only | `ensure_db.yml -e ep_force_reset=true` then `apply_ao_cr` / `health_check` |
| Repair admin login vs Secret | `sync_admin_password.yml` (also runs at end of clean/update) |

`update_release.yml` sets `provision_mode=update`, so `ensure_db` recreates
only the `execution_plane` database by default (`ep_reset_on_update=true`)
before the CR reconcile. That avoids EP `MigrationFailed` when an early-access
image cannot locate a prior alembic revision (e.g. `n0k1l2m3n4o5`).

Both `clean_install.yml` and `update_release.yml` finish with
`sync_admin_password` (`admin_password_sync=true`) so the UI login matches
`{ao}-initial-admin-password` after a true fresh install **and** after
teardown+recreate on a persistent `orchestrator` DB (seed only sets the
password on first admin create).

### Script nodes / Execution Plane

With `script_nodes_enabled: true` (default):

- `ea-ensure-db` creates PG role/database `execution_plane`
- `ea-ensure-secrets` applies Secret `{ao_name}-execution-plane-database`
- `ea-postconfig` sets `ENABLE_EXECUTION_PLANE` and waits for
  `APP_SCRIPT_NODES_ENABLED=true` on the worker

Pass `pg_execution_plane_password` (or `vault_pg_execution_plane_password`)
via AAP credentials alongside the other PG passwords. Set
`script_nodes_enabled: false` only to skip EP entirely.
Details: [aap-setup.md](aap-setup.md).

## Do not log secrets

Script nodes are glue only: defaults and a non-secret summary.

Never print or persist:

- OpenShift API tokens (`k8s_api_key`, `K8S_AUTH_*`)
- PostgreSQL passwords (`pg_admin_password`, `pg_orchestrator_password`,
  `pg_temporal_password`, `vault_pg_*`)
- Secret `.data` values (`oc extract`, `jsonpath='{.data.password}'`)
- `{ao_name}-initial-admin-password` (health check asserts **presence**
  only; `sync_admin_password` applies it with `no_log` and never prints it)

`ea-health-check` already suppresses that Secret's value. Keep AO
script output to `ao_channel`, `route_url`, and boolean facts
(`cr_ready`, `pods_ready`, `route_ok`, `http_probe_ok`,
`script_nodes_effective`, `admin_password_synced`).
