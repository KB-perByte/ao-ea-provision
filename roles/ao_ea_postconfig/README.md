# ao_ea_postconfig

Make script nodes effective on the early-access Automation Orchestrator
worker. Job Template: `ea-postconfig`.

## Chosen method

The operator sets `APP_SCRIPT_NODES_ENABLED=true` on `{ao_name}-worker`
**only when Execution Plane is enabled**. That flag is not a public
`AutomationOrchestrator` CR field. The reconciler reads
`{ao_name}-admin-settings` (`ENABLE_EXECUTION_PLANE=true`) and then SSA
applies the worker env.

With `script_nodes_enabled=true` (default), earlier playbooks prepare EP:

- `ea-ensure-db` — PG role/database `execution_plane`
- `ea-ensure-secrets` — Secret `{ao_name}-execution-plane-database`

This role then:

1. Detects whether the worker already has `APP_SCRIPT_NODES_ENABLED=true`.
2. If a live CR field `spec.executionPlane` exists, sets `enabled: true`
   (forward-compatible; current CRD has no such field).
3. Else, if Secret `{ao_name}-execution-plane-database` is present with
   `database` / `username` / `password`, applies `ENABLE_EXECUTION_PLANE=true`
   on the admin-settings ConfigMap so the operator enables EP.
4. Waits for the reconciler to set the worker env.
5. Otherwise **fails** with a clear message. It does **not** patch the
   worker Deployment (operator SSA would revert that env).

## Skip

`script_nodes_enabled=false` is a no-op: `script_nodes_effective=false`,
no cluster mutation. Pass the same flag on ensure_db / ensure_secrets to
skip creating the EP role and Secret.

## Verification

```bash
oc get deploy -n "$app_ns" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{.env[*].name}{"="}{.env[*].value}{"\n"}{end}{end}' | grep APP_SCRIPT_NODES_ENABLED
```
