# 🚀 FinTech SRE Platform – GitOps Repository

> ArgoCD 기반 Kubernetes 운영을 위한 **Single Source of Truth**

---

## 📌 Overview

이 저장소는 FinTech SRE Platform의  **Kubernetes 배포 상태(desired state)** 를 관리하는 GitOps 저장소입니다.

* ArgoCD가 이 저장소를 기준으로 클러스터를 동기화합니다.
* 모든 인프라/플랫폼 구성은 **Git → ArgoCD → Kubernetes** 흐름으로 반영됩니다.
* 수동 kubectl 적용은 금지하며, 반드시 Git을 통해 변경합니다.

---

## 🧭 Architecture

```text
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

```text
control-plane-1   (172.30.1.109)  → platform
app-node-1        (172.30.1.106)  → application
data-node-1       (172.30.1.107)  → data
obs-node-1        (172.30.1.108)  → observability
```

### Node Role Label

```bash
node-role=platform
node-role=app
node-role=data
node-role=observability
```

---

## ⚙️ Core Stack

### Kubernetes

* kubeadm 기반 cluster
* containerd runtime
* Calico CNI

### Service Mesh

* Istio
* IngressGateway (NodePort)
* VirtualService / Gateway 기반 라우팅

### External Access

* Cloudflare Tunnel
* HTTPS → Cloudflare → HTTP → Istio

---

## 🔄 GitOps Architecture

### App of Apps Pattern

```text
ft-sre-root
    ├── projects.yaml
    ├── root-app.yaml
    └── apps/
        ├── monitoring.yaml
        ├── kiali.yaml
        └── kiali-operator.yaml
        ...
        ...
```

### Root Application

```yaml
path: bootstrap
directory:
  recurse: true
```

---

## 🗂️ Repository Structure

```text
bootstrap/
├── root-app.yaml        # ArgoCD root app (entry point)
├── projects.yaml        # AppProject 정의
└── apps/                # 실제 Application 정의

observability/
├── kube-prometheus-stack/
│   └── values.yaml
└── kiali/
    ├── kiali-cr.yaml
    └── kiali-istio.yaml

data/
├── mysql/
├── redis/
├── kafka/
├── elasticsearch/
└── oracle-xe/
```

---

## 🧩 ArgoCD Project Design

### observability

* namespace: observability
* namespace: kiali-operator
* namespace: kube-system (monitoring exporter용)

### data

* namespace: data
* namespace: tracing
* namespace: elastic-system

### apps

* namespace: fintech
* namespace: sre-agent

---

## 📊 Observability Stack

### kube-prometheus-stack

* Prometheus
* Grafana
* Alertmanager
* kube-state-metrics
* node-exporter

### 특징

* PVC 기반 persistent storage
* Helm chart 기반 GitOps 관리
* ServerSideApply 적용 (CRD 대응)

---

## ⚠️ Key Issues & Learnings

### 1. CRD Annotation Limit

```text
metadata.annotations too long
```

✔ 해결:

```yaml
syncOptions:
- ServerSideApply=true
```

---

### 2. Grafana PVC Permission Issue

```text
init-chown-data CrashLoopBackOff
```

✔ 원인:

* local-path storage 권한 문제

✔ 해결:

* PVC 재생성 + GitOps 재배포

---

### 3. ArgoCD Project Permission Error

```text
namespace kube-system is not permitted
```

✔ 해결:

```yaml
destinations:
- namespace: kube-system
```

---

### 4. Root App Recursive Sync

```text
apps not detected
```

✔ 해결:

```yaml
directory:
  recurse: true
```

---

## 🌐 Traffic Flow

```text
User
 → Cloudflare
 → Cloudflare Tunnel (cloudflared)
 → NodePort (Istio IngressGateway)
 → Gateway
 → VirtualService
 → Kubernetes Service
 → Pod
```

---

## 🚨 Operational Rules

* ❌ kubectl apply 직접 금지
* ✅ 모든 변경은 Git PR 기반
* ✅ ArgoCD 자동 Sync (self-heal)
* ❌ 수동 리소스 변경 금지
* ✅ values.yaml 기반 Helm 관리

---

## 🔮 Next Steps

### Tracing (진행 예정)

* OpenTelemetry Operator
* OpenTelemetry Collector
* Jaeger v2
* Elasticsearch backend

### Data Layer

* MySQL
* Redis
* Kafka
* Elasticsearch

### Application Layer

* api-server
* worker
* web-console
* sre-agent

---

## 🎯 Goal

이 저장소는 단순 배포가 아니라:

> **SRE 관점의 “운영 가능한 플랫폼”을 Git으로 정의하는 것**

---

## 🧠 Related Repository

* Portfolio: 설계 / 문서 / RAG / Runbook
* GitOps: 실제 Kubernetes 상태 관리

---

## 📌 Summary

```text
Cluster: Ready
Istio: Routing OK
Cloudflare: External Access OK
ArgoCD: GitOps OK
Monitoring: Running
Kiali: Running

→ Production-grade SRE Platform Base 완료
```

---
