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
4. PDE AO-05 Kubernetes credential and PG password credentials attached
   to every JT.

## Node order

Pass the **same extra_vars map** into every `aap_job_template` node.

| # | Node | Target | Role |
|---|---|---|---|
| 1 | Script | — | Normalize inputs. Set `ao_channel` to the workflow input or `"early-access"`. Pass through the fixed PDE vars. No secrets. |
| 2 | `aap_job_template` | `ea-preflight` | API, CatalogSource, channel available |
| 3 | `aap_job_template` | `ea-teardown-app` | Delete EA CR; keep DB ns / PVC / StatefulSet |
| 4 | `aap_job_template` | `ea-ensure-db` | Ensure PG; never delete the PVC |
| 5 | `aap_job_template` | `ea-ensure-secrets` | Recreate app-ns PG Secrets from AAP creds |
| 6 | `aap_job_template` | `ea-ensure-operator` | Assert existing Subscription `channel == ao_channel` |
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
cluster_api: "https://api.pde-ao-05.rnil.p3.openshiftapps.com:443"
app_ns: syntarogation-ao-nov
db_ns: syntarogation-ao-nov-db
ao_name: my-orchestrator
pg_service_host: ao-postgres.syntarogation-ao-nov-db.svc
pg_service_name: ao-postgres
route_host: syntarogation-ao-nov.apps.rosa.pde-ao-05.rnil.p3.openshiftapps.com
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
| Override | Workflow input / extra-var `ao_channel` (example: `stable`) |
| Constraint | `ea-ensure-operator` asserts the **existing** Subscription channel. It does not patch OLM. If the cluster is still `early-access` and you pass `ao_channel=stable`, that JT fails clearly. Change the Subscription outside this workflow first, or leave the default. |

### Postconfig skip

`ea-postconfig` needs Execution Plane DB Secret
`{ao_name}-execution-plane-database` (or EP already enabled). On a
stock weekly provision that Secret is not created. Either create it
before node 8, or set `script_nodes_enabled: false` in the extra_vars
map so postconfig is a no-op. Details: [aap-setup.md](aap-setup.md).

## Do not log secrets

Script nodes are glue only: defaults and a non-secret summary.

Never print or persist:

- OpenShift API tokens (`k8s_api_key`, `K8S_AUTH_*`)
- PostgreSQL passwords (`pg_admin_password`, `pg_orchestrator_password`,
  `pg_temporal_password`, `vault_pg_*`)
- Secret `.data` values (`oc extract`, `jsonpath='{.data.password}'`)
- `{ao_name}-initial-admin-password` (health check asserts **presence**
  only)

`ea-health-check` already suppresses that Secret's value. Keep AO
script output to `ao_channel`, `route_url`, and boolean facts
(`cr_ready`, `pods_ready`, `route_ok`, `http_probe_ok`,
`script_nodes_effective`).
