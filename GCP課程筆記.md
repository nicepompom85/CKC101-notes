# GCP 雲端服務實作課程筆記

> **Google Cloud Platform — GCE + Docker + Startup Script + Metadata**
> CKC101 網路與雲端系統管理課程 · pom (racing18028@gmail.com)

---

## 📋 課程概覽

| 項目 | 說明 |
|------|------|
| 平台 | Google Cloud Platform (GCP) |
| 核心服務 | GCE、Cloud Billing、IAM、Metadata Server |
| 技術工具 | Docker、Bash Startup Script |
| 課程目標 | 從手動安裝 Docker 到開機自動化部署容器 |

---

## Step 1：啟用 GCP 服務與設定預算

### 1-1 啟用 GCP 服務

- 登入 Google Cloud Console：https://console.cloud.google.com
- 建立或選取專案（Project）
- 在「API 和服務」頁面中啟用所需的 API（如 Compute Engine API）

### 1-2 設置預算（Budget）

- 前往「帳單」→「預算與快訊」
- 點選「建立預算」，設定預算名稱
- 設定預算金額與提醒百分比（例如：50%、90%、100%）
- 系統會在達到門檻時寄送 Email 通知

> 💡 **提示**：設定預算可有效避免超額費用，建議每個專案都設置。

---

## Step 2：開啟專案檢視權限給同學

### 2-1 設定 IAM 權限

- 前往 GCP Console → 「IAM 與管理員」→「IAM」
- 點選「新增成員（Add Principal）」
- 輸入同學的 Google 帳號（Email）
- 選擇適當角色（Role），例如：
  - **Viewer**（檢視者）：唯讀，只能查看資源
  - **Editor**（編輯者）：可修改資源但不能管理權限
- 點選「儲存」完成授權

---

## Step 3：用 GCE 建立虛擬機器

### 3-1 建立 VM 執行個體

1. 前往 Compute Engine → VM 執行個體
2. 點選「建立執行個體」
3. 設定 VM 基本參數：

| 參數 | 建議設定 |
|------|---------|
| 名稱 | 自訂（如 docker-server） |
| 區域 / 地區 | asia-east1 / asia-east1-b（台灣） |
| 機器類型 | e2-micro（免費方案可用） |
| 作業系統 | Ubuntu 22.04 LTS |
| 開機磁碟 | 10 GB（Standard） |
| 防火牆 | 允許 HTTP 流量 / 允許 HTTPS 流量 |

4. 點選「建立」，等待 VM 啟動（狀態顯示綠色勾勾）
5. 點選 VM 列表中的「SSH」按鈕即可遠端連線

---

## Step 4：在 VM 上安裝 Docker 並執行容器

### 4-1 手動安裝 Docker

SSH 連線進入 VM 後，執行以下指令：

```bash
# 下載並執行 Docker 官方安裝腳本
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# 啟動 Docker 服務
sudo systemctl start docker

# 驗證安裝成功
sudo docker version
```

### 4-2 拉取並執行之前製作的 Docker Image

```bash
# 從 Docker Hub 拉取之前推送的 Image
sudo docker pull pompom85/<image-name>:<tag>

# 啟動容器（背景執行，對應 Port）
sudo docker run -d -p 80:5000 pompom85/<image-name>:<tag>

# 查看執行中的容器
sudo docker ps
```

開啟瀏覽器輸入 VM 的外部 IP 即可驗證服務是否正常運行。

---

## Step 5：使用啟動腳本（Startup Script）自動化部署

將安裝流程改寫為 Startup Script，讓 VM 每次開機時自動完成 Docker 安裝與容器啟動，無需人工介入。

### 5-1 Startup Script 設定位置

- 建立 VM 時 → 「進階選項」→「管理」→「自動化」→ 貼上腳本
- 或在 VM 詳細頁面 → 「編輯」→「自訂中繼資料」→ Key：`startup-script`

### 5-2 Startup Script 完整內容

```bash
#!/bin/bash
#Write by pom (racing18028@gmail.com)

# 腳本名稱: startup-script

echo "等待 apt-get 鎖定被釋放..."

# 檢查 dpkg 和 apt 的鎖定檔案，如果發現被佔用則等待 10 秒後重試
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 || \
      fuser /var/lib/apt/lists/lock >/dev/null 2>&1 ; do
  echo "偵測到 apt-get 正在運行，等待 10 秒..."
  sleep 10
done

echo "apt-get 鎖定已釋放，繼續執行安裝..."

# 1. 安裝 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# 2. 啟動 Docker 服務
echo "啟動 Docker 服務..."
service docker start

# 3. 從 Metadata 讀取自訂變數
METADATA_DEMO=$(curl "http://metadata.google.internal/computeMetadata/v1/instance/attributes/METADATA_DEMO" \
  -H "Metadata-Flavor: Google")

# 4. 建立測試檔案（驗證 Metadata 是否取得成功）
touch /tmp/$METADATA_DEMO

# 5. 啟動 Docker 容器
docker run -d -p 80:5000 -e PORT=5000 \
  -e METADATA_DEMO=$METADATA_DEMO \
  b97607065/gcp_metadata_demo:0.0.3
```

### 5-3 Startup Script 重要說明

| 機制 | 說明 |
|------|------|
| apt 鎖定等待 | VM 開機時系統自動更新可能佔用 apt，必須等待釋放才能安裝 Docker |
| service vs systemctl | Startup Script 執行環境中 systemctl 可能失敗，改用 `service docker start` 更穩定 |
| Metadata 讀取 | 透過 GCP 內部 Metadata Server HTTP API 讀取自訂變數 |
| touch 驗證 | 在 /tmp 建立以 Metadata 值命名的檔案，用於驗證取得成功 |

---

## Step 6：透過 Metadata Server 查詢 VM 資訊

GCP Metadata Server 是每個 VM 內部可存取的特殊端點，提供執行個體的各種資訊（IP、區域、自訂屬性等），**不需要 API Key**。

### 6-1 Metadata Server 基本資訊

| 屬性 | 值 |
|------|---|
| Metadata Server 位址 | `http://metadata.google.internal` |
| 必要 Header | `Metadata-Flavor: Google` |
| 存取方式 | 僅限 VM 內部存取（不可從外部呼叫） |
| 版本路徑 | `/computeMetadata/v1/` |

### 6-2 常用 Metadata 查詢指令

```bash
# 查詢 VM 的外部 IP（公有 IP）
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/externalIp

# 查詢 VM 的內部 IP（私有 IP）
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip

# 查詢 VM 名稱
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/name

# 查詢自訂 Metadata 屬性（METADATA_DEMO）
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/METADATA_DEMO
```

### 6-3 設定自訂 Metadata

- GCP Console → Compute Engine → 點選 VM 名稱 → 「編輯」
- 在「自訂中繼資料」區域，點選「新增項目」
- 輸入 Key（如 `METADATA_DEMO`）與 Value（如 `hello-pom`）
- 儲存後，VM 內即可用 curl 讀取此值

---

## 📌 課程重點整理

| 步驟 | 核心要點 |
|------|---------|
| 1. 啟用服務 & 預算 | 先設定預算避免超額，再啟用 Compute Engine API |
| 2. IAM 權限 | Viewer 角色只能查看，不影響資源安全 |
| 3. 建立 GCE VM | 區域選台灣（asia-east1），防火牆允許 HTTP |
| 4. Docker 部署 | curl 官方腳本安裝，docker run 啟動容器 |
| 5. Startup Script | 開機自動化：等待 apt → 裝 Docker → 讀 Metadata → 起容器 |
| 6. Metadata Server | 內部 API，需加 `Metadata-Flavor: Google` Header 才能存取 |

---

## ⚠️ 常見問題與解決方式

### Q1：Startup Script 中 Docker 安裝失敗？
- **原因**：開機時 apt 被系統自動更新佔用
- **解決**：加入 `while fuser` 迴圈等待 apt 鎖定釋放

### Q2：Startup Script 中 `systemctl start docker` 失敗？
- **原因**：非 interactive shell 環境中 systemd 可能未完全初始化
- **解決**：改用 `service docker start`

### Q3：curl Metadata 回傳 403 Forbidden？
- **原因**：缺少必要的 Header
- **解決**：加上 `-H "Metadata-Flavor: Google"`

### Q4：瀏覽器無法存取 VM 外部 IP？
- **原因**：GCE 防火牆規則未開放 HTTP（Port 80）
- **解決**：VM 設定中勾選「允許 HTTP 流量」或在 VPC 防火牆新增規則

---

*CKC101 網路與雲端系統管理課程 · pom (racing18028@gmail.com)*
