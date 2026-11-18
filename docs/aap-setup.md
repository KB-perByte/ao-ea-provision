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

## Collections

`requirements.yml` lists `kubernetes.core` (>=5.0.0) and `redhat.openshift`
(>=1.0.0). `kubernetes.core` installs from Galaxy. `redhat.openshift` is
published on Red Hat Automation Hub, not public Galaxy — install it in the
AAP Execution Environment (or with a Hub token), not via
`ansible-galaxy` against galaxy.ansible.com.
