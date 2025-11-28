# 🧱 Operator Framework 架構與 OLM 安裝流程（完整 Markdown 檔案）

---

## 一、Operator Framework 架構概觀

Operator Framework 是 Red Hat 與社群共同維護的一套開源工具集，  
目的是讓開發者能以一致方式 **開發、封裝、發佈、安裝與管理 Operator**。  

整體結構如下：

```
┌────────────────────────────────────┐
│          Operator Framework        │
│────────────────────────────────────│
│                                    │
│  ┌──────────────┐   ┌──────────────┐
│  │ Operator SDK │→  │ Operator Image│
│  │  建立與打包   │   │ (bundle.csv) │
│  └──────────────┘   └──────┬───────┘
│                            │
│                            ▼
│                 ┌─────────────────────┐
│                 │ Operator Registry   │
│                 │ (Catalog Image)     │
│                 │ 存放 CSV、CRD 資訊   │
│                 └─────────┬──────────┘
│                           │
│                           ▼
│              ┌────────────────────────────┐
│              │ Operator Lifecycle Manager │
│              │ 控制安裝、升級、授權、依賴   │
│              └────────────┬──────────────┘
│                           │
│                           ▼
│                 ┌───────────────────────┐
│                 │     OperatorHub       │
│                 │ Web UI → 安裝/查詢     │
│                 └───────────────────────┘
└────────────────────────────────────┘
```

---

### 模組說明

| 元件 | 功能說明 |
|------|-----------|
| **Operator SDK** | 提供開發框架與工具（支援 Go、Ansible、Helm），用來撰寫 Operator 程式與打包成 Bundle Image。 |
| **Operator Registry** | 將多個 Bundle Image 彙整為 Catalog Image，並提供給叢集中的 OLM 查詢使用。 |
| **Operator Lifecycle Manager（OLM）** | 核心控制器，負責 Operator 的安裝、升級、授權與依賴管理。 |
| **OperatorHub** | Web 介面，讓使用者在 OCP Console 上可視化地搜尋、安裝 Operator。 |

---

## 二、OpenShift Operator 安裝流程（實務邏輯）

核心概念：  
OpenShift 不直接幫你安裝 App。  
它透過 **Operator（小幫手）** 去幫你自動安裝、監控與維護 App。  

整條鏈路是：

> 你（管理員） → OLM（控制層） → Operator（小幫手） → App（最終應用）

---

### ① CatalogSource（清單來源）

**作用**  
- 提供 OLM 一個查詢來源，讓它知道有哪些可用的 Operator、版本與描述資訊。  
- CatalogSource 指向一個「Catalog Image」（index image），OCP 會在 `openshift-marketplace` 啟動一個 Pod 提供 gRPC 服務。

**實務設定**
```
harbor.com/darren-lab/olm-redhat-operator-index:v4.12
```

**結果**  
- 系統啟動一個 Pod，OLM 可透過它查詢 Operator 清單與版本。  

**一句話**
> CatalogSource = Operator 的「App 商店伺服器」。

---

### ② Subscription（下單）

**作用**  
- 告訴 OLM：「我要哪個 Operator、從哪個 Catalog 安裝、要不要自動更新。」

**內容包含**
- Operator 名稱（name）  
- 來源 Catalog（source、sourceNamespace）  
- 頻道（channel）  
- 安裝批准模式（installPlanApproval: Automatic / Manual）

**範例**
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: vertical-pod-autoscaler
  namespace: vertical-pod-autoscaler
spec:
  channel: stable
  name: vertical-pod-autoscaler
  source: olm-redhat-operator-index
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

**一句話**
> Subscription = 你按下「我要裝這個 Operator」的訂單。

---

### ③ InstallPlan（施工藍圖）

**作用**
- 當 Subscription 被建立後，OLM 會自動產生一個 InstallPlan。  
- InstallPlan 是一份「安裝藍圖」，列出要建立的所有物件（CRD、RBAC、Deployment、CSV）。  

**自動與手動差異**
- 若 Subscription 設為 `Automatic` → InstallPlan 會自動批准並執行。  
- 若設為 `Manual` → 需要手動 Approve。  

**查詢方式**
```bash
oc get installplan -n <namespace>
oc describe installplan <installplan-name> -n <namespace>
```

**範例輸出**
```
Name:             install-vpa-abcde
Namespace:        vertical-pod-autoscaler
Approval:         Automatic
Phase:            Complete
CSV:              verticalpodautoscaler.4.12.x
```

**一句話**
> InstallPlan = OLM 自動產生的安裝腳本，決定要裝什麼、怎麼裝。

---

### ④ ClusterServiceVersion（CSV，說明書）

**作用**
- 定義此版本 Operator 的全部內容，包括：
  - 要啟動哪些 Deployment（Pod）
  - 要建立哪些 CRD（功能）
  - 要哪些 RBAC 權限
  - Operator 的版本資訊與描述

**CSV 狀態轉換**
- `Pending` → `Installing` → `Succeeded`  
  當 CSV 變成 `Succeeded`，代表 Operator 已成功部署並啟動。

**一句話**
> CSV = Operator 的「版本說明書」，OLM 照它啟動服務。

---

### ⑤ Operator Ready（可用）

**結果**
- 當 CSV 為 `Succeeded`，Operator 已啟動並運作。  
- 系統中會新增新的 CRD，可建立對應 CR 物件使用功能。

**例子**
```
oc get VerticalPodAutoscaler
```

**一句話**
> Operator Ready = 小幫手正式上班，開始幫你管理應用。

---

## 三、生活比喻對照表

| 階段 | 比喻 | 代表意義 |
|------|------|-----------|
| CatalogSource | App 商店清單 | 告訴系統有哪些 Operator 可安裝 |
| Subscription | 你在商店按下「安裝」 | 指定要安裝哪個 Operator |
| InstallPlan | 背景安裝流程 | 列出並執行安裝步驟 |
| CSV | 安裝說明書 | Operator 的版本與運作規格 |
| Operator Ready | App 可使用 | Operator 運作中，可建立 CR 使用功能 |

---

## 四、完整流程總結

1. **CatalogSource**：提供 OLM 可查詢的 Operator 清單來源。  
2. **Subscription**：使用者選擇要安裝的 Operator，並設定更新方式。  
3. **InstallPlan**：OLM 根據 Subscription 自動產生的安裝計畫。  
4. **CSV**：描述 Operator 的版本與行為，OLM 依此啟動服務。  
5. **Operator Ready**：Operator 啟動完成，可提供 CRD 功能。  

---

## 五、10 秒記憶口訣

> **清單 → 下單 → 施工 → 說明書 → 可用**  
> （CatalogSource → Subscription → InstallPlan → CSV → Ready）

---

## 六、重點補充（技術層面）

| 名稱 | 建立者 | Namespace | 是否手動 | 功能 |
|------|---------|------------|-----------|------|
| CatalogSource | 管理員 | openshift-marketplace | 手動建立 | 提供 Catalog Image 來源 |
| Subscription | 管理員 | 應用 Namespace | 手動建立 | 指定要安裝哪個 Operator |
| InstallPlan | OLM 自動產生 | 同 Subscription | 自動或手動批准 | 實際執行安裝任務 |
| CSV | OLM 自動產生 | 同 Subscription | 自動 | Operator 的版本控制與說明 |
| Operator Pod | OLM 建立 | 同 Namespace | 自動 | 真正執行 Operator 程式 |

---

## 七、完整流程圖（邏輯關係）

```
CatalogSource (index image)
        │
        ▼
Subscription (你下單)
        │
        ▼
InstallPlan (OLM 自動產生)
        │
        ▼
CSV (說明書)
        │
        ▼
Operator Pod 啟動
        │
        ▼
App / CR 可用
```

---

**總結一句話：**  
> 你只需要手動建立 CatalogSource 與 Subscription，
>其餘 InstallPlan、CSV、Operator Pod 皆由 OLM 自動接手完成安裝與啟動。

---

# OpenShift Operator 核心概念摘要（OLM 最精準版）

## Subscription（Sub）
代表 OLM 要安裝哪一個 Operator、使用哪個 Channel、從哪個 Catalog 取得。

查詢所有 Subscription：
```bash
oc get sub -A
```

---

## ClusterServiceVersion（CSV）
代表 Operator 已安裝的版本與其 Metadata（CRD、RBAC、Deployment 規格等）。

查詢 CSV（預設 namespace）：
```bash
oc get csv
```

查詢所有 namespace（包含 OLM 投影）：
```bash
oc get csv -A
```

---

## Catalog（CatalogSource / Index Image）
Operator 的版本清單來源，提供 Channel、Bundle 與 Metadata，不包含實際映像檔。

查詢 CatalogSource：
```bash
oc get catalogsource -n openshift-marketplace
```

---

## Harbor（鏡像來源）
OLM 依 ICSP（ImageContentSourcePolicy）或 IDMS 設定，從 Harbor 擷取 Operator Bundle 映像檔。

查詢 ICSP：
```bash
oc get imagecontentsourcepolicy
```

---

## 為什麼 `oc get csv -A` 會很多？
因為 OLM 會將 Operator 的 CSV Metadata 投影至所有 Namespace，
此為顯示用途，不代表每個 Namespace 都重新安裝 Operator。

---

## 一句話總結（背起來）
Sub 決定要安裝什麼 → Catalog 提供版本 → CSV 表示實際已裝版本 →
OLM 依 ICSP 從 Harbor 抓映像 → `csv -A` 多為投影不是重複安裝。
