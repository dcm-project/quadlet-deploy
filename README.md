# quadlet-deploy

Deploy [DCM (Data Center Management)](https://github.com/dcm-project) as a systemd-managed Podman quadlet service stack on RHEL 9 or Fedora.

DCM is a microservice platform for managing service providers, catalogs, policies, and placement across infrastructure targets. This repo provides the canonical Ansible role for deploying all DCM services as individual quadlet containers with systemd integration — automatic restarts, dependency ordering, journal logging, and lifecycle management. It is the recommended production deployment path for running DCM on a RHEL 9 host without Kubernetes.

## Stack Components

| Service | Image | Description |
|---------|-------|-------------|
| postgres | `docker.io/library/postgres:16-alpine` | Shared PostgreSQL database (5 databases) |
| nats | `docker.io/library/nats:2-alpine` | NATS message broker with JetStream |
| service-provider-manager | `quay.io/dcm-project/service-provider-manager` | Manages service provider registrations |
| catalog-manager | `quay.io/dcm-project/catalog-manager` | Service type catalog and instances |
| policy-manager | `quay.io/dcm-project/policy-manager` | Policy evaluation engine |
| placement-manager | `quay.io/dcm-project/placement-manager` | Resource placement decisions |
| gateway | `docker.io/traefik:v3.4` | Traefik reverse proxy (API gateway) |
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
  │     ├── dcm-service-provider-manager.service  (+ dcm-nats.service)
  │     ├── dcm-catalog-manager.service
  │     ├── dcm-policy-manager.service
  │     └── dcm-placement-manager.service
  ├── dcm-nats.service
  └── dcm-gateway.service  (after all 4 managers)
        ├── dcm-ui.service
        ├── dcm-kubevirt-service-provider.service  (optional)
        ├── dcm-k8s-container-service-provider.service  (optional)
        │     └── dcm-three-tier-demo-service-provider.service  (optional)
        └── dcm-acm-cluster-service-provider.service  (optional)
```

Container names match the upstream `compose.yaml` exactly (no prefix). Systemd unit names use a `dcm-` prefix. Services resolve each other by container name via Podman DNS.

Configuration files (Traefik routes, PostgreSQL init SQL) are sourced from the [api-gateway](https://github.com/dcm-project/api-gateway) repository, cloned at deploy time. This keeps api-gateway as the single source of truth.

## Prerequisites

- RHEL 9 or Fedora target host with Podman 4.4+ (quadlet support)
- Ansible 2.15+ on the control node
- `ansible.posix` collection: `ansible-galaxy collection install ansible.posix`
- Network access to pull container images from `docker.io` and `quay.io`
- Dedicated host (container names like `postgres` and `nats` assume no collisions)

## Deployment Phases

The `dcm_deploy` role executes in six phases:

1. **Prerequisites** — installs container tools, firewalld, and git; opens the gateway and UI ports; creates config directories
2. **Generate configs** — clones the api-gateway repo, copies Traefik config and PostgreSQL init SQL, templates the shared environment file, then cleans up the clone
3. **Deploy quadlet files** — places `.container`, `.network`, and `.volume` unit files into `/etc/containers/systemd/` and reloads systemd
4. **Initialize database** — starts PostgreSQL, creates all service databases if they don't exist
5. **Start services** — phased startup: NATS, then all four managers (with health checks), then the gateway, then the UI, then optional providers
6. **Validate** — checks the Traefik `/ping` endpoint, verifies all manager health endpoints through the gateway, and asserts all expected containers are running

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
| `dcm_traefik_image` | `docker.io/traefik` | Traefik image |
| `dcm_traefik_image_tag` | `v3.4` | Traefik tag |
| `dcm_image_registry` | `quay.io/dcm-project` | Registry for DCM service images |

### Manager Versions

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_service_provider_manager_version` | `main` | Service provider manager image tag |
| `dcm_catalog_manager_version` | `main` | Catalog manager image tag |
| `dcm_policy_manager_version` | `main` | Policy manager image tag |
| `dcm_placement_manager_version` | `main` | Placement manager image tag |
| `dcm_ui_version` | `main` | DCM UI image tag |

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_db_user` | `admin` | PostgreSQL username |
| `dcm_db_password` | `adminpass` | PostgreSQL password (override for production) |

### Networking

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_gateway_port` | `9080` | Traefik gateway port published to host |
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

### API Gateway Source

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_api_gateway_repo` | `https://github.com/dcm-project/api-gateway.git` | Repo cloned for config files |
| `dcm_api_gateway_version` | `main` | Branch/tag to clone |

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

## Verification

After deployment, verify the stack is healthy:

```bash
# Traefik gateway responds
curl http://<host>:9080/ping

# Manager health endpoints (through gateway)
curl http://<host>:9080/api/v1alpha1/health/providers
curl http://<host>:9080/api/v1alpha1/health/catalog
curl http://<host>:9080/api/v1alpha1/health/policies
curl http://<host>:9080/api/v1alpha1/health/placement

# DCM UI accessible
curl http://<host>:7007

# All systemd units active
systemctl status dcm-*.service

# All containers running
podman ps

# Databases created
podman exec postgres psql -U admin -l
```

## Compose Alignment

A standalone playbook `verify_compose_alignment.yml` checks that every service in the upstream `compose.yaml` has a corresponding quadlet template. Run it in CI to detect drift:

```bash
ansible-playbook verify_compose_alignment.yml
```

## Related Repositories

- [dcm-project/api-gateway](https://github.com/dcm-project/api-gateway) — upstream compose file, Traefik config, and init SQL
- [dcm-project/utilities](https://github.com/dcm-project/utilities) — podman-compose dev deployment and E2E tooling
- [brianredbeard/rhis-builder-quay](https://github.com/brianredbeard/rhis-builder-quay) — structural inspiration for the Ansible quadlet deployment pattern
