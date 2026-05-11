# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Does

Ansible role (`dcm_deploy`) that deploys DCM (Data Center Management) as podman quadlet containers managed by systemd on RHEL 9 / Fedora. Part of the [dcm-project](https://github.com/dcm-project) org — this is the canonical production deployment path for running DCM on a RHEL 9 host without Kubernetes.

## Dependencies

Requires the `ansible.posix` collection (for `firewalld` module):
```bash
ansible-galaxy collection install ansible.posix
```

## Commands

```bash
# Core-only deployment (defaults are sufficient, no vars file needed)
ansible-playbook -i inventory/hosts main.yml

# Deployment with custom vars
ansible-playbook -i inventory/hosts --extra-vars "vars_path=vars/dcm.yml" main.yml

# Run a single deployment phase
ansible-playbook -i inventory/hosts --extra-vars "role_name=dcm_deploy" --extra-vars "task_name=validate_deployment" run_role_task.yml

# Check quadlet templates match upstream compose.yaml (runs locally, no inventory needed)
ansible-playbook verify_compose_alignment.yml

# Run Molecule template rendering tests (no inventory needed)
molecule test

# Run a single deployment phase using tags
ansible-playbook -i inventory/hosts --tags deploy_quadlet_files main.yml
```

## Inventory

Inventory needs a host with root SSH access. Example (`inventory/hosts`):
```ini
[dcm]
myhost ansible_host=10.0.0.1 ansible_user=root
```

For hosts behind a jump box, add `ansible_ssh_common_args='-o ProxyJump=root@jumphost'`.

## Deploying Providers

Core services deploy with defaults — no vars file required. To enable optional providers, create a vars file:

**KubeVirt and K8s Container** only need a kubeconfig:
```yaml
dcm_provider_kubevirt: true
dcm_kubevirt_kubeconfig: "/path/to/kubeconfig"

dcm_provider_k8s_container: true
dcm_k8s_container_sp_kubeconfig: "/path/to/kubeconfig"
```

**Three-tier demo** requires `dcm_provider_k8s_container` to also be enabled — it shares the k8s-container kubeconfig:
```yaml
dcm_provider_k8s_container: true
dcm_k8s_container_sp_kubeconfig: "/path/to/kubeconfig"
dcm_provider_three_tier_demo: true
```

**ACM Cluster** requires additional fields (all mandatory, validated before deploy):
```yaml
dcm_provider_acm_cluster: true
dcm_acm_cluster_sp_kubeconfig: "/path/to/kubeconfig"
dcm_acm_cluster_sp_base_domain: "example.com"
dcm_acm_cluster_sp_pull_secret: '{{ lookup("env", "REDHAT_PULL_SECRET") | b64encode }}'
dcm_acm_cluster_sp_default_infra_env: "default"
dcm_acm_cluster_sp_agent_namespace: "assisted-installer"
```

The `pull_secret` must be a valid `.dockerconfigjson` JSON string, base64-encoded. Use `| b64encode` in the same Jinja2 expression as the lookup — if done separately (e.g., in the template), Ansible auto-parses the JSON into a Python dict first, producing single-quoted keys that break the container. Each provider uses a different kubeconfig env var internally — don't assume they're interchangeable in templates.

## Architecture

**Playbook entry points:** `main.yml` runs the full role. `run_role.yml`, `run_role_task.yml`, and `run_task.yml` are helpers for running individual roles or task phases. All four load variables from `vault_path`/`vault_dir` and `vars_path`/`vars_dir` extra-vars before executing.

**The role** (`roles/dcm_deploy/`) deploys 12 containers in six sequential phases defined in `tasks/main.yml`: prerequisites → generate_configs → deploy_quadlet_files → initialize_database → start_services → validate_deployment.

**Config source of truth:** Traefik config and PostgreSQL init SQL are NOT templated — they're copied from a clone of the upstream [api-gateway](https://github.com/dcm-project/api-gateway) repo at deploy time. Only `dcm.env.j2` and the quadlet unit files are Jinja2 templates.

## Critical Naming Convention

There is a deliberate split between container names and systemd unit names:

- **Quadlet filenames** use `dcm-` prefix: `dcm-postgres.container` → generates `dcm-postgres.service`
- **Container names** match compose exactly (NO prefix): `ContainerName=postgres`
- **Systemd dependencies** use `dcm-` unit names: `After=dcm-postgres.service`
- **Environment variables** use compose hostnames: `DB_HOST=postgres`
- **Network quadlet** `dcm-network.network` generates `dcm-network-network.service` (note the double `-network`)

Exception: `dcm-ui` keeps the `dcm-` prefix as its container name (`ContainerName=dcm-ui`) to avoid conflicts with any other `ui` container.

Mixing these up breaks either systemd dependency ordering or container DNS resolution.

## Modifying Templates

All quadlet templates are in `roles/dcm_deploy/templates/`. When editing:

- Every container must have `dcm-network-network.service` in both `After=` and `Requires=`
- Manager images use `Pull=always` (important for `main` tags that update)
- Volume mounts need SELinux suffixes: `:Z` for private (data volumes), `:z` for shared (read-only configs)
- Environment variables shared across managers go in `dcm.env.j2`; per-service vars go as `Environment=` directives in the container template
- Cross-reference env vars against upstream `compose.yaml` — the compose file is the source of truth

## Provider Templates

Optional providers (kubevirt, k8s-container, acm-cluster, three-tier-demo) are gated by `dcm_provider_*` boolean vars. Each provider's required vars are validated before templating — kubevirt/k8s-container only need a kubeconfig, but ACM needs five additional fields (see "Deploying Providers" above). The kubeconfig env var name differs per provider (`KUBERNETES_KUBECONFIG`, `SP_K8S_KUBECONFIG`, `KUBECONFIG`) — don't copy-paste between templates.

The three-tier-demo provider requires `dcm_provider_k8s_container` to be enabled; it shares the `dcm_k8s_container_sp_kubeconfig` path.

## Syncing with Upstream api-gateway

The upstream compose.yaml at `https://github.com/dcm-project/api-gateway` is the source of truth. When it changes, this role must be updated to match. Run `ansible-playbook verify_compose_alignment.yml` first — it checks template file existence, environment variable coverage, dependency ordering (`depends_on` vs `After=`/`Requires=`), port mappings, and image name alignment. If it passes, the templates are in sync with compose.

### New service added to compose.yaml

First classify it: **core manager** (needs DB, always deployed) or **optional provider** (behind a feature flag, needs kubeconfig). If unclear, check the `dcm-project` GitHub org — a service image `quay.io/dcm-project/<name>` typically has its source at `https://github.com/dcm-project/<name>`. Read the repo's README and Dockerfile for env vars, health endpoints, and dependencies.

Files to update (in order):

1. `roles/dcm_deploy/templates/dcm-<service>.container.j2` — create from an existing template of the same type
2. `roles/dcm_deploy/defaults/main.yml` — add image version variable
3. `roles/dcm_deploy/tasks/deploy_quadlet_files.yml` — TWO places: add to the `dcm_expected_quadlets` set_fact (used for stale file detection) AND to the core deploy loop (or as a conditional block with provider validation for optional providers). These lists overlap but serve different purposes — `dcm_expected_quadlets` includes all enabled services, the deploy loop only includes unconditional core services
4. `roles/dcm_deploy/tasks/start_services.yml` — add to startup sequence and running-state verification loop
5. `roles/dcm_deploy/tasks/validate_deployment.yml` — add to container assertions (and health check if it has one)
6. `roles/dcm_deploy/tasks/initialize_database.yml` — add database name if the service needs one. Core manager DBs go in the unconditional loop; optional provider DBs get their own conditional check/create tasks gated by `dcm_provider_*`
7. `roles/dcm_deploy/templates/dcm-gateway.container.j2` — if it's a core manager routed through Traefik, add to `After=`/`Requires=`
8. `verify_compose_alignment.yml` — add to `expected_core_services` or `expected_optional_services`, `service_to_template` mapping, and `infra_image_mapping` if it's an infrastructure image (not a DCM manager)
9. `molecule/default/verify.yml` — add to the appropriate container list (`core_containers` or `provider_containers`) and any relevant assertion groups (`manager_containers`, `env_file_containers`)
10. `README.md` — update stack components table and variable reference

### Existing service changed (env vars, dependencies, ports)

Run `ansible-playbook verify_compose_alignment.yml` — it will catch most drift automatically. Then manually check anything the playbook can't verify:

- Check `DB_NAME=` matches the database name in `initialize_database.yml`
- Update health endpoints in `validate_deployment.yml` if API paths changed
- If you add env vars to `alignment_allowlist` in `verify_compose_alignment.yml`, document why the divergence is intentional

### Image version bumps

Only update `defaults/main.yml` if compose pins a new major/minor (e.g., `postgres:16→17`, `traefik:v3.4→v3.5`). Manager versions default to `main` — don't change unless compose pins a specific tag.

### Config file changes (traefik.yml, routes.yml, init SQL)

These are copied at deploy time from the cloned repo — usually no action needed here. But if new config files are added upstream (e.g., additional Traefik middleware), update `tasks/generate_configs.yml` to copy them.

## CI

GitHub Actions runs five jobs on every PR targeting `main`:

- **check-clean-commits** — shared workflow that checks for merge commits, fixup/squash/WIP markers, vague commit messages
- **Lint** — `yamllint` + `ansible-lint`
- **Syntax check** — `ansible-playbook --syntax-check` on all playbooks
- **Compose-to-quadlet alignment** — runs `verify_compose_alignment.yml` against live upstream compose.yaml
- **Quadlet template rendering** — `molecule test` renders all core quadlet templates locally and asserts naming conventions, dependency symmetry, SELinux suffixes, Pull policy, env file references, and network membership

### Molecule scenario

The `molecule/default/` scenario uses a delegated driver (no containers) and only runs `deploy_quadlet_files` with `ANSIBLE_SKIP_TAGS: systemd` to avoid systemd calls. It tests template rendering correctness, not deployment. Assertions are in `molecule/default/verify.yml`. When adding a new service, update the container lists and assertion groups there.

### Task tags

Each phase in `tasks/main.yml` is tagged (`prerequisites`, `generate_configs`, `deploy_quadlet_files`, `initialize_database`, `start_services`, `validate_deployment`). The two `systemd_service` tasks in `deploy_quadlet_files.yml` are tagged `systemd` — this is how Molecule skips them in CI.
