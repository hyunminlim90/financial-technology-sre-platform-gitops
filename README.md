# fin-tech-sre-platform-gitops

GitOps repository for the Financial Technology SRE Platform.

All platform components are managed declaratively through ArgoCD.  
No manual `kubectl apply` or `helm install` in production — everything goes through Git.

---

## Architecture Overview

```
Cloudflare Tunnel
      ↓
Istio IngressGateway (NodePort 30660/30870)
      ↓
VirtualService (ft-sre-*.opentofu.click)
      ↓
Kubernetes Services (observability / argocd / tracing / data / fintech)
```

**Cluster Nodes**

| Node | IP | Role | Label |
|---|---|---|---|
| control-plane-1 | 172.30.1.109 | Control Plane + ArgoCD + Istio | node-role=platform |
| app-node-1 | 172.30.1.106 | Application Workloads | node-role=app |
| data-node-1 | 172.30.1.107 | Data Layer | node-role=data |
| obs-node-1 | 172.30.1.108 | Observability Stack | node-role=observability |
| gateway | 172.30.1.105 | Cloudflare Tunnel / Nginx / CI-CD | (K8s 외부) |

---

## GitOps Design

### App of Apps Pattern

ArgoCD는 `bootstrap/root-app.yaml` 하나만 수동으로 등록한다.  
이후 모든 Application은 `bootstrap/apps/` 아래 선언으로 자동 배포된다.

```
bootstrap/root-app.yaml          ← ArgoCD에 수동 등록 (1회)
bootstrap/apps/
  ├── monitoring.yaml            ← kube-prometheus-stack Application
  ├── kiali-operator.yaml        ← Kiali Operator Application
  ├── kiali.yaml                 ← Kiali CR Application
  ├── tracing.yaml               ← OpenTelemetry + Jaeger Application
  ├── argocd-projects.yaml       ← AppProject 정의
  └── ...
```

### Project 구조

| Project | Namespace | 담당 |
|---|---|---|
| observability | observability, kiali-operator | Prometheus, Grafana, Alertmanager, Kiali |
| data | data, tracing, elastic-system | MySQL, Redis, Kafka, Elasticsearch, Jaeger |
| apps | fintech, sre-agent | Spring Boot API, SRE Agent |

### 도메인 규칙

모든 외부 노출 서비스는 `ft-sre-<name>.opentofu.click` 형식을 따른다.

| 서비스 | 도메인 |
|---|---|
| Grafana | ft-sre-grafana.opentofu.click |
| Prometheus | ft-sre-prometheus.opentofu.click |
| Alertmanager | ft-sre-alertmanager.opentofu.click |
| Kiali | ft-sre-kiali.opentofu.click |
| ArgoCD | ft-sre-argocd.opentofu.click |
| Jaeger | ft-sre-jaeger.opentofu.click |
| App (Spring Boot) | ft-sre-app.opentofu.click |

---

## Repository Structure

```
fin-tech-sre-platform-gitops/
│
├── bootstrap/                         # App of Apps 진입점
│   ├── root-app.yaml                  # ArgoCD Root Application (수동 등록)
│   ├── projects.yaml                  # AppProject 정의 (observability / data / apps)
│   └── apps/
│       ├── monitoring.yaml            # kube-prometheus-stack Application
│       ├── kiali-operator.yaml        # Kiali Operator Application
│       ├── kiali.yaml                 # Kiali CR Application
│       ├── tracing.yaml               # OpenTelemetry Operator + Jaeger Application
│       └── (추후) argocd.yaml / data.yaml / apps.yaml
│
├── observability/
│   ├── kube-prometheus-stack/
│   │   └── values.yaml                # Helm values (nodeSelector / storage / grafana admin)
│   ├── kiali/
│   │   ├── kiali-cr.yaml              # Kiali CR (kiali.kiali.io)
│   │   └── kiali-istio.yaml           # Kiali Gateway + VirtualService
│   └── tracing/
│       ├── otel-collector.yaml        # OpenTelemetryCollector CR (Jaeger v2)
│       ├── jaeger-istio.yaml          # Jaeger Gateway + VirtualService
│       └── istio-telemetry.yaml       # Istio Telemetry (sampling 설정)
│
├── data/
│   ├── mysql/
│   │   ├── values.yaml
│   │   └── istio.yaml                 # Gateway + VirtualService (필요시)
│   ├── redis/
│   │   └── values.yaml
│   ├── kafka/
│   │   └── values.yaml
│   ├── elasticsearch/
│   │   └── values.yaml                # ECK Operator or Helm
│   └── oracle-xe/
│       └── values.yaml
│
├── apps/
│   ├── fintech-api/
│   │   ├── values.yaml                # Spring Boot WebFlux
│   │   └── istio.yaml
│   └── sre-agent/
│       ├── values.yaml
│       └── istio.yaml
│
└── platform/
    ├── istio/
    │   ├── istio-operator.yaml        # IstioOperator (MeshConfig + extensionProviders)
    │   └── telemetry.yaml             # mesh-default-tracing
    └── argocd/
        └── argocd-values.yaml         # ArgoCD Helm values
```

---

## Bootstrap 절차

처음 클러스터에 GitOps를 적용할 때 단 한 번만 수동으로 수행한다.

```bash
# 1. ArgoCD 설치 (이미 완료)
helm upgrade --install argocd argo/argo-cd -n argocd -f platform/argocd/argocd-values.yaml

# 2. Root Application 등록 (이 한 번으로 나머지는 자동)
kubectl apply -f bootstrap/root-app.yaml
```

이후 모든 변경은 Git commit → ArgoCD 자동 Sync로만 이루어진다.

---

## Sync 정책

모든 Application은 아래 정책을 기본으로 한다.

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

kube-system 컴포넌트 Service 등 ArgoCD가 관리하지 않는 리소스는 `ignoreDifferences`로 처리한다.

---

## 현재 배포 상태 (2026-04-26 기준)

| 컴포넌트 | 네임스페이스 | 상태 | 접속 도메인 |
|---|---|---|---|
| kube-prometheus-stack | observability | Running | ft-sre-grafana / prometheus / alertmanager |
| Kiali Operator | kiali-operator | Running | - |
| Kiali | observability | Running | ft-sre-kiali.opentofu.click |
| ArgoCD | argocd | Running | ft-sre-argocd.opentofu.click |
| OpenTelemetry Operator | opentelemetry-operator-system | Running | - |
| Jaeger v2 | tracing | 구성 중 | ft-sre-jaeger.opentofu.click |
| Istio | istio-system | Running | (IngressGateway) |

---

## 다음 단계

1. data/ 구성 — MySQL / Redis / Kafka / Elasticsearch / Oracle XE StatefulSet
2. Jaeger storage → Elasticsearch 전환
3. apps/ 구성 — Spring Boot WebFlux API, SRE Agent
4. SRE Agent + RAG + LLM Gateway 배포