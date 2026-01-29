# Kiến trúc & Guide triển khai Platform Tools và Business Apps với Argo CD trên EKS

Tài liệu này mô tả kiến trúc GitOps trên **Amazon EKS** với **Argo CD**, dùng để tự động triển khai:

- **Platform Tools (project: `platform`)**
  - Jenkins – CI
  - SonarQube – Code Quality
  - Trivy – Security scanner (operator/cronjobs)
  - Prometheus + Grafana – Monitoring
  - ELK – Logging (Elasticsearch + Kibana + FluentBit)
- **Business Applications (project: `business`)**
  - Ví dụ: `shopping-cart` (Java 17 Maven), `payment-service` (Python), ...

---

## 1. Kiến trúc tổng quan

```text
                   +--------------------------+
                   |        GitLab            |
                   |  GitOps Repos & AppCode  |
                   +------------+-------------+
                                |
                                | (git clone / fetch)
                                v
+----------------------------------------------------------+
|                    Amazon EKS Cluster                    |
|                                                          |
|  +------------ Namespace: argocd ---------------------+  |
|  |   +---------------------+     +------------------+ |  |
|  |   | argocd-server       |<--->| argocd-redis     | |  |
|  |   | (UI/API)            |     | (cache)          | |  |
|  |   +----------+----------+     +--------+---------+ |  |
|  |              |                         ^           |  |
|  |              v                         |           |  |
|  |   +----------------------+   +---------+--------+  |  |
|  |   | argocd-app-controller|<->| argocd-repo-server| |  |
|  |   | (reconcile Git <->   |   | (Git, Helm,       | |  |
|  |   |  cluster)            |   |  Kustomize render)| |  |
|  |   +----------------------+   +------------------+ |  |
|  +--------------------------------------------------+  |
|                                                          |
|  +---------- Platform Namespaces (project: platform) ----+
|  |  - jenkins        (Jenkins CI)                       |
|  |  - sonarqube      (code quality)                     |
|  |  - monitoring     (Prometheus + Grafana)             |
|  |  - elk            (Elasticsearch + Kibana + Fluent)  |
|  |  - tooling        (Trivy operator / jobs)            |
|  +------------------------------------------------------+
|  +---------- Business Namespaces (project: business) ----+
|  |  - shopping-cart   (Java 17 app)                      |
|  |  - payment-service (Python app)                       |
|  |  - ...                                                |
|  +------------------------------------------------------+
+----------------------------------------------------------+

                +------------------------+
                | Amazon ECR (Images)    |
                +------------------------+
```
---
## 2. Cấu trúc GitOps repo
```text
.
├── apps
│   ├── platform-tools.yaml           # Root ArgoCD app cho tools
│   ├── business-apps.yaml            # Root ArgoCD app cho business apps
│   ├── tools
│   │   ├── jenkins-app.yaml
│   │   ├── sonarqube-app.yaml
│   │   ├── monitoring-app.yaml       # kube-prometheus-stack
│   │   ├── elk-app.yaml              # Elasticsearch + Kibana + FluentBit
│   │   └── tooling-app.yaml          # Trivy operator / jobs
│   └── business
│       ├── shopping-cart-app.yaml    # Java 17 app
│       └── payment-service-app.yaml  # Python app (ví dụ)
└── helm
    ├── jenkins
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── sonarqube
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── monitoring                 # kube-prometheus-stack
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── elk                        # elasticsearch + kibana + fluentbit
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── tooling                    # trivy operator / jobs
    │   ├── Chart.yaml
    │   └── values.yaml
    └── shopping-cart              # ứng dụng Java 17
        ├── Chart.yaml
        └── values.yaml

```
---
## 3. AppProject – Phân tách quyền & phạm vi
### 3.1. AppProject cho platform
```text
# projects/platform-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Platform tools (Jenkins, SonarQube, Prometheus, Grafana, ELK, Trivy, ...)

  sourceRepos:
    - https://gitlab.com/your-group/gitops-platform.git

  destinations:
    - namespace: jenkins
      server: https://kubernetes.default.svc
    - namespace: sonarqube
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
    - namespace: elk
      server: https://kubernetes.default.svc
    - namespace: tooling
      server: https://kubernetes.default.svc
    - namespace: argocd
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

  roles:
    - name: platform-admin
      description: Full access to platform apps
      policies:
        - p, proj:platform:platform-admin, applications, *, platform/*, allow
      groups: []
```
---
### 3.2. AppProject cho business
```text
# projects/business-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: business
  namespace: argocd
spec:
  description: Business applications (Java, Python, ...)

  sourceRepos:
    - https://gitlab.com/your-group/gitops-platform.git

  destinations:
    - namespace: shopping-cart
      server: https://kubernetes.default.svc
    - namespace: payment-service
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

  roles:
    - name: business-admin
      description: Full access to business apps
      policies:
        - p, proj:business:business-admin, applications, *, business/*, allow
      groups: []

```
### Apply 2 AppProject:
```text
kubectl apply -f projects/platform-project.yaml -n argocd
kubectl apply -f projects/business-project.yaml -n argocd
```
---
## 4. Root Application – App-of-Apps
### 4.1. Root app cho Platform Tools
```text
# apps/platform-tools.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-tools
  namespace: argocd
spec:
  project: platform

  source:
    repoURL: https://gitlab.com/your-group/gitops-platform.git
    targetRevision: main
    path: apps/tools

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true

  revisionHistoryLimit: 10
```
Thư mục apps/tools/ chứa:
```text
apps/tools/
├── jenkins-app.yaml
├── sonarqube-app.yaml
├── monitoring-app.yaml
├── elk-app.yaml
└── tooling-app.yaml
```
### 4.2. Root app cho Business Apps
```text
# apps/business-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: business-apps
  namespace: argocd
spec:
  project: business

  source:
    repoURL: https://gitlab.com/your-group/gitops-platform.git
    targetRevision: main
    path: apps/business

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true

  revisionHistoryLimit: 10
```
Thư mục apps/business/ chứa:
```text
apps/business/
├── shopping-cart-app.yaml
└── payment-service-app.yaml

```
### Apply 2 root-app:
```text
kubectl apply -f apps/platform-tools.yaml -n argocd
kubectl apply -f apps/business-apps.yaml -n argocd
```
---
## 5. Các Application con (ví dụ)
### 5.1. Jenkins (app tools)
```text
# apps/tools/jenkins-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: jenkins
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://gitlab.com/your-group/gitops-platform.git
    targetRevision: main
    path: helm/jenkins
    helm:
      releaseName: jenkins
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: jenkins
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```
---
### 5.2. Ứng dụng Java 17 – shopping-cart
```text
# apps/business/shopping-cart-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shopping-cart
  namespace: argocd
spec:
  project: business
  source:
    repoURL: https://gitlab.com/your-group/gitops-platform.git
    targetRevision: main
    path: helm/shopping-cart
    helm:
      releaseName: shopping-cart
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: shopping-cart
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```
---
## 6. Luồng CI/CD + GitOps
### - 1. Developer push code lên GitLab (repo ứng dụng).
### - 2. Jenkins:
Build Maven (Java 17),
SonarQube scan,
Docker build & Trivy scan,
Push image lên ECR,
Update helm/shopping-cart/values.yaml (image.tag) trong repo GitOps và commit.
### - 3. Argo CD:
Thấy GitOps repo có commit mới,
Application shopping-cart OutOfSync,
Auto-sync → cập nhật Deployment trong namespace shopping-cart trên EKS.
### - 4. ELK thu thập logs của tất cả pods,
### - 5. Prometheus + Grafana thu metrics,
### - 6. Dev/Platform quan sát hệ thống qua Grafana, Kibana, ArgoCD UI.






---

