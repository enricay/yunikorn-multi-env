# YuniKorn Multi-Environment Deployment with Kustomize

This repository demonstrates how to deploy YuniKorn to multiple environments (dev, staging, production) using Kustomize overlays with Flux.

## Directory Structure

```
yunikorn-multi-env/
├── base/                           # Base configuration (shared)
│   ├── helmrelease.yaml           # Base YuniKorn HelmRelease
│   ├── helmrepository.yaml        # YuniKorn Helm Repository
│   ├── namespace.yaml             # Base namespace
│   └── kustomization.yaml         # Base kustomization
└── environments/                   # Environment-specific overlays
    ├── dev/
    │   └── kustomization.yaml     # Dev environment patches
    ├── staging/
    │   └── kustomization.yaml     # Staging environment patches
    └── production/
        └── kustomization.yaml     # Production environment patches
```

## Environment Configurations

### Development Environment
- **Resources**: 200m CPU, 256Mi memory (minimal)
- **Replicas**: 1 for all components
- **Namespace**: `yunikorn-dev`

### Staging Environment
- **Resources**: 1000m CPU, 1Gi memory (moderate)
- **Replicas**: 2 for all components
- **Namespace**: `yunikorn-staging`
- **Node Selection**: Nodes labeled with `environment: staging`

### Production Environment
- **Resources**: 2000m CPU, 4Gi memory (high)
- **Replicas**: 3 admission controllers & schedulers, 5 web instances
- **Namespace**: `yunikorn-prod`
- **Advanced Features**:
  - Node affinity for production nodes
  - Pod anti-affinity for high availability
  - Tolerations for production taints

## Testing the Configuration

Test what each environment generates:

```bash
# Development
kustomize build environments/dev

# Staging
kustomize build environments/staging

# Production
kustomize build environments/production
```

## Deploying with Flux

### Option 1: Separate GitRepository per Environment

Create separate Flux Kustomization resources:

```yaml
# flux-dev-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: yunikorn-dev
  namespace: flux-system
spec:
  interval: 5m
  path: ./environments/dev
  prune: true
  sourceRef:
    kind: GitRepository
    name: yunikorn-config
```

### Option 2: Multi-Environment in Single Repository

Add to your main Flux repository:

```bash
# Copy to your Flux repository
cp -r yunikorn-multi-env/ /path/to/flux-repo/apps/

# Flux will automatically discover and apply all environments
```

### Option 3: Selective Environment Deployment

Deploy only specific environments by adding them to your Flux Kustomization:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: yunikorn-production-only
  namespace: flux-system
spec:
  interval: 10m
  path: ./yunikorn-multi-env/environments/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

## Key Benefits of This Approach

1. **DRY Principle**: Base configuration shared across environments
2. **Environment Isolation**: Each environment has its own namespace and configuration
3. **Scalable**: Easy to add new environments
4. **GitOps Ready**: Works seamlessly with Flux
5. **Testable**: Can validate configurations locally with `kustomize build`

## Customization Examples

### Adding a New Environment (QA)

```bash
mkdir environments/qa
cat << EOF > environments/qa/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: yunikorn-qa

resources:
  - ../../base

commonLabels:
  environment: qa
  app.kubernetes.io/instance: yunikorn-qa

nameSuffix: -qa

patches:
  - target:
      kind: HelmRelease
      name: yunikorn
    patch: |-
      - op: replace
        path: /metadata/namespace
        value: yunikorn-qa
      - op: replace
        path: /spec/values/web/replicaCount
        value: 2
EOF
```

### Environment-Specific Secrets

Add secrets per environment:

```bash
# In environments/production/
kubectl create secret generic yunikorn-config \
  --from-file=config.yaml \
  --dry-run=client -o yaml > yunikorn-secret.yaml
```

Then reference in the HelmRelease values through patches.
