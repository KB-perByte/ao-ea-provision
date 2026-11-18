# AAP setup (human-owned)

AAP Controller bootstrap, Execution Environment, credentials, Job Template
mapping, and stable AO workflow wiring are **out of scope for this Git repo
scaffold**.

Those steps are owned by the operator/human. Do not encode cluster tokens or
database passwords here.

This repository is the AAP Project content: playbooks, roles, and inventory
vars for weekly early-access Automation Orchestrator provision on PDE AO-05.

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

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-teardown-app
oc get automationorchestrator -n syntarogation-ao-nov
# expected: no my-orchestrator (or empty list)

oc get pvc -n syntarogation-ao-nov-db
# expected: data-ao-postgres-0 Bound

oc get statefulset ao-postgres -n syntarogation-ao-nov-db
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

Init SQL (idempotent): roles `orchestrator` / `temporal`; databases
`orchestrator`, `temporal`, `temporal_visibility`; `CREATE EXTENSION pgcrypto`
as user `orchestrator` (not `postgres`).

Produces facts `pg_host`, `pvc_bound`, `db_ready`.

Passwords **must** come from AAP credentials as extra-vars (never commit
real values):

| extra-var | vault alias | Used for |
|---|---|---|
| `pg_admin_password` | `vault_pg_admin_password` | SCLORG `POSTGRESQL_ADMIN_PASSWORD` (first boot) |
| `pg_orchestrator_password` | `vault_pg_orchestrator_password` | role `orchestrator` |
| `pg_temporal_password` | `vault_pg_temporal_password` | role `temporal` |

AAP Job Template creation is human-owned.

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-ensure-db
oc get pvc -n syntarogation-ao-nov-db
# expected: data-ao-postgres-0 Bound (same volumeName as before)

oc get statefulset,svc,pod -n syntarogation-ao-nov-db
# expected: ao-postgres 1/1; pod Ready

# Re-run JT: success / no-op; PVC still Bound; same volumeName
```

## Ensure secrets (`ea-ensure-secrets`)

Playbook: `playbooks/ensure_secrets.yml`.

Ensures Namespace `app_ns` and applies Secrets `orchestrator-pg-credentials`
and `temporal-pg-credentials` in that namespace. Keys match the AO CR
`secretRef` schema (`database`, `username`, `password`). Users/databases
are `orchestrator` / `temporal` — the same roles `ea-ensure-db` creates.

Passwords **must** come from AAP credentials as extra-vars and **must**
match `ea-ensure-db` (never commit real values):

| extra-var | vault alias | Secret / key |
|---|---|---|
| `pg_orchestrator_password` | `vault_pg_orchestrator_password` | `orchestrator-pg-credentials` / `password` |
| `pg_temporal_password` | `vault_pg_temporal_password` | `temporal-pg-credentials` / `password` |

Optional read-only connect: exec `psql` in the PostgreSQL pod as those
users (TCP to localhost so `pg_hba` uses password auth). Auth failure
fails the play with `password mismatch / DB reuse`.

**Offline / unreachable PG:** skip the connect check with
`-e secrets_verify_connect=false`. Secrets are still applied. Default is
`true` for the live JT.

AAP Job Template creation is human-owned.

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-ensure-secrets
oc get secret orchestrator-pg-credentials temporal-pg-credentials \
  -n syntarogation-ao-nov
# expected: both present (Opaque)

oc get secret orchestrator-pg-credentials -n syntarogation-ao-nov \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
# expected: database / password / username (keys only; do not extract values)

# Re-run JT: success / no-op; same secret names
```

## Ensure operator (`ea-ensure-operator`)

Playbook: `playbooks/ensure_operator.yml`.

**Asserts** the existing OperatorHub Subscription in `operator_ns`. It does
**not** create or patch a Subscription in v1.

Checks:

1. Subscription `operator_subscription_name` exists in `operator_ns`.
2. `spec.channel == ao_channel` (inventory default `early-access`; honor
   extra-var override).
3. `status.state` is `AtLatestKnown` and the installed CSV phase is
   `Succeeded`.
4. Exactly one Subscription for package `automation-orchestrator-operator`
   cluster-wide.

Channel mismatch (example: `-e ao_channel=stable` while the cluster is on
`early-access`) **fails clearly**. Do not patch the Subscription; either
set `ao_channel` to the installed channel, or have an operator change the
existing Subscription outside this playbook.

Produces facts `installed_csv` and `channel_ok` (`false` until asserts
pass).

AAP Job Template creation is human-owned.

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-ensure-operator
oc get subscription automation-orchestrator-operator \
  -n automation-orchestrator-operator-system \
  -o custom-columns=CHANNEL:.spec.channel,SOURCE:.spec.source,CSV:.status.installedCSV,STATE:.status.state
# expected: CHANNEL=early-access; STATE=AtLatestKnown

# Negative: launch JT with ao_channel=stable while the cluster is
# early-access — expect fail; Subscription must be unchanged
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

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-apply-ao-cr
oc get automationorchestrator my-orchestrator -n syntarogation-ao-nov
# expected: CR present. ConfigurationValid=True. Ready may still be False
# (ea-health-check waits for Ready).

oc get automationorchestrator my-orchestrator -n syntarogation-ao-nov \
  -o jsonpath='{.spec.postgres.host}{"\n"}{.spec.ingress.host}{"\n"}'
# expected: ao-postgres.syntarogation-ao-nov-db.svc
#           syntarogation-ao-nov.apps.rosa.pde-ao-05.rnil.p3.openshiftapps.com

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

`ea-ensure-db` / `ea-ensure-secrets` do **not** create the Execution
Plane database or Secret today. On a stock weekly provision the JT
fails closed until those exist (or EP is already enabled). Do not work
around that with a Deployment env patch.

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-postconfig (success)
oc get deploy -n syntarogation-ao-nov -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{.env[*].name}{"="}{.env[*].value}{"\n"}{end}{end}' | grep APP_SCRIPT_NODES_ENABLED
# expected: APP_SCRIPT_NODES_ENABLED=true on my-orchestrator-worker

oc get configmap my-orchestrator-admin-settings -n syntarogation-ao-nov \
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

Cluster verification (human, PDE AO-05):

```bash
# After JT ea-health-check
oc get automationorchestrator my-orchestrator -n syntarogation-ao-nov
# expected: READY True

oc get deploy -n syntarogation-ao-nov
# expected: my-orchestrator-{backend,worker,background-worker,ui,redis,temporal} available

oc get route -n syntarogation-ao-nov
# expected: host syntarogation-ao-nov.apps.rosa.pde-ao-05.rnil.p3.openshiftapps.com

curl -sS -o /dev/null -w "%{http_code}\n" \
  https://syntarogation-ao-nov.apps.rosa.pde-ao-05.rnil.p3.openshiftapps.com/health
# expected: 200

oc get secret my-orchestrator-initial-admin-password \
  -n syntarogation-ao-nov \
  -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
# expected: password (key only; do not extract the value)
```

## Collections

`requirements.yml` lists `kubernetes.core` (>=5.0.0) and `redhat.openshift`
(>=1.0.0). `kubernetes.core` installs from Galaxy. `redhat.openshift` is
published on Red Hat Automation Hub, not public Galaxy — install it in the
AAP Execution Environment (or with a Hub token), not via
`ansible-galaxy` against galaxy.ansible.com.
