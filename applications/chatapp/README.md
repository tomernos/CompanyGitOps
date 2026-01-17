# ChatApp - GitOps Configuration

**This directory contains all deployment configurations for ChatApp across all environments.**

---

## 📁 Structure

```
chatapp/
├── helm-chart/                      # Modular Helm chart (copied from ChatApplication)
│   ├── Chart.yaml                   # Helm chart definition
│   ├── values.yaml                  # Default values
│   ├── charts/                      # Subcharts (microservices)
│   │   ├── backend/                 # Backend service
│   │   ├── frontend/                # Frontend service
│   │   ├── postgres/                # Database (optional)
│   │   ├── rabbitmq/                # Message queue (optional)
│   │   └── redis/                   # Cache (optional)
│   └── templates/                   # Shared templates
│       ├── ingress.yaml
│       ├── serviceaccount.yaml
│       └── ...
│
├── dev/
│   ├── values.yaml                  # Dev environment overrides
│   ├── config.yaml                  # Dev-specific config
│   └── deployment.yaml              # Version tracking (CI updates this)
│
├── staging/
│   ├── values.yaml                  # Staging environment overrides
│   └── deployment.yaml              # Version tracking
│
├── prod/
│   ├── values.yaml                  # Prod environment overrides
│   └── deployment.yaml              # Version tracking
│
└── README.md                        # This file
```

---

## 🎯 How It Works

### **1. Helm Chart (Shared)**
- **Location:** `helm-chart/`
- **Purpose:** Reusable, modular chart with subcharts for each service
- **Managed by:** Developers (copied from ChatApplication repo)
- **Updates:** When chart structure changes (rare)

### **2. Environment Values (Per Environment)**
- **Location:** `dev/values.yaml`, `staging/values.yaml`, `prod/values.yaml`
- **Purpose:** Environment-specific overrides (replicas, resources, URLs)
- **Managed by:** DevOps team
- **Updates:** When environment config changes

### **3. Version Tracking (Per Environment)**
- **Location:** `dev/deployment.yaml`, `staging/deployment.yaml`, `prod/deployment.yaml`
- **Purpose:** Track deployed image versions
- **Managed by:** CI Pipeline (auto-updated)
- **Updates:** On every build

---

## 🔄 Deployment Workflow

### **CI Pipeline (ChatApplication → CompanyGitOps)**

```bash
1. Developer pushes code to ChatApplication repo
   ↓
2. Jenkins CI builds image
   - Builds: chatapp-backend:dev-abc123
   - Pushes to ECR
   ↓
3. CI updates CompanyGitOps
   - Updates: applications/chatapp/dev/deployment.yaml
   - Changes: backend_image: chatapp-backend:dev-abc123
   - Updates: helm-chart/values-dev.yaml (if needed)
   - Commits & pushes
   ↓
4. ArgoCD detects change
   - Dev: Auto-syncs (3 min)
   - Staging: Auto-syncs (3 min)
   - Prod: Manual approval required
```

---

## 📝 Values File Hierarchy

ArgoCD merges values in this order:

```
1. helm-chart/values.yaml           (defaults)
   ↓ overridden by
2. helm-chart/values-{env}.yaml     (deprecated - for reference only)
   ↓ overridden by
3. {env}/values.yaml                (active - used by ArgoCD)
```

**Active files used by ArgoCD:**
- `dev/values.yaml`
- `staging/values.yaml`
- `prod/values.yaml`

---

## 🚀 How to Update

### **Update Application Code**
```bash
# CI handles this automatically!
# Just push code to ChatApplication repo
git push origin develop
```

### **Update Environment Configuration**
```bash
cd CompanyGitOps/applications/chatapp

# Edit environment values
vim dev/values.yaml

# Examples:
# - Increase replicas: backend.replicaCount: 3
# - Change resources: backend.resources.limits.memory: 1Gi
# - Update URLs: ingress.hosts[0].host: new-domain.com

# Commit and push
git add dev/values.yaml
git commit -m "Increase dev backend replicas to 3"
git push origin main

# ArgoCD will auto-sync within 3 minutes
```

### **Update Helm Chart Structure**
```bash
# 1. Update chart in ChatApplication repo
cd ChatApplication/helm-chart
# ... make changes ...
git commit -m "Add new service"
git push

# 2. Copy updated chart to GitOps
cd CompanyGitOps/applications/chatapp
cp -r ../../../../ChatApplication/helm-chart ./

# 3. Commit and push
git add helm-chart/
git commit -m "Update Helm chart with new service"
git push origin main

# ArgoCD will detect and deploy changes
```

---

## 🎯 Environment-Specific Overrides

### **Dev Environment** (`dev/values.yaml`)
```yaml
backend:
  enabled: true
  replicaCount: 1              # Low resources for dev
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
  monitoring:
    enabled: true              # Prometheus scraping

ingress:
  enabled: true
  hosts:
    - host: chatapp-dev.tomernos.xyz
      paths:
        - path: /api
          serviceName: chatapp-backend
          servicePort: 5000
```

### **Prod Environment** (`prod/values.yaml`)
```yaml
backend:
  enabled: true
  replicaCount: 3              # HA with multiple replicas
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  monitoring:
    enabled: true

ingress:
  enabled: true
  hosts:
    - host: chatapp.tomernos.xyz
      paths:
        - path: /api
          serviceName: chatapp-backend
          servicePort: 5000
  tls:
    - secretName: chatapp-tls
```

---

## 🔧 Testing Locally

### **Validate Helm Chart**
```bash
cd helm-chart

# Lint chart
helm lint .

# Dry-run for dev
helm install chatapp-dev . \
  -f ../dev/values.yaml \
  --dry-run --debug \
  --namespace chatapp-dev

# Template and check output
helm template chatapp-dev . \
  -f ../dev/values.yaml \
  --namespace chatapp-dev
```

### **Test ArgoCD Application**
```bash
# Validate ArgoCD app manifest
kubectl apply --dry-run=client -f ../../argocd/applications/chatapp-dev.yaml

# Preview what ArgoCD will deploy
argocd app diff chatapp-dev
```

---

## 🎓 Best Practices

### **✅ DO:**
- Keep Helm chart in sync with ChatApplication repo
- Use environment-specific values files for overrides
- Test changes in dev before promoting to staging/prod
- Document major configuration changes
- Use semantic versioning for chart updates

### **❌ DON'T:**
- Don't edit `deployment.yaml` manually (CI updates it)
- Don't hardcode secrets in values files
- Don't bypass ArgoCD by using `kubectl apply`
- Don't modify prod without testing in staging first

---

## 🔒 Secrets Management

**Never commit secrets to this repo!**

```yaml
# BAD - Don't do this:
database:
  password: "my-secret-password"

# GOOD - Use external secrets:
database:
  passwordSecretName: "chatapp-db-password"
  passwordSecretKey: "password"
```

Secrets are managed via:
- AWS Secrets Manager
- Kubernetes Secrets (referenced by name)
- External Secrets Operator (future)

---

## 📊 Monitoring

Each environment deployment is monitored via:
- **Prometheus:** Metrics scraping (ServiceMonitor)
- **Grafana:** Dashboards (metrics, traces, logs)
- **Jaeger:** Distributed tracing
- **Loki:** Log aggregation

See: [Observability Stack Documentation](../../../Observability/README.md)

---

## 🆘 Troubleshooting

### **ArgoCD shows "OutOfSync"**
```bash
# Check what's different
argocd app diff chatapp-dev

# Manual sync if needed
argocd app sync chatapp-dev
```

### **Helm chart validation fails**
```bash
# Test locally
cd helm-chart
helm lint .
helm template . -f ../dev/values.yaml --debug
```

### **Pods not starting**
```bash
# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# Check pod events
kubectl describe pod -n chatapp-dev -l app=backend
```

---

## 📚 Related Documentation

- [Helm Chart README](./helm-chart/README.md)
- [ArgoCD Applications](../../argocd/applications/)
- [CI/CD Pipeline](../../../ChatApplication/docs/jenkins/)
- [Observability Stack](../../../Observability/docs/)

---

## ✅ Checklist

- [x] Helm chart copied from ChatApplication
- [x] Environment values organized (dev/staging/prod)
- [x] ArgoCD applications configured for Helm
- [ ] CI pipeline updates image tags in values
- [ ] Secrets externalized
- [ ] Tested in dev environment
