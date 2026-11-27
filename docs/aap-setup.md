# AAP setup (human-owned)

This repository is the AAP Project content: playbooks, roles, and inventory
vars for weekly early-access Automation Orchestrator provision on OpenShift instance.

AAP Controller bootstrap, the Execution Environment image, credential
objects, Job Template registration, and the stable AO workflow are
**human-owned**. This doc is the stitching guide: JT names, playbook
paths, extra-vars, credentials, EE collections, and timeouts.

Do not encode cluster tokens or database passwords here. Never paste
secret values into Job Template extra-vars JSON, surveys, or this repo.

Stable AO workflow node order lives in [ao-workflow.md](ao-workflow.md).

## Job Templates

Create all eight in the same Organization, Project (`ao-ea-provision`),
Inventory (`OpenShift-instance` / localhost), Execution Environment, and
credential set. Limit to the Project's SCM playbook paths below.

| JT name | Playbook |
|---|---|
| `ea-preflight` | `playbooks/preflight.yml` |
| `ea-teardown-app` | `playbooks/teardown_app.yml` |
| `ea-ensure-db` | `playbooks/ensure_db.yml` |
| `ea-ensure-secrets` | `playbooks/ensure_secrets.yml` |
| `ea-ensure-operator` | `playbooks/ensure_operator.yml` |
| `ea-apply-ao-cr` | `playbooks/apply_ao_cr.yml` |
| `ea-postconfig` | `playbooks/postconfig.yml` |
| `ea-health-check` | `playbooks/health_check.yml` |
| `ea-sync-admin-password` | `playbooks/sync_admin_password.yml` |

Optional survey on launch: `ao_channel` (default `early-access`). Do not
survey for tokens or passwords.

`playbooks/safety_assert.yml` is a local guard wrapper, not a Job
Template.

## Credentials (no secret values)

Attach the same credentials to every JT. Inject secrets via Controller
credentials, not Git or extra-vars files in this repo.

| Credential | Type | Injects | Used by |
|---|---|---|---|
| OpenShift instance OpenShift | Kubernetes / OpenShift (API URL + token) | `K8S_AUTH_*` (or extra-vars `k8s_api_key` / `k8s_kubeconfig`) | all eight JTs (`host` is always `cluster_api`) |
| PostgreSQL admin | Custom / Vault | `pg_admin_password` or `vault_pg_admin_password` | `ea-ensure-db` |
| PostgreSQL orchestrator | Custom / Vault | `pg_orchestrator_password` or `vault_pg_orchestrator_password` | `ea-ensure-db`, `ea-ensure-secrets` |
| PostgreSQL temporal | Custom / Vault | `pg_temporal_password` or `vault_pg_temporal_password` | `ea-ensure-db`, `ea-ensure-secrets` |
| PostgreSQL execution plane | Custom / Vault | `pg_execution_plane_password` or `vault_pg_execution_plane_password` | `ea-ensure-db`, `ea-ensure-secrets` (when `script_nodes_enabled=true`) |

The OpenShift credential's host/API URL must match `cluster_api`. The
token must be able to list Nodes and mutate `app_ns` / `db_ns` as the
playbooks require.

PG role passwords on `ea-ensure-secrets` **must** match the roles
`ea-ensure-db` created. A mismatch fails as `password mismatch / DB
reuse`.

## extra_vars

Non-secret OpenShift instance defaults live in `inventory/group_vars/all.yml`.
Pass the same map into every JT (or rely on the Project inventory).
Override per launch only when needed.

| extra-var | Default | Notes |
|---|---|---|
| `cluster_api` | `https://api.OpenShift-instance.rnil.p3.openshiftapps.com:443` | OpenShift instance API |
| `app_ns` | `syntarogation-ao` | App project (CR torn down weekly; ns recreated as needed) |
| `db_ns` | `syntarogation-ao-db` | Persistent DB project |
| `ao_name` | `my-orchestrator` | `AutomationOrchestrator` CR name |
| `pg_service_host` | `ao-postgres.syntarogation-ao-db.svc` | BYO PG DNS |
| `pg_service_name` | `ao-postgres` | StatefulSet / Service |
| `route_host` | `syntarogation-ao.apps.rosa.OpenShift-instance.rnil.p3.openshiftapps.com` | Ingress host |
| `ao_channel` | `early-access` | Target OLM channel (`early-access` or `stable`); ensure_operator reconciles the Subscription |
| `ep_reset_on_update` | `true` | On `update_release` (`provision_mode=update`), recreate only the `execution_plane` DB before CR reconcile |
| `ep_force_reset` | `false` | Force that EP DB recreate from any playbook (`ensure_db.yml -e ep_force_reset=true`) |
| `admin_password_sync` | `true` | After Ready, sync `{ao}-initial-admin-password` into the DB (clean install + update/recreate). Skip with `false` |
| `catalog_source` | `redhat-operators` | Marketplace catalog |
| `catalog_ns` | `openshift-marketplace` | CatalogSource namespace |
| `operator_ns` | `automation-orchestrator-operator-system` | Existing operator install |
| `operator_subscription_name` | `automation-orchestrator-operator` | Existing Subscription |
| `ao_script_python_image_reference` | `registry.redhat.io/ansible-automation-platform/ao-python-rhel9:2026.8` | Injected into the OLM Subscription for Python script steps |
| `ao_script_bash_image_reference` | `registry.redhat.io/ansible-automation-platform/ao-bash-rhel9:2026.8` | Injected into the OLM Subscription for Bash script steps |
| `script_nodes_enabled` | `true` | See [Postconfig / script nodes](#postconfig--script-nodes) |
| `pg_pvc_name` | `data-ao-postgres-0` | Never delete |

`ea-ensure-operator` applies these image references through
`Subscription.spec.config.env`, waits for the Operator manager rollout, and
fails before AO reconciliation if the live manager Deployment does not carry
both values. The refs can be changed per launch with extra-vars when the lab
moves to another published image version.

Credential extra-vars (values from AAP credentials, never committed):

| extra-var | Vault alias | Required on |
|---|---|---|
| `pg_admin_password` | `vault_pg_admin_password` | `ea-ensure-db` |
| `pg_orchestrator_password` | `vault_pg_orchestrator_password` | `ea-ensure-db`, `ea-ensure-secrets` |
| `pg_temporal_password` | `vault_pg_temporal_password` | `ea-ensure-db`, `ea-ensure-secrets` |
| `pg_execution_plane_password` | `vault_pg_execution_plane_password` | `ea-ensure-db`, `ea-ensure-secrets` when script nodes on |

Optional:

| extra-var | Default | When |
|---|---|---|
| `secrets_verify_connect` | `true` | `-e secrets_verify_connect=false` skips the PG connect check |
| `k8s_api_key` / `k8s_kubeconfig` | unset | Prefer the Kubernetes credential; do not log these |

## Execution Environment / collections

`requirements.yml` lists `kubernetes.core` (>=5.0.0) and
`redhat.openshift` (>=1.0.0).

- **`kubernetes.core`**: install from Galaxy (`galaxy.ansible.com`).
- **`redhat.openshift`**: published on Red Hat Automation Hub, not
  public Galaxy. Bake it into the AAP Execution Environment (or
  `ansible-galaxy` with a Hub token). Do **not** install it from
  galaxy.ansible.com.

Confirm on the EE before the first real run:

```bash
ansible-galaxy collection list | grep -E 'kubernetes.core|redhat.openshift'
```

Both collections must be present. `ea-preflight` uses
`kubernetes.core.k8s_info` so syntax-check still works if Hub is
unavailable; live OpenShift mutations expect both.

## Postconfig / script nodes

`ea-postconfig` makes script nodes effective on the EA worker. The
operator sets `APP_SCRIPT_NODES_ENABLED=true` on `{ao_name}-worker`
**only when Execution Plane is enabled**.

With `script_nodes_enabled=true` (default):

- `ea-ensure-db` creates PG role/database `execution_plane`
- `ea-ensure-secrets` applies Secret `{ao_name}-execution-plane-database`
  (keys `database`, `username`, `password`)
- `ea-postconfig` sets `ENABLE_EXECUTION_PLANE=true` and waits for
  `APP_SCRIPT_NODES_ENABLED=true` on the worker

Pass `pg_execution_plane_password` with the other PG credentials.
Do not work around failures with a worker Deployment env patch
(operator SSA reverts it).

To skip Execution Plane / script nodes for a run:

```bash
-e script_nodes_enabled=false
```

That is a no-op (`script_nodes_effective=false`). Put the same extra-var
on the JT / workflow extra_vars map when EP is not ready.

## Suggested JT timeouts

AAP Controller job timeout must exceed the playbook wait, plus a buffer
for API lag.

| JT | Suggested timeout | Why |
|---|---|---|
| `ea-preflight` | 10 min | Read-only catalog / channel checks |
| `ea-teardown-app` | 15 min | CR delete + ownerRef GC wait 600s |
| `ea-ensure-db` | 15 min | StatefulSet Ready + init SQL (~300s wait) |
| `ea-ensure-secrets` | 10 min | Secret apply + optional connect check |
| `ea-ensure-operator` | 20 min | Create on clean cluster or assert existing (wait CSV) |
| `ea-apply-ao-cr` | 10 min | Apply + ~90s `ConfigurationValid` |
| `ea-postconfig` | 10 min | EP opt-in + ~3 min worker-env wait |
| `ea-health-check` | **> 25 min** | Ready wait 1200s (~20 min) plus HTTPS `/health` retries (~4 min) |

Default Controller timeouts that are 1 hour are fine. Anything at or
below 20 minutes will fail `ea-health-check` on a cold apply.

## Teardown policy (`ea-teardown-app`)

Playbook: `playbooks/teardown_app.yml`.

v1 **keeps** `app_ns`. It deletes `AutomationOrchestrator` `ao_name` in
`app_ns` and waits for ownerRef garbage collection. It does **not** delete
Namespace `db_ns`, PVC `pg_pvc_name`, StatefulSet `pg_service_name`, or
`app_ns` itself.

Guards are var-driven in `roles/ao_ea_common` (`teardown_target_ns`,
`teardown_delete_namespaces`, `teardown_delete_pvcs`,
`teardown_delete_statefulsets`). The teardown role feeds those lists before
any `k8s` `state=absent` and refuses `app_ns == db_ns`.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-teardown-app
oc get automationorchestrator -n syntarogation-ao
# expected: no my-orchestrator (or empty list)

oc get pvc -n syntarogation-ao-db
# expected: data-ao-postgres-0 Bound

oc get statefulset ao-postgres -n syntarogation-ao-db
# expected: still present

# Re-run JT: success / no-op; PVC still Bound
```

## Ensure DB (`ea-ensure-db`)

Playbook: `playbooks/ensure_db.yml`.

Ensures Namespace `db_ns`, StatefulSet `pg_service_name` (quay SCLORG
PostgreSQL 15, `POSTGRESQL_MAX_CONNECTIONS=200`), Service `pg_service_name`,
and PVC `pg_pvc_name` via `volumeClaimTemplates` (`gp3-csi`, 20Gi). **Never
deletes** that PVC. If it is already Bound, the play only ensures
StatefulSet/Service and pod Ready.

Init SQL (idempotent): roles `orchestrator` / `temporal` (and
`execution_plane` when `script_nodes_enabled=true`); databases
`orchestrator`, `temporal`, `temporal_visibility` (and `execution_plane`);
`CREATE EXTENSION pgcrypto` as user `orchestrator` (not `postgres`).

Produces facts `pg_host`, `pvc_bound`, `db_ready`.

Passwords **must** come from AAP credentials as extra-vars (never commit
real values):

| extra-var | vault alias | Used for |
|---|---|---|
| `pg_admin_password` | `vault_pg_admin_password` | SCLORG `POSTGRESQL_ADMIN_PASSWORD` (first boot) |
| `pg_orchestrator_password` | `vault_pg_orchestrator_password` | role `orchestrator` |
| `pg_temporal_password` | `vault_pg_temporal_password` | role `temporal` |
| `pg_execution_plane_password` | `vault_pg_execution_plane_password` | role `execution_plane` (script nodes) |

AAP Job Template creation is human-owned.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-ensure-db
oc get pvc -n syntarogation-ao-db
# expected: data-ao-postgres-0 Bound (same volumeName as before)

oc get statefulset,svc,pod -n syntarogation-ao-db
# expected: ao-postgres 1/1; pod Ready

# Re-run JT: success / no-op; PVC still Bound; same volumeName
```

## Ensure secrets (`ea-ensure-secrets`)

Playbook: `playbooks/ensure_secrets.yml`.

Ensures Namespace `app_ns` and applies Secrets `orchestrator-pg-credentials`
and `temporal-pg-credentials` in that namespace. When
`script_nodes_enabled=true`, also applies
`{ao_name}-execution-plane-database`. Keys match the AO CR / EP schema
(`database`, `username`, `password`). Users/databases match the roles
`ea-ensure-db` creates.

Passwords **must** come from AAP credentials as extra-vars and **must**
match `ea-ensure-db` (never commit real values):

| extra-var | vault alias | Secret / key |
|---|---|---|
| `pg_orchestrator_password` | `vault_pg_orchestrator_password` | `orchestrator-pg-credentials` / `password` |
| `pg_temporal_password` | `vault_pg_temporal_password` | `temporal-pg-credentials` / `password` |
| `pg_execution_plane_password` | `vault_pg_execution_plane_password` | `{ao}-execution-plane-database` / `password` |

Optional read-only connect: exec `psql` in the PostgreSQL pod as those
users (TCP to localhost so `pg_hba` uses password auth). Auth failure
fails the play with `password mismatch / DB reuse`.

**Offline / unreachable PG:** skip the connect check with
`-e secrets_verify_connect=false`. Secrets are still applied. Default is
`true` for the live JT.

AAP Job Template creation is human-owned.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-ensure-secrets
oc get secret orchestrator-pg-credentials temporal-pg-credentials \
  -n syntarogation-ao
# expected: both present (Opaque)

oc get secret orchestrator-pg-credentials -n syntarogation-ao \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
# expected: database / password / username (keys only; do not extract values)

# Re-run JT: success / no-op; same secret names
```

## Orchestration wrappers (local / full run)

| Playbook | Mode | Keeps DB | Keeps AO CR |
|---|---|---|---|
| `playbooks/update_release.yml` | update | yes | yes (ends with admin password sync) |
| `playbooks/clean_install.yml` | clean | creates / never deletes PVC | creates (ends with admin password sync) |

Pass `-e ao_channel=early-access` or `-e ao_channel=stable`. Optional
weekly wipe: `teardown_app` then `update_release`.

## Ensure operator (`ea-ensure-operator`)

Playbook: `playbooks/ensure_operator.yml`.

**Clean cluster (no AO Subscription):** creates `operator_ns`, OperatorGroup,
and Subscription from `catalog_source` / `ao_channel` (default
`early-access`, `installPlanApproval` from
`operator_install_plan_approval`, default `Automatic`), then waits for
`AtLatestKnown` + CSV `Succeeded`.

**Existing Subscription:** with `operator_manage_channel=true` (default),
reconciles `spec.channel` (and `installPlanApproval`) to `ao_channel`,
waits until state `AtLatestKnown` and CSV `Succeeded`. Never creates a
second Subscription. Still refuses duplicates (`ao_subscription_count > 1`).

Produces facts `installed_csv`, `channel_ok`, `operator_created`, and
`operator_channel_reconciled`.

AAP Job Template creation is human-owned.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-ensure-operator
oc get subscription automation-orchestrator-operator \
  -n automation-orchestrator-operator-system \
  -o custom-columns=CHANNEL:.spec.channel,SOURCE:.spec.source,CSV:.status.installedCSV,STATE:.status.state
# expected: CHANNEL matches ao_channel; STATE=AtLatestKnown
```

## Apply AO CR (`ea-apply-ao-cr`)

Playbook: `playbooks/apply_ao_cr.yml`.

Applies `AutomationOrchestrator` `ao_name` in `app_ns` (BYO PostgreSQL +
OpenShift Route). `postgres.host` is `pg_service_host`. `secretRef` names
are `orchestrator-pg-credentials` / `temporal-pg-credentials` (same Secrets
`ea-ensure-secrets` creates). `ingress.type` is `Route`; host is
`route_host`. `sslMode: disable` matches the SCLORG PostgreSQL in `db_ns`.

Waits **briefly** (~90s) for operator `ConfigurationValid=True` (spec
Accepted). It does **not** wait for `Ready=True` — that is
`ea-health-check` (15–20m).

Produces facts `cr_applied` and `cr_accepted` (`false` until apply and
ConfigurationValid succeed).

AAP Job Template creation is human-owned.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-apply-ao-cr
oc get automationorchestrator my-orchestrator -n syntarogation-ao
# expected: CR present. ConfigurationValid=True. Ready may still be False
# (ea-health-check waits for Ready).

oc get automationorchestrator my-orchestrator -n syntarogation-ao \
  -o jsonpath='{.spec.postgres.host}{"\n"}{.spec.ingress.host}{"\n"}'
# expected: ao-postgres.syntarogation-ao-db.svc
#           syntarogation-ao.apps.rosa.OpenShift-instance.rnil.p3.openshiftapps.com

# Re-run JT: success / no-op; same CR name and postgres.host
```

## Postconfig script nodes (`ea-postconfig`)

Playbook: `playbooks/postconfig.yml`.

Makes script nodes effective on the EA worker. The operator sets
`APP_SCRIPT_NODES_ENABLED=true` on `{ao_name}-worker` **only when
Execution Plane is enabled**. That is not a public CR field. The
reconciler-safe opt-in is ConfigMap `{ao_name}-admin-settings` with
`ENABLE_EXECUTION_PLANE=true`, which also requires Secret
`{ao_name}-execution-plane-database` (keys `database`, `username`,
`password`) and an Execution Plane database on the BYO PostgreSQL.

This playbook:

1. Detects whether the worker already has the env (EP already on).
2. If the live CR has `spec.executionPlane`, sets `enabled: true`.
3. Else, if the EP database Secret is already present, applies
   `ENABLE_EXECUTION_PLANE=true` and waits for the reconciler.
4. Otherwise **fails** with that limitation. It does **not** patch the
   worker Deployment env (operator Server-Side Apply would revert it).

`script_nodes_enabled=false` is a no-op (`script_nodes_effective=false`).

Produces fact `script_nodes_effective` (`false` until the worker env is
`true`).

AAP Job Template creation is human-owned.

`ea-ensure-db` / `ea-ensure-secrets` create the Execution Plane database
and Secret when `script_nodes_enabled=true`. Postconfig then enables EP.
Do not work around failures with a Deployment env patch.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-postconfig (success)
oc get deploy -n syntarogation-ao -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{.env[*].name}{"="}{.env[*].value}{"\n"}{end}{end}' | grep APP_SCRIPT_NODES_ENABLED
# expected: APP_SCRIPT_NODES_ENABLED=true on my-orchestrator-worker

oc get configmap my-orchestrator-admin-settings -n syntarogation-ao \
  -o jsonpath='{.data.ENABLE_EXECUTION_PLANE}{"\n"}'
# expected: true when this JT enabled EP

# Re-run JT: success / no-op when the worker env is already true
```

## Health check (`ea-health-check`)

Playbook: `playbooks/health_check.yml`.

Waits for `AutomationOrchestrator` `ao_name` in `app_ns` to reach
`Ready=True` (~15–20 minutes; `wait_timeout` 1200s). Then asserts:

1. Key Deployments have `availableReplicas >= 1`: `{ao_name}-backend`,
   `-worker`, `-background-worker`, `-ui`, `-redis`, `-temporal`.
2. A Route exists whose `spec.host` is `route_host`.
3. HTTPS GET `https://{route_host}/health` returns 200 (nginx UI health
   on the Route, with retries). Backend `/healthz/ready` is not on the
   Route (nginx only proxies `/api`).
4. Secret `{ao_name}-initial-admin-password` exists in `app_ns` with
   key `password`. **Presence only — the value is never printed.**

Produces facts `cr_ready`, `pods_ready`, `route_ok`, `http_probe_ok`,
`route_url`. Fail-closed until each check succeeds. Missing Route fails
clearly (`Route down`).

AAP Job Template creation is human-owned. This playbook is read-only.

Cluster verification (human, OpenShift instance):

```bash
# After JT ea-health-check
oc get automationorchestrator my-orchestrator -n syntarogation-ao
# expected: READY True

oc get deploy -n syntarogation-ao
# expected: my-orchestrator-{backend,worker,background-worker,ui,redis,temporal} available

oc get route -n syntarogation-ao
# expected: host syntarogation-ao.apps.rosa.OpenShift-instance.rnil.p3.openshiftapps.com

curl -sS -o /dev/null -w "%{http_code}\n" \
  https://syntarogation-ao.apps.rosa.OpenShift-instance.rnil.p3.openshiftapps.com/health
# expected: 200

oc get secret my-orchestrator-initial-admin-password \
  -n syntarogation-ao \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
# expected: password (key only; do not extract the value)
```

## Sync admin password (`ea-sync-admin-password`)

Playbook: `playbooks/sync_admin_password.yml`.

After Ready, runs `ao-admin reset-password` in a backend pod using the
operator Secret `{ao_name}-initial-admin-password` (key `password`).
Needed when `app_ns`/CR is recreated against a persistent `orchestrator`
DB (seed only sets the password on first admin create). Also safe on a
true fresh install — keeps Secret and UI login aligned. Default on in
`clean_install.yml` and `update_release.yml`
(`admin_password_sync=true`). Never prints the value (`no_log`).

AAP Job Template creation is human-owned.

## Collections

See [Execution Environment / collections](#execution-environment--collections).
`kubernetes.core` from Galaxy; `redhat.openshift` from Automation Hub in
the EE.
