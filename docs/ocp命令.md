# 🧰 OpenShift CLI 常用命令筆記

## 🟩 基本命令
| 命令 | 說明 |
|------|------|
| `oc project` | 顯示目前作用中的專案（Namespace）與連線伺服器 |
| `oc project -q` | 僅顯示目前專案名稱 |
| `oc project <project-name>` | 切換到指定專案 |
| `oc get project` | 列出叢集中所有專案（對應 Kubernetes 的 Namespace） |
| `oc whoami` | 顯示目前登入使用者 |
| `oc version` | 顯示 CLI 與伺服器版本 |
| `oc login` | 登入 OpenShift 叢集 |
| `oc logout` | 登出目前連線 |

## 🟦 檢視資源狀態
| 命令 | 說明 |
|------|------|
| `oc get pods -n <namespace>` | 列出指定 Namespace 中所有 Pod 狀態 |
| `oc get pods -n <namespace> -o wide` | 顯示完整欄位（包含 IP、節點） |
| `oc get pods -A` 或 `oc get pods --all-namespaces` | 查看所有命名空間的 Pod |
| `oc get pods -l <label>=<value>` | 以 label 篩選 Pod |
| `oc describe pod <pod-name> -n <namespace>` | 查看單一 Pod 詳細資訊（事件、容器狀態） |
| `oc get all -n <namespace>` | 查看命名空間內所有主要資源（Pod、Service、Deployment、ReplicaSet） |
| `oc describe <type> <name> -n <namespace>` | 檢視任何資源的詳細事件與設定 |
| `oc get co` | 查看叢集層級 Cluster Operators 狀態（系統核心元件） |
| `oc get csv -A` | 查看所有命名空間的 Operator（OLM 管理）版本與狀態 |
| `oc get events -A --sort-by=.metadata.creationTimestamp` | 查看所有事件（依時間排序） |
| `oc get pods -w -n <namespace>` | 監看 Pod 狀態即時變化 |

### `-A`（All Namespaces）使用說明
| 指令 | 功能 | 是否允許 |
|------|------|-----------|
| `oc get pods -A` | 查所有命名空間的所有 Pod | ✅ 允許 |
| `oc get pod <pod-name> -A` | 查單一 Pod 跨命名空間 | ❌ 不允許 |
| `oc describe pod <pod-name> -A` | 查單一 Pod 詳細資訊跨命名空間 | ❌ 不允許 |
| `oc get pod <pod-name> -n <namespace>` | 查單一命名空間內 Pod | ✅ 允許 |

📘 Pod 名稱不是全域唯一，不同 Namespace 可能存在相同 Pod 名，因此禁止以名稱跨命名空間查詢。

### 查詢 Pod 所屬 Namespace
```bash
oc get pods -A | grep <pod-name>
```

## 🔍 常用查詢指令快速表
| 操作 | 指令 |
|------|------|
| 查 Pod 詳細資訊 | `oc get pod nginx-6f8565f9fc-8hb79 -n nginx-lab -o wide` |
| 查事件與容器狀態 | `oc describe pod nginx-6f8565f9fc-8hb79 -n nginx-lab` |

## 🧠 補充說明
| 指令 | 功能 |
|------|------|
| `oc get co` | 檢查 OpenShift 系統層核心組件（如 etcd、kube-apiserver、network、machine-config） |
| `oc get csv` | 查看 Operator Lifecycle Manager（OLM）管理的應用層 Operator（如 VPA、PackageServer） |

## 🟨 日誌與容器互動
| 命令 | 說明 |
|------|------|
| `oc logs pod/<pod-name> -n <namespace>` | 顯示 Pod 預設容器日誌 |
| `oc logs pod/<pod-name> -c <container-name> -n <namespace>` | 顯示特定容器日誌（多容器 Pod 使用） |
| `oc logs -f pod/<pod-name> -n <namespace>` | 追蹤即時輸出日誌 |
| `oc exec -it pod/<pod-name> -n <namespace> -- /bin/sh` | 進入容器 Shell |
| `oc rsh pod/<pod-name> -n <namespace>` | 使用遠端 Shell 登入 Pod |
| `oc cp <namespace>/<pod-name>:<path> <local-path>` | 從 Pod 複製檔案至本地 |
| `oc cp <local-path> <namespace>/<pod-name>:<path>` | 從本地複製檔案至 Pod |

## 🟥 節點與叢集檢查
| 命令 | 說明 |
|------|------|
| `oc get nodes` | 列出所有節點 |
| `oc describe node <node-name>` | 查看節點詳情（條件、taints、容量等） |
| `oc adm top nodes` | 查看節點 CPU／記憶體使用量 |
| `oc adm top pods -n <namespace>` | 查看命名空間中 Pod 的資源使用情況 |
| `oc get clusteroperators` | 等同 `oc get co`，檢查系統核心 Operator 狀態 |
| `oc get clusterversion` | 查看整體 OpenShift 版本與升級狀態 |
| `oc get mcp` | 查看 MachineConfigPool 狀態 |
| `oc get mcd -A` | 查看 MachineConfigDaemon 狀態 |
| `oc adm node-logs --role=<master|worker> -u <systemd_unit>` | 抓取節點 systemd 單元日誌 |
| `oc adm node-logs --role=master --path=<path_under_/var/log>` | 指定路徑抓取節點日誌 |

## 🟪 進階／工具性命令
| 命令 | 說明 |
|------|------|
| `oc api-resources` | 列出叢集支援的所有 API 資源種類 |
| `oc api-versions` | 顯示叢集支援的 API 版本 |
| `oc explain <resource>` | 查看資源結構與欄位說明 |
| `oc explain deployment.spec.template.spec.containers` | 查看 Deployment 中容器設定的詳細結構 |
| `oc get pods -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName` | 自訂輸出欄位顯示 Pod 與所屬節點 |
| `oc adm release info` | 顯示目前 OpenShift 發行版本資訊 |
| `oc adm release extract --tools` | 提取 release 版本中包含的 CLI 工具 |

## 🧩 常見操作整合
| 操作目的 | 對應指令 |
|-----------|-----------|
| 查看系統核心元件狀態 | `oc get co` |
| 查看 Operator 狀態 | `oc get csv -A` |
| 查看節點使用率 | `oc adm top nodes` |
| 檢查升級狀態 | `oc get clusterversion` |
| 查看節點日誌 | `oc adm node-logs --role=master -u crio` |
| 追蹤 Pod 啟動日誌 | `oc logs -f pod/<pod-name> -n <namespace>` |
