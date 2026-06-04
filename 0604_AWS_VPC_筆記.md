# AWS VPC 實作筆記 — 2026/06/04

> 課程目標：在 AWS 建立 VPC，設定公有與私有子網路，部署 EC2 執行個體，並透過 SSH 從外部連線，實作 NAT Gateway 及 VPC Peering。

---

## 目錄

1. [環境架構總覽](#1-環境架構總覽)
2. [建立 EC2 執行個體](#2-建立-ec2-執行個體)
3. [MobaXterm SSH Jump Host 連線](#3-mobaxterm-ssh-jump-host-連線)
4. [NAT Gateway](#4-nat-gateway)
5. [在 VPC 建立 NAT Gateway 實作](#5-在-vpc-建立-nat-gateway-實作)
6. [VPC Peering 跨帳號連線](#6-vpc-peering-跨帳號連線)

---

## 1. 環境架構總覽

本次課程使用一個自訂 VPC，內含一個公有子網路和一個私有子網路，分別部署一台 EC2。

| 資源 | 名稱 / ID | CIDR / IP |
|------|-----------|-----------|
| VPC | ckc023 (`vpc-04b8336f95c936dc7`) | `10.23.0.0/16` |
| 公有子網路 | public-1 (`subnet-0ff3a95da8550d306`) | `10.23.16.0/24` |
| 私有子網路 | private-1 (`subnet-021b9c70353cbb30c`) | `10.23.10.0/24` |
| public-server | `i-0d281272ee4960b0f` | 公有 IP：`43.212.240.1` / 私有 IP：`10.23.16.37` |
| private-server | `i-04bb0d37078ebe56b` | 公有 IP：`43.212.253.112`（無法直接連） / 私有 IP：`10.23.10.200` |

> **架構說明：** public-server 放在公有子網路，透過 Internet Gateway 對外提供 SSH 入口，同時作為跳板機（Jump Host）連接 private-server。private-server 放在私有子網路，無法從外部直接連入。

---

## 2. 建立 EC2 執行個體

### 2-1. 建立 public-server

#### ① 名稱與 AMI

| 設定 | 值 |
|------|-----|
| 名稱 | `public-server` |
| AMI | Ubuntu Server 26.04 LTS (HVM), SSD Volume Type |
| 架構 | 64 位元 (x86) |
| 執行個體類型 | t3.micro（符合免費方案資格） |

#### ② 網路設定

| 設定 | 值 |
|------|-----|
| VPC | `vpc-04b8336f95c936dc7 (ckc023)`，`10.23.0.0/16` |
| 子網路 | `subnet-0ff3a95da8550d306 (public-1)`，`10.23.16.0/24` |
| 自動指派公有 IP | **啟用** ← 重要，public-server 必須開啟 |

#### ③ 安全群組（防火牆）

選擇「建立安全群組」，新增以下 Inbound Rule：

| 類型 | 通訊協定 | 連接埠 | 來源類型 | 說明 |
|------|----------|--------|----------|------|
| SSH | TCP | 22 | 隨處（`0.0.0.0/0`） | 允許外部 SSH 連入 |

> **安全建議：** 正式環境建議將來源改為「My IP」，只允許自己的 IP 連入，避免暴露在公開網路。

#### ④ 設定儲存

| 設定 | 值 |
|------|-----|
| 容量 | 20 GiB |
| 磁碟類型 | gp3（3000 IOPS，未加密） |
| 檔案系統 | 無 |

---

### 2-2. 建立 private-server

#### ① 名稱與 AMI

| 設定 | 值 |
|------|-----|
| 名稱 | `private-server` |
| AMI | Ubuntu Server 26.04 LTS (HVM), SSD Volume Type |
| 架構 | 64 位元 (x86) |
| 執行個體類型 | t3.micro（符合免費方案資格） |

#### ② 網路設定

| 設定 | 值 |
|------|-----|
| VPC | `vpc-04b8336f95c936dc7 (ckc023)`，`10.23.0.0/16` |
| 子網路 | `subnet-021b9c70353cbb30c (private-1)`，`10.23.10.0/24` |
| 自動指派公有 IP | 啟用（但 private subnet 無對外路由，實際上無法從外部直接連入） |

#### ③ 安全群組（防火牆）

| 類型 | 通訊協定 | 連接埠 | 來源類型 |
|------|----------|--------|----------|
| SSH | TCP | 22 | 隨處（`0.0.0.0/0`） |

> **進階設定：** 來源可以改成 `10.23.16.0/24`（只允許來自 public subnet 的流量），安全性更高，符合最小權限原則。

#### ④ 設定儲存

| 設定 | 值 |
|------|-----|
| 容量 | 20 GiB |
| 磁碟類型 | gp3 |
| 檔案系統 | 無 |

---

### 2-3. 兩台共同設定

- **Key Pair**：兩台都選同一把 `.pem` 私鑰檔案，這樣 MobaXterm Jump Host 才能用同一把鑰匙認證兩台機器。
- **執行個體類型**：t3.micro，符合 AWS 免費方案（每月 750 小時）。

---

### 2-4. 建立完成後確認清單

| 項目 | public-server | private-server |
|------|--------------|----------------|
| 執行狀態 | 執行中 ✅ | 執行中 ✅ |
| 狀態檢查 | 3/3 通過 ✅ | 3/3 通過 ✅ |
| 公有 IPv4 | `43.212.240.1` | `43.212.253.112`（無法直接連） |
| 私有 IPv4 | `10.23.16.37` | `10.23.10.200` |
| 所在 Subnet | public-1 | private-1 |
| 可用區域 | ap-east-2a | ap-east-2a |

---

## 3. MobaXterm SSH Jump Host 連線

private-server 沒有公網入口，必須透過 public-server 當**跳板機（Jump Host）**才能連入。

### 3-1. 前提

- 兩台 EC2 使用**同一把 Key Pair**（`.pem` 私鑰）
- private-server 的 Security Group 需允許來自 public-server（`10.23.16.37` 或整個 `10.23.0.0/16`）的 Port 22

---

### 3-2. MobaXterm 設定步驟

#### Step 1：設定連到 public-server 的 Session

`Session → SSH`

| 欄位 | 填入值 |
|------|--------|
| Remote host | `43.212.240.1` |
| Username | `ubuntu` |
| Port | `22` |

點 **Advanced SSH settings** 分頁：
- ☑ 勾選 **Use private key**
- 選擇 `.pem` 檔案路徑（例如 `C:\Users\TMP-214\Desktop\private.pem`）

---

#### Step 2：設定連到 private-server 的 Session（透過跳板）

`Session → SSH`

| 欄位 | 填入值 |
|------|--------|
| Remote host | `10.23.10.200` |
| Username | `ubuntu` |
| Port | `22` |

點 **Network settings** 分頁：
- 點選中間圖示，選擇 **SSH gateway (jump host)**

下方 Proxy settings 填入：

| 欄位 | 填入值 |
|------|--------|
| Proxy type | `SSH command` |
| Host | `43.212.240.1` |
| Login | `ubuntu` |
| Port | `22` |
| SSH proxy command | `nc %host %port \|\| socat - TCP:%host:%port` |

---

#### Step 3：設定 Jump Host 金鑰

點擊 **SSH gateway (jump host)** 圖示後，跳出 **MobaXterm jump hosts configuration** 視窗：

| 欄位 | 填入值 |
|------|--------|
| Gateway host | `43.212.240.1` |
| Username | `ubuntu` |
| Port | `22` |
| ☑ Use SSH key | 選擇同一把 `.pem` 檔案 |

填完按 **OK**。

---

### 3-3. 指令行等效方式（參考）

```bash
# 先設定私鑰權限（Mac/Linux 必做）
chmod 400 my-key.pem

# 透過 Jump Host 連線到 private-server
ssh -i my-key.pem \
    -J ubuntu@43.212.240.1 \
    ubuntu@10.23.10.200
```

> `-J` 參數代表 Jump Host，SSH 會先連到跳板機，再從跳板機連到目標機器。

---

### 3-4. 常見 SSH 預設使用者名稱

| AMI | 預設使用者名稱 |
|-----|--------------|
| Amazon Linux 2 / 2023 | `ec2-user` |
| Ubuntu | `ubuntu` |
| CentOS | `centos` |
| Debian | `admin` |

---

## 4. NAT Gateway

### 4-1. 一句話解釋

> 私有子網路的機器想要**主動連出**網際網路，但不想被外部直接連入，就靠 NAT Gateway 幫它轉發流量。

### 4-2. 生活比喻

想像你住在一棟大樓（VPC）裡，你的房間（private subnet）沒有對外窗，但你可以委託大廳服務台（NAT Gateway）幫你寄信出去。外面的人不知道你的房間號碼，只看到服務台的地址。

---

### 4-3. NAT 是什麼

**NAT = Network Address Translation（網路地址轉換）**

讓私有子網路的機器可以主動連出網際網路（例如 `apt update`、下載套件），但外部無法主動連進來。

---

### 4-4. Internet Gateway vs NAT Gateway

| 項目 | Internet Gateway (IGW) | NAT Gateway |
|------|----------------------|-------------|
| 放在哪 | 附加在整個 VPC 上 | 必須放在 **Public Subnet** 內 |
| 用途 | 讓 Public Subnet 雙向通網路 | 讓 Private Subnet 單向出網路 |
| 流量方向 | 雙向（可進可出） | 單向（只能出，外部無法主動進入） |
| 需要公有 IP | 否（IGW 本身不需要） | 是（NAT GW 本身需要 Elastic IP） |
| 費用 | **免費** | **收費**（按小時 + 流量計算） |

---

### 4-5. 運作流程

```
private-server (10.23.10.200)
    │
    ↓ ① 送出請求（來源 IP = 10.23.10.200）
    │
NAT Gateway（位於 Public Subnet，Elastic IP = 43.x.x.x）
    │
    ↓ ② 把來源 IP 替換成自己的 Elastic IP
    │
Internet Gateway
    │
    ↓ ③ 到達網際網路
    │
外部網站（只看到 NAT GW 的 Elastic IP，完全看不到私網 IP）
```

---

## 5. 在 VPC 建立 NAT Gateway 實作

### Step 1：建立 NAT Gateway

`AWS Console → VPC → 左側選單 NAT Gateways → Create NAT Gateway`

| 設定 | 填入值 |
|------|--------|
| Name | `nat-gw`（自訂） |
| Subnet | `subnet-0ff3a95da8550d306 (public-1)` ← **一定要選公有子網路** |
| Connectivity type | `Public` |
| Elastic IP | 點 **Allocate Elastic IP**（自動產生一個公有 IP） |

填完按 **Create NAT gateway**，等狀態變成 **Available**（約 1～2 分鐘）。

---

### Step 2：修改 Private Subnet 的 Route Table

`VPC → Route Tables → 找 private-1 對應的 Route Table`

點進去 → **Routes** 分頁 → **Edit routes** → **Add route**

| Destination | Target |
|-------------|--------|
| `0.0.0.0/0` | NAT Gateway → 選剛建立的 `nat-gw` |

按 **Save changes**。

---

### Step 3：驗證連線

用 MobaXterm Jump Host 連進 private-server 後，執行：

```bash
# 測試能否連外網（Google DNS）
ping 8.8.8.8

# 查看對外 IP（應顯示 NAT GW 的 Elastic IP）
curl https://ifconfig.me
```

如果有回應，且顯示的 IP 是 NAT GW 的 Elastic IP，代表設定成功。

---

### ⚠️ 費用提醒

> NAT Gateway **按小時計費**，測試完畢後記得：
> 1. `VPC → NAT Gateways` → 選取 → **Delete NAT Gateway**
> 2. `EC2 → Elastic IPs` → 選取 → **Release Elastic IP address**（未關聯的 Elastic IP 也會收費）

---

## 6. VPC Peering 跨帳號連線

### 6-1. 概念說明

**VPC Peering（VPC 對等連線）** 讓兩個不同 AWS 帳號的 VPC 內部私有網段能夠互相通訊，就像在同一個內部網路一樣。

> ⚠️ **前提條件：兩個 VPC 的 CIDR 不能重疊**
> 例如帳號 A 用 `10.0.0.0/16`，帳號 B 用 `10.1.0.0/16`。

---

### 6-2. 操作步驟

#### Step 1：帳號 A 發出 Peering 請求

`登入帳號 A → VPC → 左側 Peering Connections → Create Peering Connection`

| 設定 | 值 |
|------|-----|
| Name | `peering-a-to-b` |
| VPC ID (Requester) | 選帳號 A 的 VPC |
| Account | **Another account** |
| Account ID | 填入帳號 B 的 12 位數帳號 ID |
| Region | 帳號 B 的 VPC 所在 Region |
| VPC ID (Accepter) | 填入帳號 B 的 VPC ID |

按 **Create Peering Connection**，狀態會變成 **Pending Acceptance**。

---

#### Step 2：帳號 B 接受請求

`登入帳號 B → VPC → Peering Connections`

找到狀態為 **Pending Acceptance** 的請求 → 選取 → **Actions** → **Accept Request**

狀態變成 **Active** 代表連線通道建立成功。

---

#### Step 3：帳號 A 修改 Route Table

`帳號 A → VPC → Route Tables → 找 private subnet 的 Route Table → Edit routes → Add route`

| Destination | Target |
|-------------|--------|
| `10.1.0.0/16`（帳號 B 的 VPC CIDR） | Peering Connection → 選 `pcx-xxxxxxxx` |

---

#### Step 4：帳號 B 修改 Route Table

`帳號 B → VPC → Route Tables → 找 private subnet 的 Route Table → Edit routes → Add route`

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16`（帳號 A 的 VPC CIDR） | Peering Connection → 選同一個 `pcx-xxxxxxxx` |

> 兩邊都要加路由，缺一不可。

---

#### Step 5：兩邊 Security Group 互相開放

**帳號 A 的 EC2 Security Group** 新增 Inbound Rule：

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH（或 All traffic） | TCP | 22 | `10.1.0.0/16`（帳號 B 網段） |

**帳號 B 的 EC2 Security Group** 新增 Inbound Rule：

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH（或 All traffic） | TCP | 22 | `10.0.0.0/16`（帳號 A 網段） |

---

#### Step 6：測試連線

在帳號 A 的 EC2 內執行：

```bash
# Ping 測試
ping 10.1.1.20

# SSH 連線測試
ssh ubuntu@10.1.1.20
```

有回應就代表跨帳號 VPC Peering 成功。

---

### 6-3. 重點整理

| 項目 | 說明 |
|------|------|
| 費用 | 同 Region 內 Peering 流量**免費**，跨 Region 要收費 |
| 限制 | 不支援 Transitive Routing（A→B→C 不通，只有直接對等的兩個 VPC 才通） |
| CIDR 限制 | 兩個 VPC 網段**絕對不能重疊** |
| 必做兩邊 | Route Table 和 Security Group **兩邊都要改**，缺一不可 |
| 跨帳號需要 | 對方帳號 ID 和 VPC ID |

---

## 附錄：常見 SSH 連線問題排查

| 錯誤訊息 | 可能原因 | 解決方式 |
|----------|----------|----------|
| `Connection timed out` | Security Group 沒開 Port 22，或來源 IP 限制 | 檢查 SG Inbound rules，確認 Port 22 開放 |
| `Connection refused` | SSH 服務未啟動，或 ufw 擋住 | 確認 EC2 內 `sudo systemctl status ssh` |
| `Permission denied` | 私鑰不符，或使用者名稱錯誤 | 確認 `.pem` 檔案正確，並用對應的預設使用者名稱 |
| `Route Table 沒有 IGW` | public subnet 無法連外 | VPC → Route Tables → 加入 `0.0.0.0/0 → igw-xxx` |
| EC2 停機重開後 IP 變了 | 沒有綁定 Elastic IP | 分配 Elastic IP 並關聯到執行個體 |
