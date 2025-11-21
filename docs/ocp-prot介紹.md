# OpenShift 安裝與運作期間所有關鍵 Port 整理

本文件整理 OpenShift Container Platform（OCP）在 **Bootstrap → Master → Worker** 完整流程中所有必備且最重要的 Port。  
包含作用、來源角色、用途與是否會被接管（bootstrap → master）。

---

## 1. Bootstrap / Master 安裝期間最重要的 Port（四大核心）

| Port   | 協定 | 來源角色 | 用途說明 |
|--------|------|----------|----------|
| **6443** | TCP | Bootstrap → Master | Kubernetes API Server。節點首次註冊、控制平面初始化、kubectl 全部靠這個入口。 |
| **22623** | TCP | Bootstrap → Master | MachineConfig / Ignition 配置服務（Machine Config Server）。所有節點第一次開機都從此抓設定。 |
| **2379** | TCP | Bootstrap → Master | etcd Client API。控制平面用來存取叢集狀態的主要入口。 |
| **2380** | TCP | Bootstrap → Master | etcd Peer。etcd 節點之間同步資料的管道。 |

> 這四個 port 是 OCP 出生的「生命線」，缺一不可。

---

## 2. Master（Control Plane）正常運作必備 Ports

| Port   | 協定 | 用途說明 |
|--------|------|----------|
| **10250** | TCP | kubelet API。API server 對各節點 kubelet 的健康檢查與執行命令。 |
| **10257** | TCP | kube-controller-manager API（HTTPS）。 |
| **10259** | TCP | kube-scheduler API（HTTPS）。 |

---

## 3. Worker 節點加入所需 Ports

| Port   | 協定 | 用途說明 |
|--------|------|----------|
| **6443** | TCP | Worker 的 kubelet 透過此 port 註冊至 API server。 |
| **22623** | TCP | Worker 第一次開機抓 MachineConfig / Ignition 設定。 |

> Worker 加入叢集只需要 **6443 + 22623**，不需要 etcd ports。

---

## 4. Ingress / Router 運作相關 Ports（非安裝必備，但叢集運作必需）

| Port   | 協定 | 用途說明 |
|--------|------|----------|
| **80** | TCP | HTTP Ingress 流量入口（HAProxy Router）。 |
| **443** | TCP | HTTPS Ingress 流量入口。 |
| **1936** | TCP | HAProxy Router stats 介面（啟用後監控用）。 |

---

## 5. Port 接管流程（bootstrap → master）圖示

      ┌────────────────────────┐
      │       Bootstrap        │
      │------------------------│
      │ 6443 → 臨時 API        │
      │ 22623 → 臨時 MCS       │
      │ 2379 → 臨時 etcd client│
      │ 2380 → 臨時 etcd peer  │
      └───────────┬────────────┘
                  │  Master 起來後接手
                  ▼
      ┌────────────────────────┐
      │         Master         │
      │------------------------│
      │ 6443 → 正式 API        │
      │ 22623 → 正式 MCS       │
      │ 2379 → 正式 etcd client│
      │ 2380 → 正式 etcd peer  │
      └────────────────────────┘
6443 → Kubernetes API Server
22623 → MachineConfig / Ignition Server
2379 → etcd client API
2380 → etcd peer sync