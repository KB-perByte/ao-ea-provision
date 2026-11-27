# ao_ea_postconfig

Make script nodes effective on the early-access Automation Orchestrator
worker. Job Template: `ea-postconfig`.

## Chosen method

When Execution Plane is enabled, the operator updates `{ao_name}-worker`.
That opt-in is not a public `AutomationOrchestrator` CR field. The
reconciler reads `{ao_name}-admin-settings`
(`ENABLE_EXECUTION_PLANE=true`) and SSA-applies worker env.

Success signals on the worker (either is enough):

- `APP_SCRIPT_NODES_ENABLED=true` (newer operator builds)
- `APP_EP_API_URL` set (some early-access CSVs omit the former)

With `script_nodes_enabled=true` (default), earlier playbooks prepare EP:

- `ea-ensure-db` — PG role/database `execution_plane`
- `ea-ensure-secrets` — Secret `{ao_name}-execution-plane-database`

This role then:

1. Detects whether the worker already has an EP success signal.
2. If a live CR field `spec.executionPlane` exists, sets `enabled: true`
   (forward-compatible; current CRD has no such field).
3. Else, if Secret `{ao_name}-execution-plane-database` is present with
   `database` / `username` / `password`, applies `ENABLE_EXECUTION_PLANE=true`
   on the admin-settings ConfigMap so the operator enables EP.
4. Waits for the reconciler to set a worker EP signal.
5. Otherwise **fails** with a clear message. It does **not** patch the
   worker Deployment (operator SSA would revert that env).

## Skip

`script_nodes_enabled=false` is a no-op: `script_nodes_effective=false`,
no cluster mutation. Pass the same flag on ensure_db / ensure_secrets to
skip creating the EP role and Secret.

## Verification

```bash
oc get deploy "$ao_name"-worker -n "$app_ns" -o jsonpath='{range .spec.template.spec.containers[*].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep -E 'APP_SCRIPT_NODES_ENABLED|APP_EP_API_URL'
oc get deploy -n "$app_ns" | grep execution-plane
```
