# ao_ea_postconfig

Make script nodes effective on the early-access Automation Orchestrator
worker. Job Template: `ea-postconfig`.

## Chosen method

The operator sets `APP_SCRIPT_NODES_ENABLED=true` on `{ao_name}-worker`
**only when Execution Plane is enabled**. That flag is not a public
`AutomationOrchestrator` CR field. The reconciler reads
`{ao_name}-admin-settings` (`ENABLE_EXECUTION_PLANE=true`) and then SSA
applies the worker env.

This role therefore:

1. Detects whether the worker already has `APP_SCRIPT_NODES_ENABLED=true`.
2. If a live CR field `spec.executionPlane` exists, sets `enabled: true`
   (forward-compatible; current CRD has no such field).
3. Else, if Secret `{ao_name}-execution-plane-database` (or
   `EXECUTION_PLANE_DATABASE_SECRET`) is present with `database` /
   `username` / `password`, applies `ENABLE_EXECUTION_PLANE=true` on the
   admin-settings ConfigMap so the operator enables EP.
4. Waits for the reconciler to set the worker env.
5. Otherwise **fails** with the limitation below. It does **not** patch
   the worker Deployment. Operator SSA would revert that env on the next
   reconcile.

Putting `APP_SCRIPT_NODES_ENABLED` in admin-settings `envFrom` without
`ENABLE_EXECUTION_PLANE` is not sufficient: script nodes need the
Execution Plane, and the operator will not treat that key as EP opt-in.

## Limitation (when EP cannot be enabled here)

Early-access weekly provision does not create the Execution Plane
database or Secret (`ea-ensure-db` / `ea-ensure-secrets` only cover
orchestrator + temporal). Enabling EP without that Secret makes
`ConfigurationValid` fail (`Execution Plane database secret not found`).

Reconciler-safe admin path:

1. Create the Execution Plane database/role on the BYO PostgreSQL.
2. Create Secret `{ao_name}-execution-plane-database` in `app_ns` with
   keys `database`, `username`, `password`.
3. Create or update ConfigMap `{ao_name}-admin-settings` with
   `ENABLE_EXECUTION_PLANE=true`.
4. Re-run `ea-postconfig`. The operator then sets the worker env.

## Skip

`script_nodes_enabled=false` is a no-op: `script_nodes_effective=false`,
no cluster mutation.

## Verification

```bash
oc get deploy -n "$app_ns" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{.env[*].name}{"="}{.env[*].value}{"\n"}{end}{end}' | grep APP_SCRIPT_NODES_ENABLED
```
