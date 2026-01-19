# ArgoCD Configuration - GitOps Deployment

**This folder contains ArgoCD Projects and Applications for automated deployments.**

---

## 📁 Structure

```
argocd/
├── projects/
│   └── chatapp-project.yaml       # ArgoCD Project (RBAC, allowed repos)
├── applications/
│   ├── chatapp-dev.yaml           # Dev environment app
│   ├── chatapp-staging.yaml       # Staging environment app
│   └── chatapp-prod.yaml          # Prod environment app (manual sync)
└── README.md                      # This file
```

---

## 🎯 Key Concepts

### **ArgoCD Project**
- Logical grouping of applications
- Defines RBAC (who can deploy what)
- Whitelists allowed repositories
- Whitelists allowed K8s resources

### **ArgoCD Application**
- Represents a single deployment (e.g., chatapp-dev)
- Points to Git repo + path
- Defines sync policy (auto vs manual)
- Manages health checks

---

## 🚀 Deployment Strategy

| Environment | Sync Mode | Prune | Self-Heal | Purpose |
|-------------|-----------|-------|-----------|---------|
| **Dev** | Automatic | ✅ Yes | ✅ Yes | Fast iteration |
| **Staging** | Automatic | ✅ Yes | ✅ Yes | Pre-prod testing |
| **Prod** | **Manual** | ❌ No | ❌ No | Safety first |

### **Dev & Staging:**
- **Auto-sync:** Git push → automatic deployment
- **Self-heal:** Cluster drift → auto-fix from Git
- **Prune:** Deleted from Git → deleted from cluster

### **Production:**
- **Manual sync:** Requires explicit approval
- **No self-heal:** Manual investigation required
- **No prune:** Safety - prevents accidental deletions

---

## 📋 Applying ArgoCD Configuration

### **Step 1: Install ArgoCD**

```bash
# Install ArgoCD in your EKS cluster
cd /path/to/Observability

# Add Argo CD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f infra-tf/helm/argocd/values.yaml \
  --wait

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=5m
```

### **Step 2: Access ArgoCD UI**

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# Username: admin
# Password: (from command above)
```

### **Step 3: Apply ArgoCD Project & Applications**

```bash
cd /path/to/CompanyGitOps

# Apply Project (defines RBAC and allowed resources)
kubectl apply -f argocd/projects/chatapp-project.yaml

# Apply Applications (one per environment)
kubectl apply -f argocd/applications/chatapp-dev.yaml
kubectl apply -f argocd/applications/chatapp-staging.yaml
kubectl apply -f argocd/applications/chatapp-prod.yaml
```

### **Step 4: Verify Deployment**

```bash
# Check ArgoCD applications status
kubectl get applications -n argocd

# View application details
argocd app get chatapp-dev

# Or use the UI: https://localhost:8080
```

---

## 🔄 Workflow: How It Works

### **1. Developer pushes code:**
```bash
git commit -m "Update backend"
git push origin develop
```

### **2. CI Pipeline runs:**
- Builds Docker image
- Pushes to ECR
- Updates image tag in `CompanyGitOps/applications/chatapp/dev/backend-deployment.yaml`
- Pushes change to GitOps repo

### **3. ArgoCD detects change:**
- **Dev:** Auto-syncs immediately (within 3 minutes)
- **Staging:** Auto-syncs immediately
- **Prod:** Shows "Out of Sync" - requires manual approval

### **4. Manual sync for Prod:**
```bash
# Via CLI
argocd app sync chatapp-prod

# Via UI
# 1. Open ArgoCD UI
# 2. Click "chatapp-prod"
# 3. Click "Sync" button
# 4. Review changes
# 5. Click "Synchronize"
```

---

## 🎯 Best Practices (Staff-Level)

### **1. Separation of Concerns** ✅
- **Observability repo:** Infrastructure (EKS, ArgoCD, monitoring)
- **CompanyGitOps repo:** Application configs, ArgoCD apps
- **ChatApplication repo:** Application code
- **JenkinsSharedLibrary:** CI/CD pipelines

### **2. Progressive Delivery** ✅
```
Dev (auto) → Staging (auto) → Prod (manual approval)
```

### **3. RBAC** ✅
- Dev team: Can sync dev only
- Ops team: Can sync all environments
- Viewers: Read-only access

### **4. Safety Features** ✅
- **Prod requires manual sync**
- **Sync windows** (optional - block prod deploys during business hours)
- **Orphaned resource detection**
- **Rollback capability**

### **5. Observability Integration** ✅
- ArgoCD metrics → Prometheus
- Deployment notifications → Slack
- Health checks integrated

---

## 🔧 Common Operations

### **View Application Status**
```bash
# List all applications
kubectl get applications -n argocd

# Get detailed status
argocd app get chatapp-dev

# View sync history
argocd app history chatapp-dev
```

### **Manual Sync**
```bash
# Sync specific app
argocd app sync chatapp-prod

# Sync and wait
argocd app sync chatapp-prod --wait
```

### **Rollback**
```bash
# View history
argocd app history chatapp-prod

# Rollback to previous version
argocd app rollback chatapp-prod <HISTORY_ID>
```

### **Diff Before Sync**
```bash
# See what will change
argocd app diff chatapp-prod
```

### **Delete Application** (careful!)
```bash
# Delete app from ArgoCD (keeps resources in cluster)
argocd app delete chatapp-dev

# Delete app and all resources
argocd app delete chatapp-dev --cascade
```

---

## 🚨 Troubleshooting

### **Application stuck "Progressing"**
```bash
# Check events
kubectl describe application chatapp-dev -n argocd

# Check application controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### **Sync failing**
```bash
# View sync errors
argocd app get chatapp-dev

# Check repo access
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server

# Manually validate manifests
kubectl apply --dry-run=client -f applications/chatapp/dev/
```

### **Application not auto-syncing**
```bash
# Check sync policy
kubectl get application chatapp-dev -n argocd -o yaml | grep -A 10 syncPolicy

# Force refresh
argocd app get chatapp-dev --refresh
```

---

## 📊 Monitoring ArgoCD

### **Prometheus Metrics**
ArgoCD exports metrics to Prometheus (configured in `Observability/infra-tf/helm/argocd/values.yaml`):

- `argocd_app_info` - Application info
- `argocd_app_sync_total` - Sync counts
- `argocd_app_k8s_request_total` - K8s API calls

### **Grafana Dashboards**
Import ArgoCD dashboards:
- Dashboard ID: 14584 (ArgoCD operational)
- Dashboard ID: 19993 (ArgoCD app metrics)

---

## 🔐 Security Best Practices

1. ✅ **Never commit secrets** to this repo
2. ✅ **Use AWS Secrets Manager** for sensitive data
3. ✅ **Rotate ArgoCD admin password** after initial setup
4. ✅ **Enable SSO** for production (GitHub/Google)
5. ✅ **Use RBAC** - principle of least privilege
6. ✅ **Audit logs** - track who deployed what

---

## 📚 Related Documentation

- [ArgoCD Official Docs](https://argo-cd.readthedocs.io/)
- [CompanyGitOps README](../README.md)
- [Application README](../applications/chatapp/README.md)
- [Observability README](../../Observability/README.md)

---

## ✅ Success Criteria

- [x] ArgoCD installed and accessible
- [x] Project created with RBAC
- [x] Applications deployed per environment
- [ ] Dev auto-syncing from Git
- [ ] Staging auto-syncing from Git
- [ ] Prod requiring manual approval
- [ ] Metrics flowing to Prometheus
- [ ] Team trained on ArgoCD usage



