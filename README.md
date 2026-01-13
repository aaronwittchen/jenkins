# Jenkins Setup for Kubernetes

Kubernetes manifests for deploying Jenkins CI/CD server with Docker-in-Docker support.

## Features

- Jenkins LTS with persistent storage
- Docker-in-Docker sidecar for building container images
- RBAC configured for pod creation (build agents)
- Istio Gateway integration via HTTPRoute
- Kustomize overlays for different storage classes

## Architecture

```
┌─────────────────────────────────────────┐
│              Jenkins Pod                │
│  ┌─────────────┐  ┌─────────────────┐   │
│  │   Jenkins   │  │  Docker (DinD)  │   │
│  │  Container  │──│    Sidecar      │   │
│  │  :8080      │  │    :2376        │   │
│  └─────────────┘  └─────────────────┘   │
│         │                  │            │
│         └──────┬───────────┘            │
│                │                        │
│    ┌───────────┴───────────┐            │
│    │   Shared Volumes      │            │
│    │  - docker-certs       │            │
│    │  - docker-cli         │            │
│    │  - jenkins-home (PVC) │            │
│    └───────────────────────┘            │
└─────────────────────────────────────────┘
```

## Directory Structure

```
jenkins-setup/
├── base/
│   ├── namespace.yaml        # jenkins namespace
│   ├── service-accounts.yaml # RBAC for Jenkins
│   ├── pvcs.yaml             # Persistent storage (10Gi)
│   ├── deployment.yaml       # Jenkins + DinD deployment
│   ├── httproute.yaml        # Istio gateway route
│   └── kustomization.yaml
└── overlays/
    ├── longhorn/             # Longhorn storage class
    │   └── kustomization.yaml
    └── local-path/           # Local-path storage class
        └── kustomization.yaml
```

## Deployment

### Via ArgoCD (Recommended)

Apply the ArgoCD application:

```bash
kubectl apply -f ArgoCD/applications/jenkins.yaml
```

### Manual Deployment

```bash
# Using Longhorn storage
kubectl apply -k overlays/longhorn/

# Using local-path storage
kubectl apply -k overlays/local-path/
```

## Access

- **URL**: `https://jenkins.k8s.local`
- **Initial Admin Password**:
  ```bash
  kubectl exec -n jenkins deploy/jenkins -c jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
  ```

## Configuration

### Required Plugins

Install these plugins after initial setup:

- Docker Pipeline
- Git
- Pipeline

### Docker Hub Credentials

1. Go to **Manage Jenkins** → **Credentials**
2. Add credential with ID: `dockerhub-creds`
3. Type: Username with password
4. Use Docker Hub access token as password

## Resource Requirements

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Jenkins   | 250m        | 1000m     | 512Mi          | 2Gi          |
| DinD      | 250m        | 1000m     | 512Mi          | 2Gi          |

## Storage

- **jenkins-home-pvc**: 10Gi persistent volume for Jenkins data

## Security Notes

- DinD runs in privileged mode (required for Docker)
- Jenkins service account has pod creation permissions
- TLS enabled between Jenkins and DinD daemon

## Troubleshooting

### Docker not found

Ensure the init container completed successfully:

```bash
kubectl logs -n jenkins deploy/jenkins -c init-docker-cli
```

### Build stuck waiting for executor

Check executor count on built-in node:

1. **Manage Jenkins** → **Nodes** → **Built-In Node** → **Configure**
2. Set **Number of executors** to 1 or higher
