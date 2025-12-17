# Loki 與 PDB 升級衝突處理與維運手冊

> 本文件**僅整理與歸納使用者實際提供的文獻、指令輸出與實測行為**，不引入任何未出現在對話中的假設或外部推論。

---

## 1. 問題背景（你實際遇到的狀況）

在 `openshift-logging` namespace 中，LokiStack 各元件存在以下狀態：

- 多數 Loki 元件 **只有 1 個 Pod 副本**
- 對應的 **PodDisruptionBudget (PDB)** 全部設定：
  - `minAvailable: 1`
  - `ALLOWED DISRUPTIONS: 0`

代表結果：

> **任何 eviction（包含 node drain、升級過程的自願驅逐）都會被 PDB 阻擋**。

這正是你在升級 / 節點維護時，節點卡住、流程無法繼續的直接原因。

---

## 2. PDB 三個關鍵欄位的「實際意思」（口語化）

### 2.1 minAvailable

- 定義：**至少必須保持多少個「健康 Pod」**
- 你看到的設定：

```yaml
spec:
  minAvailable: 1
```

白話翻譯：

> 「不管發生什麼事，至少要有 1 個 Pod 活著。」

當副本數 = 1 時，這個設定代表：

> **那唯一的一個 Pod 絕對不能被趕走**。

---

### 2.2 maxUnavailable

- 定義：**最多允許多少個 Pod 不可用**
- 你目前的 Loki PDB **沒有使用這個欄位**，因此顯示 `N/A`
- PDB 設計上：`minAvailable` 與 `maxUnavailable` **只能擇一**

---

### 2.3 ALLOWED DISRUPTIONS

- 這不是你設定的值
- 是 Kubernetes 根據「目前 Pod 狀態 + PDB spec」**即時計算的結果**

你實測與驗證：

```bash
oc get pdb lokistack-distributor -n openshift-logging \
  -o jsonpath='{.status.disruptionsAllowed}'
```

當值為 `0` 時，意義非常明確：

> **現在不允許任何 eviction 發生**。

---

## 3. 為什麼 Loki 會「全部卡死」

條件組合如下：

- Pod 副本數 = 1
- PDB `minAvailable = 1`

推論結果（你已實際驗證）：

```
ALLOWED DISRUPTIONS = currentHealthy - minAvailable
                      = 1 - 1
                      = 0
```

因此：

> **只要 drain 節點時碰到這顆 Pod，就一定會被擋。**

---

## 4. 你實際觀察到的「PDB 行為變化」（關鍵證據）

你使用以下方式進行即時監控：

```bash
oc get pdb lokistack-distributor -n openshift-logging -w \
  -o jsonpath='{.metadata.resourceVersion}\t{.spec.minAvailable}\t{.status.disruptionsAllowed}\n'
```

觀察到的實際輸出順序（重點）：

```
42818601   1   0
42823866   0   0
42823869   0   1
42823878   1   1
42823879   1   0
```

這證明了以下事實：

1. `minAvailable` 被 patch 成 `0` 後
2. `disruptionsAllowed` **確實會短暫變成 > 0**
3. 後續又被改回 `minAvailable = 1`

> **PDB 不是靜態的，它會被系統重新調整。**

---

## 5. 為什麼「patch minAvailable = 0」不是永久解法

### 5.1 你已實測到的事實

- `oc patch pdb ... minAvailable=0` **可以生效**
- 但之後 `resourceVersion` 再次變動
- `minAvailable` 被改回 `1`

這代表：

> **PDB 受 Operator / 控制器 reconcile 行為影響**。

---

### 5.2 結論（只根據你提供的證據）

- `minAvailable=0` **只能當作「維護 / 升級期間的暫時手段」**
- 不能當成永久設定依賴

---

## 6. 正確的實務處理策略（依目的區分）

### 6.1 目的一：讓升級 / drain 不要卡住（短期）

**你實際可行、且已驗證的方法：**

```bash
oc patch pdb lokistack-distributor      -n openshift-logging --type=merge -p '{"spec":{"minAvailable":0}}'
oc patch pdb lokistack-index-gateway    -n openshift-logging --type=merge -p '{"spec":{"minAvailable":0}}'
oc patch pdb lokistack-ingester         -n openshift-logging --type=merge -p '{"spec":{"minAvailable":0}}'
oc patch pdb lokistack-querier          -n openshift-logging --type=merge -p '{"spec":{"minAvailable":0}}'
oc patch pdb lokistack-query-frontend   -n openshift-logging --type=merge -p '{"spec":{"minAvailable":0}}'
```

用途：

> **只為了讓 eviction 能走得動，完成升級或節點維護。**

---

### 6.2 升級完成後（必要復原）

```bash
oc patch pdb lokistack-distributor      -n openshift-logging --type=merge -p '{"spec":{"minAvailable":1}}'
oc patch pdb lokistack-index-gateway    -n openshift-logging --type=merge -p '{"spec":{"minAvailable":1}}'
oc patch pdb lokistack-ingester         -n openshift-logging --type=merge -p '{"spec":{"minAvailable":1}}'
oc patch pdb lokistack-querier          -n openshift-logging --type=merge -p '{"spec":{"minAvailable":1}}'
oc patch pdb lokistack-query-frontend   -n openshift-logging --type=merge -p '{"spec":{"minAvailable":1}}'
```

原因：

> **否則 Loki 在日常運作時將完全失去可用性保護。**

---

## 7. 快速檢查「哪些 PDB 正在擋你」的標準指令

### 7.1 列出 disruptionsAllowed = 0（你已驗證可用）

```bash
oc get pdb -A -o jsonpath='{range .items[?(@.status.disruptionsAllowed==0)]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}'
```

### 7.2 含表頭版本（你實際需要的格式）

```bash
{
  printf "NAMESPACE\tNAME\tALLOWED_DISRUPTIONS\n"
  oc get pdb -A -o jsonpath='{range .items[?(@.status.disruptionsAllowed==0)]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.disruptionsAllowed}{"\n"}{end}'
} | column -t
```

---

## 8. 關於「直接改 Deployment 副本數」為什麼不可靠

你已實測：

```bash
oc patch deploy lokistack-distributor -n openshift-logging -p '{"spec":{"replicas":2}}'
```

結果：

- Replica 短暫變成 2
- 隨即被改回 1

結論（根據你的實測）：

> **LokiStack 屬於 Operator 管理資源，Deployment 不是最終真實來源。**

---

## 9. 若要「真正長期解決」的方向（不給未驗證命令）

唯一合理方向：

> **從 LokiStack CR 本身調整 size / 架構，讓元件天生具備多副本。**

但：

- 你未提供完整 `LokiStack` CR YAML
- 因此此文件 **不給任何推測性設定值**

---

## 10. 一句話總結（完全對應你的實測）

> **Loki 不是不能升級，是 PDB + 單副本設計讓 eviction 無法發生。**  
> **patch minAvailable=0 是維護開關，不是永久設定。**

---
（本文件結束）

