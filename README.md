# infra-gitops

GitOps-based infrastructure management for Kubernetes cluster using ArgoCD, External Secrets Operator, and Doppler.

## Overview

This repository contains Kubernetes manifests and configurations managed via GitOps principles. All infrastructure and applications are declared as code and automatically synced to the cluster using ArgoCD.

## Architecture

```
┌─────────────────────┐
│   Git Repository    │
│  (infra-gitops)     │
│                     │
│  ✅ Deployments     │
│  ✅ ConfigMaps      │
│  ✅ ExternalSecrets │  ← Only references secrets
│  ✅ SecretStores    │  ← Connection configs
│  ❌ Actual secrets  │  ← Never stored here
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│      ArgoCD         │
│  (GitOps Engine)    │
│  ❌ Never sees      │
│     secret values   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐         ┌──────────────────┐
│   Kubernetes        │◄────────│    Doppler       │
│                     │  Fetches│  (Cloud SaaS)    │
│  External Secrets   │  values │                  │
│  Operator (ESO)     │────────►│  ✅ Secrets      │
│                     │         │  ✅ Encrypted    │
│  Creates K8s        │         │  ✅ Audit logs   │
│  Secrets from       │         │  ✅ Access ctrl  │
│  Doppler            │         └──────────────────┘
└─────────────────────┘
```

## Repository Structure

```
.
├── argocd/                    # ArgoCD configuration
│   ├── base/                  # Base ArgoCD install
│   │   ├── install.yaml       # ArgoCD manifests
│   │   └── kustomization.yaml
│   └── overlays/
│       └── prod/              # Production overlay
│           ├── infra-project.yaml              # AppProject definition
│           ├── argocd-application.yaml         # ArgoCD self-management
│           ├── external-secrets-app.yaml       # ESO apps
│           ├── cloudflared-app.yaml            # Cloudflared app
│           ├── argocd-server-lb-patch.yaml     # LoadBalancer patch
│           ├── repo-server-sops-patch.yaml     # SOPS config (legacy)
│           └── kustomization.yaml
│
├── external-secrets-operator/ # External Secrets Operator
│   └── base/
│       ├── namespace.yaml     # ESO namespace
│       └── kustomization.yaml # CRDs
│
└── cloudflared/               # Cloudflare Tunnel
    ├── base/
    │   ├── configmap.yaml     # Tunnel configuration
    │   ├── deployment.yaml    # Cloudflared deployment
    │   ├── secret-store.yaml  # Doppler connection config
    │   ├── external-secret.yaml # Secret fetch definition
    │   └── kustomization.yaml
    └── overlays/
        └── prod/
            └── kustomization.yaml
```

## Applications

### Core Infrastructure

| Application | Namespace | Description |
|------------|-----------|-------------|
| **argocd** | `argocd` | GitOps continuous delivery tool |
| **external-secrets-crds** | `external-secrets` | ESO Custom Resource Definitions |
| **external-secrets-operator** | `external-secrets` | ESO controller for secret management |

### Services

| Application | Namespace | Description |
|------------|-----------|-------------|
| **cloudflared** | `cloudflared` | Cloudflare Tunnel for secure ingress |

## Secret Management

This repository uses **External Secrets Operator (ESO)** with **Doppler** as the secret backend.

### Why ESO + Doppler?

**Security:**
- ✅ Secrets never stored in Git (not even encrypted)
- ✅ ArgoCD never has access to secret values
- ✅ Separation of deployment access and secret access
- ✅ Centralized audit trail in Doppler

**Developer Experience:**
- ✅ Easy secret rotation via Doppler UI
- ✅ Automatic sync to cluster (15min refresh)
- ✅ Multiple environments (dev/stg/prd)
- ✅ Fine-grained access control

### How It Works

1. **SecretStore** defines connection to Doppler (project, environment, auth)
2. **ExternalSecret** declares what secret to fetch and where to put it
3. **ESO Controller** watches ExternalSecrets, fetches from Doppler, creates K8s Secrets
4. **Application** uses the K8s Secret (standard volume mount or env var)

### Example

```yaml
# SecretStore - connection config
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: doppler-secret-store
  namespace: cloudflared
spec:
  provider:
    doppler:
      project: baremetal-k8s-project
      config: prd
      auth:
        secretRef:
          dopplerToken:
            name: doppler-token-auth
            key: dopplerToken

---
# ExternalSecret - what to fetch
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cloudflared-credentials
  namespace: cloudflared
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: doppler-secret-store
    kind: SecretStore
  target:
    name: cloudflared-credentials
  data:
    - secretKey: tunnel-token
      remoteRef:
        key: CLOUDFLARED_TUNNEL_TOKEN
```

## Prerequisites

- Kubernetes cluster (1.24+)
- kubectl configured
- ArgoCD CLI (optional, for easier management)
- Doppler account (free tier available)

## Initial Setup

### 1. Bootstrap ArgoCD

```bash
# Apply ArgoCD base installation
kubectl apply -k argocd/overlays/prod/

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. Setup Doppler

1. Sign up at https://doppler.com
2. Create a project (e.g., `baremetal-k8s-project`)
3. Add secrets to the `prd` environment
4. Generate a service token (Project Settings → Service Tokens)
   - Name: `kubernetes-prod`
   - Environment: `prd`
   - Access: `Read`

### 3. Create Bootstrap Secret

```bash
# Create the Doppler authentication token secret
kubectl create secret generic doppler-token-auth \
  --from-literal=dopplerToken="dp.st.prd.YOUR_TOKEN_HERE" \
  -n cloudflared
```

### 4. Sync Applications

ArgoCD will automatically detect and sync applications from Git. You can also manually trigger:

```bash
# Sync in order
argocd app sync argocd
argocd app sync external-secrets-crds
argocd app sync external-secrets-operator
argocd app sync cloudflared
```

## Adding New Applications

### Using GitOps (Recommended)

1. Create application manifests in a new directory (e.g., `my-app/base/`)
2. Create ArgoCD Application definition in `argocd/overlays/prod/my-app.yaml`
3. Add to `argocd/overlays/prod/kustomization.yaml`:
   ```yaml
   resources:
     - my-app.yaml
   ```
4. Commit and push - ArgoCD will automatically deploy

### Application Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://github.com/nishanau/infra-gitops.git
    targetRevision: main
    path: my-app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Adding Secrets

1. Add secret to Doppler (in your project, `prd` environment)
2. Create ExternalSecret in your app directory:
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: my-app-secrets
     namespace: my-app
   spec:
     refreshInterval: 15m
     secretStoreRef:
       name: doppler-secret-store
       kind: SecretStore
     target:
       name: my-app-secrets
     data:
       - secretKey: my-key
         remoteRef:
           key: MY_DOPPLER_SECRET_NAME
   ```
3. Reference in your deployment:
   ```yaml
   env:
     - name: MY_VAR
       valueFrom:
         secretKeyRef:
           name: my-app-secrets
           key: my-key
   ```

## Troubleshooting

### Check ArgoCD Application Status

```bash
# List all applications
argocd app list

# Get application details
argocd app get <app-name>

# View application logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

### Check ESO Status

```bash
# Check ESO pods
kubectl get pods -n external-secrets

# Check SecretStore status
kubectl get secretstore -A

# Check ExternalSecret status
kubectl get externalsecret -A

# Check ESO logs
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
```

### Debug Secret Issues

```bash
# Check if secret was created
kubectl get secret <secret-name> -n <namespace>

# Describe ExternalSecret for errors
kubectl describe externalsecret <name> -n <namespace>

# Check ESO controller logs
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=100
```

### Common Issues

**ExternalSecret shows "SecretSyncedError":**
- Check Doppler token is valid: `kubectl get secret doppler-token-auth -n <namespace>`
- Verify secret exists in Doppler with exact key name
- Check SecretStore is Valid: `kubectl get secretstore -n <namespace>`

**ArgoCD shows "ComparisonError":**
- Hard refresh the application in UI
- Clear cache: `argocd app get <app-name> --hard-refresh`

**CRDs not found:**
- Ensure `external-secrets-crds` app is synced first
- Check CRDs exist: `kubectl get crd | grep external-secrets`

## Migration Notes

### From SOPS to ESO + Doppler

This repository was migrated from SOPS (manifest generation-based) to ESO + Doppler (destination cluster-based) for improved security and maintainability.

**Benefits:**
- No encrypted secrets in Git
- No decryption keys in ArgoCD
- Centralized secret management
- Better audit trail and rotation

See `technical_guide_and_learning_logs.md` for detailed migration steps.

## Contributing

1. Create a feature branch
2. Make changes
3. Test in a non-production environment
4. Create a pull request
5. After approval, merge triggers automatic deployment

## Security

- ⚠️ Never commit actual secrets to Git
- ⚠️ Never commit Doppler tokens to Git
- ✅ Use ExternalSecrets for all sensitive data
- ✅ Rotate Doppler tokens regularly
- ✅ Use separate Doppler projects for different environments
- ✅ Enable MFA on Doppler account

## License

Private repository - all rights reserved.

## Support

For issues or questions, contact the infrastructure team or create an issue in this repository.
