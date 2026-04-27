# 🚀 FinTech SRE Platform – GitOps Repository

> ArgoCD 기반 Kubernetes 운영을 위한 Single Source of Truth

---

## 📌 Overview

이 저장소는 FinTech SRE Platform의 Kubernetes 배포 상태 (desired state) 를 관리하는 GitOps 저장소입니다.

- ArgoCD가 이 저장소를 기준으로 클러스터를 동기화합니다.
- 모든 인프라/플랫폼 구성은 `Git → ArgoCD → Kubernetes` 흐름으로 반영됩니다.
- 수동 `kubectl apply`는 금지하며, 반드시 Git을 통해 변경합니다.

---

## 🧭 Architecture

```
Git (this repo)
    ↓
ArgoCD (App of Apps)
    ↓
Kubernetes Cluster
    ↓
Istio Gateway + Cloudflare Tunnel
    ↓
External Traffic
```

---

## 🧱 Cluster Topology

| Node | IP | Role |
|---|---|---|
| control-plane-1 | 172.30.1.109 | platform |
| app-node-1 | 172.30.1.106 | application |
| data-node-1 | 172.30.1.107 | data |
| obs-node-1 | 172.30.1.108 | observability |

**Node Role Labels**

```
node-role=platform
node-role=app
node-role=data
node-role=observability
```

---

## ⚙️ Core Stack

**Kubernetes**
- kubeadm 기반 cluster
- containerd runtime
- Calico CNI

**Service Mesh**
- Istio
- IngressGateway (NodePort)
- Gateway / VirtualService 기반 라우팅

**External Access**
- Cloudflare Tunnel
- HTTPS → Cloudflare → HTTP → Istio

---

## 🔄 GitOps Architecture

### App of Apps Pattern

```
ft-sre-root
    ├── projects.yaml
    ├── root-app.yaml
    └── apps/
        ├── monitoring.yaml
        ├── kiali.yaml
        ├── kiali-operator.yaml
        ├── opentelemetry-operator.yaml
        ├── eck-operator.yaml
        ├── tracing-elasticsearch.yaml
        └── jaeger.yaml
```

**Root Application**

```yaml
path: bootstrap
directory:
  recurse: true
```

---

## 🗂️ Repository Structure

```
bootstrap/
├── root-app.yaml
├── projects.yaml
└── apps/

observability/
├── kube-prometheus-stack/
│   └── values.yaml
├── kiali/
│   ├── kiali-cr.yaml
│   └── kiali-istio.yaml
└── tracing/
    ├── jaeger.yaml
    └── jaeger-istio.yaml

data/
└── elasticsearch/
    └── tracing-es.yaml
```

---

## 🧩 ArgoCD Project Design

| Project | Namespaces |
|---|---|
| observability | observability, kiali-operator, kube-system, opentelemetry-operator-system |
| data | data, tracing, elastic-system |
| apps | fintech, sre-agent |

---

## 📊 Observability Stack

**Metrics**
- Prometheus
- Grafana
- Alertmanager
- kube-state-metrics
- node-exporter

**Service Mesh Visibility**
- Kiali

---

## 🔍 Tracing Architecture

```
Istio
  ↓ (trace export)
OpenTelemetry (OTLP)
  ↓
Jaeger Collector
  ↓
Elasticsearch (ECK)
  ↓
Jaeger Query (UI)
  ↓
Istio Gateway → Cloudflare
```

**Components**
- OpenTelemetry Operator (CRD 기반)
- Jaeger v2 (Deployment + ConfigMap)
- Elasticsearch (ECK Operator 기반)
- OTLP (gRPC 4317 / HTTP 4318)
- Jaeger UI (16686)

---

## ⚠️ Key Issues & Learnings

### 1. CRD Annotation Limit

```
metadata.annotations too long
```

✅ 해결:

```yaml
syncOptions:
  - ServerSideApply=true
```

### 2. Grafana PVC Permission Issue

```
init-chown-data CrashLoopBackOff
```

✅ 해결: PVC 재생성 + GitOps 재배포

### 3. ArgoCD Project Permission Error

```
namespace not permitted
```

✅ 해결:

```yaml
destinations:
  - namespace: kube-system
```

### 4. OpenTelemetry Helm Chart Schema Error

```
manager.collectorImage.repository must be set
```

- 원인: Helm chart schema validation
- 해결: values.yaml 명시적 설정

### 5. ArgoCD Destination Denied

```
namespace opentelemetry-operator-system not allowed
```

✅ 해결: AppProject destination 추가

### 6. StatefulSet Immutable Field Error (ECK)

```
updates to statefulset spec are forbidden
```

✅ 해결: StatefulSet 삭제 후 재생성

### 7. Jaeger v2 + OpenTelemetryCollector Conflict

```
invalid config (metrics.address / elasticsearch auth)
```

- 원인: Operator-generated config incompatibility
- 해결: Deployment + ConfigMap 방식으로 전환 (실무형 선택)

---

## 🌐 Traffic Flow

```
User
 → Cloudflare
 → Cloudflare Tunnel
 → Istio IngressGateway
 → Gateway
 → VirtualService
 → Service
 → Pod
```

---

## 🚨 Operational Rules

| 규칙 | |
|---|---|
| `kubectl apply` 직접 사용 | ❌ 금지 |
| Git PR 기반 변경 | ✅ 필수 |
| ArgoCD 자동 Sync (self-heal) | ✅ 활성화 |
| 수동 리소스 변경 | ❌ 금지 |
| Helm values 기반 관리 | ✅ 필수 |

---

## 🔮 Next Steps

**Observability**
- Istio → Jaeger trace export 연결
- Alert system (Prometheus → Slack)
- SLO / Error Budget

**Data Layer**
- MySQL
- Redis
- Kafka
- Elasticsearch 확장

**Application Layer**
- api-server
- worker
- web-console
- sre-agent

---

## 🎯 Goal

> 단순 배포가 아니라  
> SRE 관점의 "운영 가능한 플랫폼"을 Git으로 정의

---

## 🧠 Related Repository

| 역할 | 저장소 |
|---|---|
| 설계 / 문서 / RAG / Runbook | fin-tech-sre-platform-portfolio |
| Kubernetes 상태 관리 | fin-tech-sre-platform-gitops (this repo) |

---

## 📌 Summary

| 항목 | 상태 |
|---|---|
| Cluster | ✅ Ready |
| Istio | ✅ Routing OK |
| Cloudflare | ✅ External Access OK |
| ArgoCD | ✅ GitOps OK |
| Monitoring | ✅ Running |
| Tracing | ✅ Running (Jaeger + Elasticsearch) |
| OpenTelemetry | ✅ Running |

> Production-grade SRE Platform Base (Observability + Tracing) 완료