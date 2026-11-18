# AAP setup (human-owned)

AAP Controller bootstrap, Execution Environment, credentials, Job Template
mapping, and stable AO workflow wiring are **out of scope for this Git repo
scaffold**.

Those steps are owned by the operator/human. Do not encode cluster tokens or
database passwords here.

This repository is the AAP Project content: playbooks, roles, and inventory
vars for weekly early-access Automation Orchestrator provision on PDE AO-05.

## Collections

`requirements.yml` lists `kubernetes.core` (>=5.0.0) and `redhat.openshift`
(>=1.0.0). `kubernetes.core` installs from Galaxy. `redhat.openshift` is
published on Red Hat Automation Hub, not public Galaxy — install it in the
AAP Execution Environment (or with a Hub token), not via
`ansible-galaxy` against galaxy.ansible.com.
