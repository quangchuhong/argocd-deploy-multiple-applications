# GitOps Platform trên EKS với Argo CD

## Thành phần

- **CI/CD & Tools**
  - Jenkins – CI
  - SonarQube – Code Quality
  - Trivy – Container Security Scanner
  - Prometheus + Grafana – Monitoring
  - **ELK Stack** – Logging (ElasticSearch, Kibana, FluentBit/Fluentd)
- **Applications**
  - `shopping-cart` – Ứng dụng Java 17 Maven (ví dụ)

Tất cả được quản lý bằng **Argo CD Application** theo mô hình GitOps.

---

## 1. Cấu trúc GitOps repo (mở rộng với ELK + app Java)

```text
.
├── apps
│   ├── root-app.yaml                 # App-of-apps (platform-tools)
│   ├── jenkins-app.yaml
│   ├── sonarqube-app.yaml
│   ├── monitoring-app.yaml           # prometheus + grafana
│   ├── tooling-app.yaml              # trivy operator, cronjobs, etc.
│   ├── elk-app.yaml                  # ELK stack
│   └── shopping-cart-app.yaml        # ứng dụng Java 17 Maven
└── helm
    ├── jenkins
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── sonarqube
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── monitoring                    # kube-prometheus-stack
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── tooling                       # trivy operator / jobs
    │   ├── Chart.yaml
    │   └── values.yaml
    ├── elk                           # elasticsearch + kibana + fluentbit
    │   ├── Chart.yaml
    │   └── values.yaml
    └── shopping-cart                 # ứng dụng Java 17
        ├── Chart.yaml
        └── values.yaml
