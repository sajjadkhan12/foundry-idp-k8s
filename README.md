# Foundry IDP Kubernetes Manifests

This directory contains Kubernetes manifests for deploying the Foundry IDP application.

## Prerequisites

- Kubernetes cluster (v1.24+)
- kubectl configured to access your cluster
- Ingress controller (nginx-ingress recommended)
- cert-manager (for TLS certificates)
- Storage class for persistent volumes
- ArgoCD installed and configured (for GitOps deployment)

## Setup Instructions

### 1. Configure GitHub Actions Secret

For automatic manifest updates, configure a GitHub secret in the main `foundry-idp` repository:

**Option 1: Personal Access Token (PAT)**
1. Create a Personal Access Token (PAT) with `repo` scope
2. Go to your `foundry-idp` repository → Settings → Secrets and variables → Actions
3. Add a new secret named `K8S_REPO_TOKEN` with your PAT value

**Option 2: GitHub Token (if repos are in same organization)**
- If both `foundry-idp` and `foundry-idp-k8s` are in the same GitHub organization, you can use `GITHUB_TOKEN` (automatically provided)
- The workflow will fall back to `GITHUB_TOKEN` if `K8S_REPO_TOKEN` is not set

**Required permissions:**
- `contents: write` - To commit and push changes to the k8s repository

### 2. Update Image References

The deployment manifests use GitHub Container Registry (GHCR) images. The image references will be automatically updated by GitHub Actions when new images are built.

**Initial setup:** Replace `owner` in the image paths with your GitHub username or organization:
- `deployment-backend.yaml`: `ghcr.io/owner/foundry-idp/backend:latest`
- `deployment-worker.yaml`: `ghcr.io/owner/foundry-idp/backend:latest` (uses same backend image)
- `deployment-frontend.yaml`: `ghcr.io/owner/foundry-idp/frontend:latest`

**Note:** After the first successful build, GitHub Actions will automatically update these with the correct image tags.

### 3. Create Secrets

Create the backend secrets from your `.env` file:

```bash
# Option 1: From .env file
kubectl create secret generic backend-secrets \
  --from-env-file=../foundry-idp/backend/.env \
  --namespace=foundry-idp \
  --dry-run=client -o yaml > secret-backend.yaml

# Option 2: Manual creation
kubectl create secret generic backend-secrets \
  --from-literal=DATABASE_URL="postgresql://user:pass@host:5432/db" \
  --from-literal=SECRET_KEY="your-secret-key" \
  --namespace=foundry-idp
```

**Important:** Never commit `secret-backend.yaml` to version control. Use `secret-backend.yaml.example` as a template.

### 4. Update Configuration

1. Update `configmap-backend.yaml` with your specific configuration
2. Update `ingress.yaml` with your domain name and TLS configuration
3. Update storage class in `pvc-backend-storage.yaml` based on your cluster

### 5. Deploy

**Option A: Manual Deployment**

```bash
# Apply all manifests
kubectl apply -k .

# Or apply individually
kubectl apply -f namespace.yaml
kubectl apply -f configmap-backend.yaml
kubectl apply -f secret-backend.yaml
kubectl apply -f deployment-backend.yaml
kubectl apply -f deployment-worker.yaml
kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-redis.yaml
kubectl apply -f service-backend.yaml
kubectl apply -f service-frontend.yaml
kubectl apply -f service-redis.yaml
kubectl apply -f pvc-backend-storage.yaml
kubectl apply -f ingress.yaml
```

**Option B: GitOps with ArgoCD (Recommended)**

1. **Configure ArgoCD Application:**
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: foundry-idp
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/your-org/foundry-idp-k8s
       targetRevision: main
       path: .
     destination:
       server: https://kubernetes.default.svc
       namespace: foundry-idp
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

2. **Apply ArgoCD Application:**
   ```bash
   kubectl apply -f argocd-application.yaml
   ```

3. **ArgoCD will automatically:**
   - Watch the repository for changes
   - Sync changes when manifests are updated
   - Deploy new images when GitHub Actions updates the manifests

### 6. Verify Deployment

```bash
# Check pods
kubectl get pods -n foundry-idp

# Check services
kubectl get svc -n foundry-idp

# Check ingress
kubectl get ingress -n foundry-idp

# View logs
kubectl logs -f deployment/backend -n foundry-idp
kubectl logs -f deployment/backend-worker -n foundry-idp
kubectl logs -f deployment/frontend -n foundry-idp
```

## DNS Configuration

Configure your DNS to point to your ingress controller's external IP:

- `foundry-idp.com` → Ingress IP
- `www.foundry-idp.com` → Ingress IP
- `api.foundry-idp.com` → Ingress IP

## TLS Certificates

The ingress is configured to use cert-manager for automatic TLS certificate management. Ensure you have:

1. cert-manager installed in your cluster
2. A ClusterIssuer named `letsencrypt-prod` configured

Example ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

## Database

This setup assumes you have a PostgreSQL database available. You can either:

1. Use an external managed database service
2. Deploy PostgreSQL in your cluster using Helm or operators

Update the `DATABASE_URL` in `secret-backend.yaml` accordingly.

## Storage

The backend requires persistent storage for plugins and git repositories. Update the storage class in `pvc-backend-storage.yaml` based on your cluster's available storage classes.

## Scaling

Adjust replica counts in the deployment files based on your needs:

- Backend API: Currently 3 replicas
- Backend Worker: Currently 2 replicas
- Frontend: Currently 2 replicas

## Monitoring

Consider adding:
- Prometheus metrics endpoints
- Grafana dashboards
- Application logging (ELK stack, Loki, etc.)

## Automated Deployment Workflow

When you push code to the `foundry-idp` repository:

1. **GitHub Actions builds Docker images** and pushes them to GHCR
2. **GitHub Actions updates manifests** in this repository with new image tags
3. **ArgoCD detects the changes** in the k8s repository
4. **ArgoCD automatically deploys** the new images to your cluster

**Image Tagging Strategy:**
- Semantic versioning (e.g., `v1.0.0`) when pushing tags
- Branch name + commit SHA (e.g., `main-abc1234`) for branch pushes
- `latest` tag for the default branch

**Commit Message Format:**
The automated commits include `[skip ci]` to prevent workflow loops.

## Security Notes

1. **Secrets Management**: Use a secrets management solution like:
   - Sealed Secrets
   - External Secrets Operator
   - HashiCorp Vault
   - Cloud provider secrets manager

2. **Network Policies**: Implement network policies to restrict pod-to-pod communication

3. **Pod Security**: The deployments use non-root users and security contexts

4. **Resource Limits**: Adjust resource requests and limits based on your workload

5. **GitHub Token Security**: 
   - Store `K8S_REPO_TOKEN` as a GitHub secret (never commit it)
   - Use minimal permissions (only `contents: write` for the k8s repo)
   - Consider using a GitHub App instead of PAT for better security

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n foundry-idp
kubectl logs <pod-name> -n foundry-idp
```

### Database connection issues
Check the DATABASE_URL in secrets and ensure the database is accessible from the cluster.

### CORS issues
Verify CORS configuration in:
- `configmap-backend.yaml` (CORS_ORIGINS)
- `ingress.yaml` (nginx CORS annotations)
- Backend code (main.py)

### Image pull errors
Ensure:
- Images are pushed to GitHub Container Registry
- Service account has proper permissions
- Image pull secrets are configured if using private registry

