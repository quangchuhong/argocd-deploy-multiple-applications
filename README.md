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
## 2. Cấu trúc GitOps repo
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


