# Step-by-Step Multi-Cluster GitOps Migration

## ✅ Completed Changes

### 1. Updated Environment Kustomizations
- **`env/dev/kustomization.yaml`**: Updated namespace from `online-boutique-dev` to `online-boutique`
- **`env/prod/kustomization.yaml`**: Updated namespace from `online-boutique-prod` to `online-boutique`
- **Rationale**: Since each environment has its own cluster, we no longer need environment-specific namespaces

### 2. Created Per-Cluster Application Manifests
- **`application-dev.yaml`**: Application manifest for DEV cluster
- **`application-prod.yaml`**: Application manifest for PROD cluster
- **Features**:
  - ✅ Argo CD Image Updater annotations configured
  - ✅ Semantic versioning constraints (`~v0.10` = patch updates only)
  - ✅ Git write-back method for persistent updates
  - ✅ Automatic sync with self-healing enabled

### 3. Preserved Legacy Configuration
- **`argocd-old-kargo.yaml`**: Renamed from `argocd.yaml` to preserve Kargo-based configuration

## 🎯 Next Steps (One at a time)

### Step 1: Deploy DEV Application

```bash
# Switch to DEV cluster context
kubectl config use-context gke_vupop-stg-wbx7_europe-west4-a_vupop-stg-cluster

# Apply the DEV application
kubectl apply -f application-dev.yaml

# Verify application is created
kubectl get applications -n argocd

# Monitor sync status
kubectl get application online-boutique-dev -n argocd -w
```

### Step 2: Verify DEV Deployment

```bash
# Check application status
kubectl describe application online-boutique-dev -n argocd

# Check if namespace was created
kubectl get namespace online-boutique

# Check pods deployment
kubectl get pods -n online-boutique

# Check services
kubectl get services -n online-boutique
```

### Step 3: (Optional) Test Image Updater on DEV

Before proceeding to PROD, you can test that Image Updater is working:

```bash
# Check Image Updater logs
kubectl logs -f deployment/argocd-image-updater -n argocd

# Manual test (if needed): force an image update annotation
kubectl annotate application online-boutique-dev -n argocd \
  argocd-image-updater.argoproj.io/force-update="true"
```

### Step 4: Deploy PROD Application (Only when ready)

```bash
# Switch to PROD cluster context
kubectl config use-context gke_vupop-prd-lahl_europe-west4_vupop-prd-cluster

# Apply the PROD application
kubectl apply -f application-prod.yaml

# Monitor deployment (same verification steps as DEV)
```

## 🔧 Key Configuration Details

### Image Updater Configuration
- **Version Constraint**: `~v0.10` allows updates from v0.10.0 to v0.10.x but NOT v0.11.0
- **Update Strategy**: `semver` (semantic versioning)
- **Write-back Method**: `git` (commits changes back to repository)
- **Target Branch**: `main`

### Automatic Sync Policy
- **Prune**: Removes resources not defined in Git
- **Self-Heal**: Automatically corrects configuration drift
- **Create Namespace**: Automatically creates target namespace

## 🔍 Monitoring Commands

### Check Application Status
```bash
# Get application overview
kubectl get applications -n argocd

# Detailed application status
kubectl describe application online-boutique-dev -n argocd

# Watch for changes
kubectl get application online-boutique-dev -n argocd -w
```

### Check Image Updater
```bash
# Image updater logs
kubectl logs deployment/argocd-image-updater -n argocd

# Image updater configuration
kubectl get configmap argocd-image-updater-config -n argocd -o yaml
```

### Check Deployed Resources
```bash
# All resources in namespace
kubectl get all -n online-boutique

# Specific resource types
kubectl get deployments,services,ingress -n online-boutique
```

## 🚨 Troubleshooting

### Common Issues

1. **Application Stuck in "Progressing"**
   ```bash
   kubectl describe application online-boutique-dev -n argocd
   ```

2. **Sync Failures**
   ```bash
   # Check for RBAC or permission issues
   kubectl logs deployment/argocd-application-controller -n argocd
   ```

3. **Image Updater Not Working**
   ```bash
   # Verify Image Updater is running and has proper RBAC
   kubectl get pods -n argocd | grep image-updater
   kubectl logs deployment/argocd-image-updater -n argocd
   ```

### Emergency Rollback
If needed, you can quickly return to the old configuration:
```bash
# Restore old ApplicationSet
mv argocd-old-kargo.yaml argocd.yaml
kubectl apply -f argocd.yaml

# Delete new applications
kubectl delete -f application-dev.yaml
kubectl delete -f application-prod.yaml
```

## 📝 Notes

- **No Cross-Cluster Dependencies**: Each cluster operates independently
- **Git as Source of Truth**: All configuration changes tracked in Git
- **Automatic Updates**: Image Updater handles container image updates
- **Environment Isolation**: Complete separation between dev and prod
