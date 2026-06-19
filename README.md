# quadlet-deploy

Deploy [DCM (Data Center Management)](https://github.com/dcm-project) as a systemd-managed Podman quadlet service stack on RHEL 9 or Fedora.

DCM is a platform for managing service providers, catalogs, policies, and placement across infrastructure targets. This repo provides the canonical Ansible role for deploying all DCM services as individual quadlet containers with systemd integration — automatic restarts, dependency ordering, journal logging, and lifecycle management. It is the recommended production deployment path for running DCM on a RHEL 9 host without Kubernetes.

## Stack Components

| Service | Image | Description |
|---------|-------|-------------|
| postgres | `docker.io/library/postgres:16-alpine` | Shared PostgreSQL database |
| nats | `docker.io/library/nats:2-alpine` | NATS message broker with JetStream |
| control-plane | `quay.io/dcm-project/control-plane` | DCM monolith (catalog, policy, placement, SP management) |
| dcm-ui | `quay.io/dcm-project/dcm-ui` | DCM web interface |

### Optional Service Providers

| Service | Image | Description |
|---------|-------|-------------|
| kubevirt-service-provider | `quay.io/dcm-project/kubevirt-service-provider` | KubeVirt VM management |
| k8s-container-service-provider | `quay.io/dcm-project/k8s-container-service-provider` | Kubernetes container workloads |
| acm-cluster-service-provider | `quay.io/dcm-project/acm-cluster-service-provider` | ACM cluster lifecycle |
| three-tier-app-demo-service-provider | `quay.io/dcm-project/three-tier-app-demo-service-provider` | Three-tier demo application (requires k8s-container) |

## Architecture

All services run as standalone containers on a shared bridge network (`dcm-network`). Each container is an independent systemd unit with explicit dependency ordering:

```
dcm-network-network.service
  ├── dcm-postgres.service
  ├── dcm-nats.service
  │     └── dcm-control-plane.service
  │           ├── dcm-ui.service
  │           ├── dcm-kubevirt-service-provider.service  (optional)
  │           ├── dcm-k8s-container-service-provider.service  (optional)
  │           │     └── dcm-three-tier-demo-service-provider.service  (optional)
  │           └── dcm-acm-cluster-service-provider.service  (optional)
```

Container names match the upstream `deploy/compose.yaml` exactly (no prefix). Systemd unit names use a `dcm-` prefix. Services resolve each other by container name via Podman DNS.

PostgreSQL init SQL is sourced from the [control-plane](https://github.com/dcm-project/control-plane) repository, cloned at deploy time. This keeps control-plane as the single source of truth.

## Prerequisites

- RHEL 9 or Fedora target host with Podman 4.4+ (quadlet support)
- Ansible 2.16+ on the control node (rootless mode requires `systemd_service` `scope` parameter)
- `ansible.posix` collection: `ansible-galaxy collection install ansible.posix`
- Network access to pull container images from `docker.io` and `quay.io`
- Dedicated host (container names like `postgres` and `nats` assume no collisions)

## Deployment Phases

The `dcm_deploy` role executes in six phases (seven when `dcm_rootless: true`):

1. **Resolve rootless vars** *(optional, rootless only)* — creates the service user, configures subuid/subgid, enables lingering, starts the user systemd instance, and sets internal facts for user-scoped paths and systemd
1. **Prerequisites** — installs container tools, firewalld, and git; opens the control-plane and UI ports; creates config directories
2. **Generate configs** — clones the control-plane repo, copies PostgreSQL init SQL, templates the shared environment file, then cleans up the clone
3. **Deploy quadlet files** — places `.container`, `.network`, and `.volume` unit files into `/etc/containers/systemd/` and reloads systemd
4. **Initialize database** — starts PostgreSQL, creates all service databases if they don't exist
5. **Start services** — phased startup: NATS, then control-plane (with health check), then the UI, then optional providers
6. **Validate** — checks the control-plane health endpoint and asserts all expected containers are running

## Usage

### Basic deployment

```bash
ansible-playbook -i inventory/hosts main.yml
```

### With custom vars

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "vars_path=vars/dcm.yml" \
  main.yml
```

### With vault secrets

```bash
ansible-playbook -i inventory/hosts \
  --ask-vault-password \
  --extra-vars "vault_path=vault/dcm-vault.yml" \
  --extra-vars "vars_path=vars/dcm.yml" \
  main.yml
```

### Run a specific phase

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "role_name=dcm_deploy" \
  --extra-vars "task_name=validate_deployment" \
  run_role_task.yml
```

### Enable optional providers

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "vars_path=vars/dcm.yml" \
  --extra-vars '{"dcm_provider_kubevirt": true, "dcm_kubevirt_kubeconfig": "/path/to/kubeconfig"}' \
  main.yml
```

## Variable Reference

### Container Images

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_postgres_image` | `docker.io/library/postgres` | PostgreSQL image |
| `dcm_postgres_image_tag` | `16-alpine` | PostgreSQL tag |
| `dcm_nats_image` | `docker.io/library/nats` | NATS image |
| `dcm_nats_image_tag` | `2-alpine` | NATS tag |
| `dcm_image_registry` | `quay.io/dcm-project` | Registry for DCM service images |

### Control Plane

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_control_plane_version` | `main` | Image tag and git clone ref |
| `dcm_control_plane_repo` | `https://github.com/dcm-project/control-plane.git` | Repo cloned for postgres init SQL |
| `dcm_control_plane_clone_dir` | `/tmp/dcm-control-plane` | Clone destination on target host |

### DCM UI

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_ui_version` | `main` | DCM UI image tag |

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_db_user` | `admin` | PostgreSQL username |
| `dcm_db_password` | `adminpass` | PostgreSQL password (override for production) |

### Networking

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_control_plane_port` | `8080` | Control-plane API port published to host |
| `dcm_ui_port` | `7007` | DCM UI port published to host (also used in `APP_BASE_URL`) |
| `dcm_postgres_port` | `5432` | PostgreSQL port published to host |
| `dcm_nats_port` | `4222` | NATS client port published to host |
| `dcm_nats_monitor_port` | `8222` | NATS monitoring port published to host |

The DCM UI's `APP_BASE_URL` defaults to `http://<ansible_host>:<dcm_ui_port>`. If the UI is accessed via a different hostname, domain, or behind a reverse proxy, override `ansible_host` in your inventory or set `APP_BASE_URL` directly in a custom vars file.

### Paths

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_config_dir` | `/srv/containers/dcm/config` | Configuration files on target host |
| `dcm_quadlet_dir` | `/etc/containers/systemd` | Quadlet unit file directory |

### Feature Toggles

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_provider_kubevirt` | `false` | Enable KubeVirt service provider |
| `dcm_provider_k8s_container` | `false` | Enable K8s container service provider |
| `dcm_provider_acm_cluster` | `false` | Enable ACM cluster service provider |
| `dcm_provider_three_tier_demo` | `false` | Enable three-tier demo provider (requires k8s-container) |

### KubeVirt Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_kubevirt_provider_name` | `kubevirt-service-provider` | Provider registration name |
| `dcm_kubevirt_namespace` | `default` | Kubernetes namespace |
| `dcm_kubevirt_kubeconfig` | `""` | Path to kubeconfig on target host |

### K8s Container Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_k8s_container_sp_name` | `k8s-container-provider` | Provider registration name |
| `dcm_k8s_container_sp_namespace` | `default` | Kubernetes namespace |
| `dcm_k8s_container_sp_external_svc_type` | `NodePort` | Kubernetes service type for external access |
| `dcm_k8s_container_sp_kubeconfig` | `""` | Path to kubeconfig on target host |

### ACM Cluster Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_acm_cluster_sp_name` | `acm-cluster-sp` | Provider registration name |
| `dcm_acm_cluster_sp_namespace` | `default` | ACM namespace |
| `dcm_acm_cluster_sp_base_domain` | `""` | Base domain for cluster provisioning |
| `dcm_acm_cluster_sp_pull_secret` | `""` | Pull secret for cluster provisioning |
| `dcm_acm_cluster_sp_default_infra_env` | `""` | Default InfraEnv (BareMetal platform) |
| `dcm_acm_cluster_sp_agent_namespace` | `""` | Agent namespace |
| `dcm_acm_cluster_sp_kubeconfig` | `""` | Path to kubeconfig on target host |

### Three-Tier Demo Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_three_tier_demo_sp_name` | `three-tier-provider` | Provider registration name |
| `dcm_three_tier_demo_sp_namespace` | `default` | Kubernetes namespace |
| `dcm_three_tier_demo_service_provider_version` | `main` | Image tag |

Note: three-tier-demo shares `dcm_k8s_container_sp_kubeconfig` — set `dcm_provider_k8s_container: true` alongside this provider.

### Firewall

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_firewall_zone` | `public` | Firewalld zone for published ports |

### Rootless Deployment

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_rootless` | `false` | Enable rootless Podman deployment |
| `dcm_rootless_user` | `dcm` | User account to run containers as |
| `dcm_rootless_create_user` | `true` | Whether the role should create the user |
| `dcm_rootless_home` | `/home/{{ dcm_rootless_user }}` | Home directory path for the rootless user |

## Rootless Deployment

Set `dcm_rootless: true` to run all containers under a dedicated unprivileged user instead of root. This uses user-scoped systemd and rootless Podman — no root privileges are needed at runtime.

```yaml
dcm_rootless: true
dcm_rootless_user: dcm          # default
```

Set `dcm_rootless_create_user: false` if user management is handled externally (LDAP, IPA, etc.). The `ansible.builtin.user` module is idempotent, so you do not need to disable it just because the user already exists locally.

### Path differences

| Resource | Rootful (default) | Rootless |
|----------|-------------------|----------|
| Quadlet directory | `/etc/containers/systemd` | `~dcm/.config/containers/systemd` |
| Config directory | `/srv/containers/dcm/config` | `~dcm/.local/share/dcm/config` |
| Systemd target | `multi-user.target` | `default.target` |
| Systemd scope | `system` | `user` |

### Requirements

- `ansible-core >= 2.16` on the control node (required for the `scope` parameter on the `systemd_service` module)
- `become_method: sudo` configured for the target host (the role uses nested privilege dropping to the rootless user)

### Limitations

- Rootless and rootful deployments are **mutually exclusive** on the same host. Do not enable both.
- There is **no migration path** from rootful to rootless. Data volumes live in different storage paths per Podman's storage model.

## Verification

After deployment, verify the stack is healthy:

```bash
# Control-plane health endpoint
curl http://<host>:8080/api/v1alpha1/health

# DCM UI accessible
curl http://<host>:7007

# All systemd units active (rootful)
systemctl status dcm-*.service
# For rootless, run as the service user:
#   sudo -u dcm XDG_RUNTIME_DIR=/run/user/$(id -u dcm) systemctl --user status dcm-*.service

# All containers running (rootful)
podman ps
# For rootless:
#   sudo -u dcm podman ps

# Databases created (rootful)
podman exec postgres psql -U admin -l
# For rootless:
#   sudo -u dcm podman exec postgres psql -U admin -l
```

## Compose Alignment

A standalone playbook `verify_compose_alignment.yml` checks that every service in the upstream `deploy/compose.yaml` has a corresponding quadlet template. Run it in CI to detect drift:

```bash
ansible-playbook verify_compose_alignment.yml
```

## Related Repositories

- [dcm-project/control-plane](https://github.com/dcm-project/control-plane) — upstream compose file and postgres init SQL
- [dcm-project/utilities](https://github.com/dcm-project/utilities) — podman-compose dev deployment and E2E tooling
- [brianredbeard/rhis-builder-quay](https://github.com/brianredbeard/rhis-builder-quay) — structural inspiration for the Ansible quadlet deployment pattern
