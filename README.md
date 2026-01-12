# Company GitOps Repository

## 🎯 Purpose

This repository contains **all deployment configurations** for all applications across all environments.

**Key Principles:**
- ✅ Company-wide (not app-specific)
- ✅ Modular and reusable (templates with variables)
- ✅ Version tracking per application per environment
- ✅ Single source of truth for deployments

---

## 📁 Structure

```
CompanyGitOps/
├── applications/              # All applications
│   ├── chatapp/               # Application 1
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   ├── otherapp/              # Application 2 (future)
│   └── template/              # Template for new applications
│
├── config/                     # Company-wide configuration
│   └── company-config.yaml
│
├── charts/                     # Shared Helm charts (optional)
│   ├── common/
│   └── templates/
│
└── argocd/                     # ArgoCD Application manifests (optional)
    ├── applications/
    └── projects/
```

---

## 🚀 Quick Start

### Adding a New Application

1. **Copy template**:
   ```bash
   cp -r applications/template applications/newapp
   ```

2. **Replace variables**:
   - Replace `{{APP_NAME}}` with `newapp`
   - Replace other variables from `config/company-config.yaml`

3. **CI pipeline will auto-update** deployment manifests

### Updating Deployment Version

CI pipeline automatically updates:
- `applications/{app}/{env}/deployment.yaml`

Manual override (if needed):
- Edit `deployment.yaml` directly
- Commit and push

---

## 📝 Files Per Environment

Each environment (`dev`, `staging`, `prod`) contains:

- **`deployment.yaml`** - Version tracking (auto-updated by CI)
- **`values.yaml`** - Helm values override
- **`config.yaml`** - Environment-specific configuration

---

## 🔧 Configuration

### Company Config

Edit `config/company-config.yaml` to set:
- Company name
- AWS account ID
- Domain names
- Common variables

### Application Config

Each application can override company config in:
- `applications/{app}/{env}/config.yaml`

---

## 🔄 Workflow

1. **CI builds image** → pushes to ECR
2. **CI updates** → `applications/{app}/{env}/deployment.yaml`
3. **ArgoCD/Flux syncs** → Auto-deploys to EKS
   - OR CD pipeline reads and deploys

---

## 📚 Documentation

- See `applications/template/` for template examples
- See `config/company-config.yaml` for company variables
- See application-specific READMEs in `applications/{app}/`

---

## 🔐 Security

- **Never commit secrets** to this repository
- Use AWS Secrets Manager
- Reference secrets by name in values.yaml
- Use Pod Identity for secret access

---

## ✅ Success Criteria

- [x] Repository structure created
- [ ] Templates with variables
- [ ] Company config file
- [ ] First application (chatapp) initialized
- [ ] CI pipeline integration
- [ ] CD pipeline/ArgoCD integration

