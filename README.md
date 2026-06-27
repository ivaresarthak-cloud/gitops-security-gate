# GitOps Security Gate — ArgoCD + Kubescape Integration

A production-ready GitOps security pipeline that automatically scans Kubernetes manifests for security misconfigurations before deployment. Combines ArgoCD for continuous deployment, Kubescape for security scanning, and comprehensive monitoring with Prometheus, Grafana, and Grafana Loki.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Repository                        │
│  manifests/ + .github/workflows/ + kind-cluster-config.yaml │
└────────────────────────┬──────────────────────────────────────┘
                         │ git push
                         ▼
         ┌───────────────────────────────┐
         │   GitHub Actions             │
         │  (Kubescape Security Scan)   │
         │ ✅ Pass  |  ❌ Fail → Alerts │
         └────┬──────────────────────────┘
              │
              ▼
        ┌──────────────┐
        │   ArgoCD     │
        │ Auto-Sync    │
        └──────┬───────┘
               │
               ▼
    ┌─────────────────────────┐
    │  Kubernetes Cluster     │
    │ ┌────────────────────┐  │
    │ │ Deployed Apps      │  │
    │ └────────────────────┘  │
    └────┬──────────────────┬──┘
         │                  │
    ┌────▼────────┐   ┌────▼──────────────┐
    │ Prometheus  │   │ Grafana + Loki    │
    │ Metrics     │   │ Dashboards & Logs │
    └─────────────┘   └───────────────────┘
         │
         ▼
    ┌────────────────┐
    │ Slack Alerts   │
    │ Notifications  │
    └────────────────┘
```

## ✨ Key Features

- **Automated Security Scanning**: Every push triggers Kubescape security analysis via GitHub Actions
- **Declarative Infrastructure**: ArgoCD continuously syncs Git manifests to Kubernetes cluster
- **Real-time Metrics**: Prometheus collects and Grafana visualizes cluster performance metrics
- **Log Aggregation**: Grafana Loki indexes and provides searchable access to container logs
- **Instant Notifications**: Slack alerts on security scan failures with direct links to failed workflows
- **Security Compliance**: Demonstrates security best practices through good/bad manifest examples
- **Local Development**: Complete setup using kind (Kubernetes in Docker) for local testing

## 🛠️ Technology Stack

| Component | Purpose | Version |
|-----------|---------|---------|
| Kubernetes | Container orchestration | kind v1.30.0 |
| ArgoCD | GitOps continuous deployment | v3.4.4 |
| Kubescape | Container security scanning | v4.0.9 |
| GitHub Actions | CI/CD pipeline automation | Native |
| Prometheus | Metrics collection & storage | kube-prometheus-stack |
| Grafana | Metrics visualization | kube-prometheus-stack |
| Grafana Loki | Log aggregation & indexing | grafana/loki-stack |
| Slack | Alert notifications | Webhooks |

## 📋 Prerequisites

- Docker Desktop (Windows/Mac) or Docker (Linux)
- WSL2 (Windows) or native Linux/Mac terminal
- Git and GitHub account
- kubectl (installed with kind)
- Helm 3.x
- Slack workspace access

## 🚀 Installation

### 1. Clone Repository
```bash
git clone https://github.com/ivaresarthak-cloud/gitops-security-gate.git
cd gitops-security-gate
```

### 2. Create Kubernetes Cluster
```bash
kind create cluster --config kind-cluster-config.yaml
kubectl get nodes  # Verify 3 nodes running
```

### 3. Deploy ArgoCD
```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --set server.service.type=NodePort
```

### 4. Create ArgoCD Application
```bash
kubectl apply -f - << 'YAML'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-security-gate
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ivaresarthak-cloud/gitops-security-gate
    targetRevision: master
    path: manifests/good
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML
```

### 5. Install Monitoring Stack
```bash
# Create monitoring namespace and install Prometheus/Grafana
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring

# Install Grafana Loki for log aggregation
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n monitoring
```

### 6. Access UIs

**ArgoCD UI:**
```bash
kubectl port-forward service/argocd-server -n argocd 8080:443 &
# URL: https://localhost:8080
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Grafana UI:**
```bash
kubectl port-forward service/kube-prometheus-stack-grafana -n monitoring 3000:80 &
# URL: http://localhost:3000
# Login: admin / admin123
```

### 7. Configure Slack Alerts
```bash
# Create webhook at https://api.slack.com/apps
# Add as GitHub repository secret
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

## 📊 CI/CD Pipeline

**Workflow File**: `.github/workflows/kubescape-scan.yaml`

**Execution Steps**:
1. Checkout repository code
2. Install Kubescape CLI from official sources
3. Scan manifests/good/ (security-compliant examples)
4. Scan manifests/bad/ (intentional misconfigurations for testing)
5. Alert Slack on any scan failures

**Trigger Events**: Commits to master branch, pull requests

## 🔐 Security Configuration

### Secure Manifest Example (manifests/good/)
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true

resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Insecure Manifest Example (manifests/bad/)
```yaml
securityContext:
  privileged: true
  runAsUser: 0

volumeMounts:
- name: host-root
  mountPath: /host

image: nginx:latest  # Unversioned image
```

## 📈 Monitoring

### Grafana Dashboards
- **Kubernetes / Compute Resources / Cluster**: CPU, memory, and pod utilization
- **Alertmanager**: Alert status and routing history
- Custom visualizations for security scan metrics

### Grafana Loki Queries
- Namespace-scoped logs: `{namespace="monitoring"}`
- Pod-specific logs: `{pod="deployment-xyz"}`
- Full-text search across all container logs

### Key Metrics Collected
- Per-namespace CPU utilization and limits
- Memory usage per pod and node
- Pod distribution and scheduling
- Network I/O and latency

## 🔔 Slack Integration

**Alert Channel**: `#security-alerts`

**Alert Format**:
```
🚨 Kubescape Security Scan FAILED!
Repository: ivaresarthak-cloud/gitops-security-gate
Branch: refs/heads/master
Commit: [commit-hash]
Check results: [GitHub Actions run link]
```

**Configuration**:
1. Create Slack app at https://api.slack.com/apps
2. Enable Incoming Webhooks feature
3. Create webhook pointing to #security-alerts channel
4. Store webhook URL as GitHub Actions secret

## 📚 Project Structure

```
gitops-security-gate/
├── manifests/
│   ├── good/
│   │   └── deployment.yaml          # Security-compliant deployment
│   └── bad/
│       └── deployment.yaml          # Intentional misconfigurations
├── .github/
│   └── workflows/
│       └── kubescape-scan.yaml      # Security scanning CI/CD pipeline
├── kind-cluster-config.yaml         # Local cluster configuration
└── README.md                         # This file
```

## 🎯 Use Cases

- **Security Assessment**: Test and validate security scanning capabilities
- **GitOps Learning**: Practical implementation of declarative infrastructure patterns
- **Compliance Testing**: Demonstrate security policy enforcement
- **Local Development**: Safe sandbox for Kubernetes and DevOps workflows
- **Production Foundation**: Extensible base for cloud deployments (AWS EKS, GKE, AKS)

## 🔧 Troubleshooting

**Kubescape CLI Not Found**
```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
export PATH="$HOME/.kubescape/bin:$PATH"
```

**Retrieve ArgoCD Admin Password**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Diagnose Pod Issues**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=100
```

**WSL2 DNS Resolution**
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

## 📖 Documentation

- [ArgoCD Official Docs](https://argoproj.github.io/argo-cd/)
- [Kubescape Security Framework](https://kubescape.io/docs/)
- [Prometheus Monitoring Guide](https://prometheus.io/docs/)
- [Grafana Dashboard Documentation](https://grafana.com/docs/grafana/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

## 📝 License

MIT License

## 👤 Author

**Sarthak Ivare**
- GitHub: [@ivaresarthak-cloud](https://github.com/ivaresarthak-cloud)
- LinkedIn: [Sarthak Ivare](https://linkedin.com/in/sarthak-ivare-94b25b3a3)

---

**Version**: 1.0.0  
**Last Updated**: June 27, 2026  
**Status**: Production Ready
