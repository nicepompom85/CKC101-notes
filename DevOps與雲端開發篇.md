# DevOps 與雲端開發篇

> AWS IAM / S3 / EC2 · Docker · Git 完整實驗筆記
>
> 日期：2026-05-28

---

## 目錄

- [實驗概述](#-實驗概述)
- [PART 1：AWS IAM — 開發人員身份與權限管理](#part-1aws-iam--開發人員身份與權限管理)
- [PART 2：AWS CLI 設定與 S3 基本操作](#part-2aws-cli-設定與-s3-基本操作)
- [PART 3：Feature 3 — S3 檔案管理頁面開發](#part-3feature-3--s3-檔案管理頁面開發)
- [PART 4：Docker 化與推送 Docker Hub](#part-4docker-化與推送-docker-hub)
- [PART 5：AWS EC2 部署 + IAM Role 安全存取 S3](#part-5aws-ec2-部署--iam-role-安全存取-s3)
- [PART 6：GitHub PR 流程 — 回報與 Merge](#part-6github-pr-流程--回報與-merge)
- [PART 7：快速指令速查表](#part-7快速指令速查表)
- [重要注意事項](#️-重要注意事項整理)

---

## 📋 實驗概述

本次實驗分為前後兩個階段，串連 AWS 雲端服務與 DevOps 開發流程：

### 前半段 — 雲端開發

- 透過 IAM 替開發人員 DEV_pom 新增 S3 操作權限並取得 Access Key
- 安裝 AWS CLI，透過金鑰登入 AWS
- 在 ap-east-2 區域建立 S3 Bucket：ckc101-23

### 後半段 — 專案功能開發 & 部署

- 開發 Feature 3：S3 檔案上傳下載管理頁面
- 撰寫單元測試（pytest + Mock），不連真實 S3 即可驗證邏輯
- Docker 化專案並推送 Image 到 Docker Hub
- 建立 EC2 虛擬主機並綁定 IAM Role，讓程式碼不需存放任何金鑰
- 從 Docker Hub 拉取 Image 並執行，確認外部瀏覽器可連線且 S3 存取正常
- 建立 Git 新分支提交 PR，主管審核後 Merge 到 main 並刪除分支

---

## PART 1：AWS IAM — 開發人員身份與權限管理

IAM（Identity and Access Management）是 AWS 的權限管理核心，控制「誰可以對哪些資源做什麼操作」。

### 1.1 為開發人員建立 IAM 使用者

管理員需要先建立 IAM 使用者並附加對應的 S3 政策，開發人員才能操作 S3。

#### 操作步驟

1. AWS Console → 搜尋 **IAM** → 點進去
2. 左側選單 → **Users** → **Create user**
3. 輸入使用者名稱（例如 `DEV_pom`）
4. **Permissions** → **Attach policies directly**
5. 搜尋並勾選適合的政策（見下表）→ Create user

| 政策名稱 | 權限範圍 | 適用情境 |
|---------|---------|---------|
| `AmazonS3FullAccess` | 所有 Bucket 的完整讀寫刪 | 測試環境，快速入門 |
| `AmazonS3ReadOnlyAccess` | 只能讀取，不能寫入或刪除 | 只需查看資料的角色 |
| 自訂政策（推薦） | 只開放特定 Bucket 的指定操作 | 正式環境，最小權限原則 |

> 💡 正式環境建議使用自訂政策，只開放 `ckc101-23` 這個 Bucket 的必要操作，降低資安風險

---

### 1.2 取得 Access Key（開發人員憑證）

Access Key 是開發人員在本機命令列驗證 AWS 身份的憑證，由 **Access Key ID** 與 **Secret Access Key** 組成。

#### 取得步驟（管理員操作）

1. 進入 IAM → Users → 點選目標使用者（`DEV_pom`）
2. 點選 **Security credentials** 分頁
3. 往下找到 **Access keys** 區塊 → 點 **Create access key**
4. 使用情境選擇（見下表）→ Next
5. 畫面顯示金鑰後立刻複製，或點 **Download .csv file** 儲存備份

| 使用情境選項 | 適用場合 |
|------------|---------|
| **Command Line Interface (CLI)** | 本機 PowerShell / Terminal 操作 ← 本實驗選此項 |
| Application running outside AWS | 本地應用程式（非 EC2 環境） |
| Application running on AWS compute | EC2 / Lambda 等（建議改用 IAM Role） |
| Local code | 本地開發程式碼 |

> ⚠️ **Secret Access Key 只會顯示這一次！** 關掉視窗後無法再查看，務必立刻下載 CSV

| 注意事項 | 說明 |
|---------|------|
| 每個 IAM 使用者最多 2 組金鑰 | 超過需先刪除舊的才能再建 |
| Secret Key 遺失 | 只能刪掉舊金鑰，重新建立一組 |
| 金鑰不要上傳 GitHub | 任何人都能拿到，會造成資安事故 |
| 不用時建議 Deactivate | 暫停使用金鑰，降低外洩風險 |
| 金鑰不要存進 S3 Bucket | 等同公開金鑰，`.pem` 等檔案都不應上傳 |

---

## PART 2：AWS CLI 設定與 S3 基本操作

### 2.1 安裝 AWS CLI

```powershell
# Windows PowerShell
winget install Amazon.AWSCLI
# 或下載安裝包：https://awscli.amazonaws.com/AWSCLIV2.msi
```

```bash
# Linux / macOS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

```bash
# 確認安裝成功
aws --version
# aws-cli/2.x.x Python/3.x.x ...
```

---

### 2.2 設定金鑰（aws configure）

```bash
aws configure
```

依序輸入四個欄位：

```
AWS Access Key ID [None]:      AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]:  wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]:    ap-east-2        ← 台北區域
Default output format [None]:  json
```

| Output Format | 說明 | 適合情境 |
|--------------|------|---------|
| `json` | 結構化資料（預設推薦） | 程式碼解析、一般使用 |
| `table` | ASCII 表格顯示 | 人眼閱讀最清晰 |
| `text` | Tab 分隔純文字 | grep / awk / shell 腳本 |
| `yaml` | YAML 格式 | 設定檔相關操作 |

> 📌 金鑰儲存位置：Linux/macOS → `~/.aws/credentials`　｜　Windows → `C:\Users\帳號\.aws\credentials`

---

### 2.3 驗證身份

```bash
aws sts get-caller-identity
```

成功回傳範例：

```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "596319332433",
    "Arn": "arn:aws:iam::596319332433:user/DEV_pom"
}
```

---

### 2.4 建立 S3 Bucket

#### Bucket 命名規則

| 規則 | 說明 |
|------|------|
| 長度 | 3–63 個字元 |
| 允許字元 | 小寫英文、數字、連字號 `-` |
| 不允許 | 大寫字母、底線、空格、點開頭/結尾 |
| 全球唯一 | 名稱在全世界所有 AWS 帳號中不能重複 |

```bash
# 建立 ckc101-23
aws s3 mb s3://ckc101-23 --region ap-east-2
# 成功回傳：make_bucket: ckc101-23

# 確認 Bucket 出現在清單中
aws s3 ls
```

---

### 2.5 常用 S3 CLI 指令

| 指令 | 說明 |
|------|------|
| `aws s3 ls` | 列出帳號下所有 Bucket |
| `aws s3 ls s3://ckc101-23/` | 列出 Bucket 內容 |
| `aws s3 cp ./file.txt s3://ckc101-23/` | 上傳檔案 |
| `aws s3 cp s3://ckc101-23/file.txt ./` | 下載檔案 |
| `aws s3 sync ./folder s3://ckc101-23/folder/` | 同步整個資料夾 |
| `aws s3 rm s3://ckc101-23/file.txt` | 刪除 S3 上的檔案 |
| `aws s3api get-bucket-location --bucket ckc101-23` | 查詢 Bucket 所在 Region |

---

### 2.6 常見錯誤排查

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `AccessDenied / not authorized` | IAM 政策未授予對應操作權限 | 請管理員在 IAM 新增 S3 相關政策 |
| `InvalidClientTokenId` | Access Key ID 輸入錯誤 | 重新確認並設定 `aws configure` |
| `IllegalLocationConstraintException` | 程式碼 Region 與 Bucket 實際 Region 不符 | boto3 client 要明確指定 `region_name` |
| `Unable to locate credentials` | 未執行 `aws configure` | 執行 `aws configure` 輸入金鑰 |
| `NoSuchBucket` | Bucket 名稱不存在 | 確認名稱拼字或先建立 Bucket |

---

## PART 3：Feature 3 — S3 檔案管理頁面開發

### 3.1 功能說明

訪問 `/feature3` 路由可看到完整的 S3 檔案管理介面：

| 功能 | 說明 | 技術細節 |
|------|------|---------|
| 上傳 | 拖曳或點擊上傳任意格式檔案 | `s3.upload_fileobj()` 直接上傳到 ckc101-23 |
| 列表 | 顯示所有檔案名稱、大小、最後修改時間 | `s3.list_objects_v2()` 取得 Contents |
| 下載 | 點擊下載按鈕取得檔案 | `generate_presigned_url()`，5 分鐘有效連結 |
| 刪除 | 移除 S3 上的指定檔案 | `s3.delete_object()` |

---

### 3.2 安全憑證設計原則

本專案採用「**零金鑰程式碼**」原則：程式碼中完全不存放任何 Access Key，由 boto3 自動憑證鏈處理：

| 環境 | 憑證來源 | 需要修改程式碼？ |
|------|---------|--------------|
| 本地開發 | `~/.aws/credentials`（`aws configure` 設定） | 不需要 |
| EC2 正式環境 | IAM Role 自動提供短期臨時憑證 | 不需要 |
| GitHub 程式碼 | 無任何金鑰資訊 | 安全 ✅ |

boto3 自動憑證取得優先順序：

```
環境變數（AWS_ACCESS_KEY_ID 等）
  → ~/.aws/credentials 設定檔
    → EC2 Instance Metadata Service（IAM Role）
```

> 💡 部署到 EC2 後，程式碼完全不需要修改，boto3 自動切換使用 IAM Role 的臨時憑證

---

### 3.3 核心程式碼

#### S3 Client 初始化（不寫金鑰）

```python
import boto3
from botocore.exceptions import ClientError

BUCKET_NAME = 'ckc101-23'

# ✅ 正確：不寫金鑰，boto3 自動從環境或 IAM Role 取得憑證
s3 = boto3.client('s3', region_name='ap-east-2')

# ❌ 錯誤：絕對不要這樣寫
# s3 = boto3.client('s3', aws_access_key_id='AKIA...',
#                   aws_secret_access_key='xxxx')
```

#### 下載功能 — 使用 Presigned URL

```python
@app.route('/feature3/download/<filename>')
def download(filename):
    url = s3.generate_presigned_url(
        'get_object',
        Params={'Bucket': BUCKET_NAME, 'Key': filename},
        ExpiresIn=300  # 5 分鐘後連結自動失效
    )
    return redirect(url)
```

> 📌 Presigned URL 讓使用者直接從 S3 下載，Bucket 不需要設為公開，更安全
> 📌 若出現 `IllegalLocationConstraintException`，代表 boto3 client 的 `region_name` 與 Bucket 所在 Region 不符，需對齊

---

### 3.4 本地開發與測試

```bash
# 安裝依賴
pip install -r requirements.txt

# 執行單元測試（不連接真實 S3）
python -m pytest
```

> 📌 pytest 使用 Mock 模擬 boto3，可在離線狀態下驗證所有 S3 操作邏輯

```bash
# 本地啟動 Flask
python src/app.py
# 訪問：http://127.0.0.1:19191/feature3
```

```powershell
# 本地測試連接真實 S3（PowerShell 設定環境變數）
$env:AWS_ACCESS_KEY_ID="您的 Access Key ID"
$env:AWS_SECRET_ACCESS_KEY="您的 Secret Access Key"
$env:AWS_DEFAULT_REGION="ap-east-2"
python src/app.py
```

> ⚠️ 環境變數只在當次終端機 Session 有效，不會寫入程式碼，關掉終端機就清除

```bash
# 使用 Docker 容器化啟動
docker-compose up --build          # 建置並啟動
docker-compose up --build -d       # 背景執行
docker-compose down                # 停止容器
```

---

## PART 4：Docker 化與推送 Docker Hub

### 4.1 Phase 1 — 建立 Docker 相關設定檔

#### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 19191
CMD ["python", "SRC/app.py"]
```

> 📌 EXPOSE 的 port 必須與 Flask 程式實際監聽的 port 一致（本例為 19191）

#### .dockerignore（減少 Image 大小）

```
.git
.pytest_cache
TEST/
__pycache__
*.pyc
.env
*.pem
.aws/
```

#### docker-compose.yml

```yaml
version: '3.8'
services:
  app:
    build: .
    container_name: myproject
    ports:
      - "19191:19191"
    environment:
      - PYTHONUNBUFFERED=1
```

#### 完成後的專案結構

| 檔案 / 資料夾 | 說明 |
|-------------|------|
| `SRC/` | 主要程式碼（包含 app.py） |
| `TEST/` | 單元測試程式碼 |
| `Dockerfile` | 定義如何建立 Docker Image |
| `docker-compose.yml` | 定義如何一鍵啟動容器 |
| `.dockerignore` | 排除不需進入 Image 的檔案 |
| `.gitignore` | 排除不需進入 Git 的檔案（含 .env、.pem） |
| `requirements.txt` | Python 套件清單 |

---

### 4.2 Phase 2 — Build & 推送到 Docker Hub

```bash
# Build Image
docker-compose build
# 或
docker build -t myproject-web:latest .
docker images   # 確認 Image 出現在清單中

# 登入並推送
docker login                                     # 輸入 Docker Hub 帳號密碼
docker tag myproject-web:latest 帳號/myproject:latest
docker push 帳號/myproject:latest
```

> 📌 tag 格式固定為：`帳號/專案名稱:版本`　← 帳號必須與 Docker Hub 帳號完全一致

---

### 4.3 Phase 3 — Push 到 GitHub

```bash
git checkout -b docker
git add .
git commit -m "Add Docker deployment"
git push -u origin docker
```

> 📌 出現 `LF will be replaced by CRLF` 是 Windows 換行符號轉換，屬正常行為，不影響功能

---

## PART 5：AWS EC2 部署 + IAM Role 安全存取 S3

這是整個實驗最關鍵的部分：讓 EC2 虛擬主機透過 IAM Role 取得臨時憑證，程式碼完全不需要存放任何 Access Key。

### 5.1 Phase 1 — 建立 IAM Policy（最小權限原則）

只開放對 `ckc101-23` 這個 Bucket 的必要操作，不給予多餘權限。

1. AWS Console → IAM → Policies → **Create policy**
2. 切換到 JSON 索引標籤，貼上以下內容：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListBucketContents",
            "Effect": "Allow",
            "Action": ["s3:ListBucket"],
            "Resource": "arn:aws:s3:::ckc101-23"
        },
        {
            "Sid": "ObjectReadWriteDelete",
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
            "Resource": "arn:aws:s3:::ckc101-23/*"
        }
    ]
}
```

3. 命名：`EC2-S3-ckc101-23-Policy` → Create policy

> 📌 Resource 分兩層：`arn:aws:s3:::ckc101-23` 控制 Bucket 層級（ListBucket）；`ckc101-23/*` 控制物件層級（GetObject 等）

---

### 5.2 Phase 2 — 建立 IAM Role

1. IAM → Roles → **Create role**
2. Trusted entity type 選 **AWS service** → **EC2**
3. 搜尋並勾選剛才建立的政策：`EC2-S3-ckc101-23-Policy`
4. 命名：`EC2-S3-Access-Role` → Create role

> 💡 IAM Role 使用短期臨時憑證（STS Token），比長期 Access Key 更安全，EC2 會自動輪換

---

### 5.3 Phase 3 — 建立 EC2 Instance

| 設定項目 | 選擇值 | 說明 |
|---------|--------|------|
| Name | my-docker-server（自訂） | 辨識用名稱 |
| AMI | Ubuntu Server 24.04 LTS | 本實驗使用的 OS |
| Instance type | t2.micro | 免費方案（Free Tier） |
| Key pair | 建立新的 .pem 檔 | SSH 登入用，務必下載保存 |
| Security Group - SSH | Port 22，來源：你的 IP | 限制只有你能 SSH 登入 |
| Security Group - App | Port 19191，來源：0.0.0.0/0 | 開放外部瀏覽器連線 |
| **IAM instance profile** | **EC2-S3-Access-Role** | **← 最重要！綁定 Role 才能存取 S3** |

> 📌 IAM instance profile 在 Launch Instance → **Advanced details** 最下方設定
> ⚠️ 如果忘記在建立時綁定，事後也可到 EC2 → Actions → Security → **Modify IAM role** 補設定

---

### 5.4 Phase 4 — SSH 連線 & 安裝 Docker

```bash
# Linux / macOS
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2公開IP>

# Windows PowerShell
ssh -i your-key.pem ubuntu@<EC2公開IP>
```

```bash
# 安裝 Docker
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker          # 開機自動啟動
sudo usermod -aG docker ubuntu        # 讓 ubuntu 用戶可執行 docker 指令
exit                                  # 重新登入讓群組設定生效
```

> ⚠️ 必須 `exit` 重新 SSH 登入，`usermod` 的 docker 群組設定才會生效，否則執行 docker 會出現 `Permission denied`

---

### 5.5 Phase 5 — 驗證 IAM Role & 部署容器

```bash
# 安裝 AWS CLI
sudo apt install -y awscli

# 直接查詢身份（不需要 aws configure，IAM Role 自動提供臨時憑證）
aws sts get-caller-identity
```

成功回傳（注意 `assumed-role` 字樣代表 IAM Role 已生效）：

```json
{
    "Arn": "arn:aws:sts::596319332433:assumed-role/EC2-S3-Access-Role/i-xxx"
}
```

```bash
# 直接測試 S3 存取
aws s3 ls s3://ckc101-23

# 從 Docker Hub 拉取並執行容器
docker pull pompom85/flask-dashboard:latest
docker run -d -p 19191:19191 --name flask-app pompom85/flask-dashboard:latest

# 確認容器運作正常
docker ps                    # 確認容器狀態為 Up
docker logs flask-app        # 查看 Flask 啟動日誌
```

瀏覽器測試：

```
http://<EC2公開IP>:19191/feature3
```

> 📌 EC2 已綁定 IAM Role，容器內的 boto3 自動向 EC2 Metadata Service（IMDS）取得臨時金鑰，完全不需要傳入任何環境變數
> 💡 如果連不上，先確認 AWS Security Group Inbound Rules 是否已開放 Port 19191

---

### 5.6 IAM Role 運作原理

| 傳統 Access Key 方式 | IAM Role 方式（推薦） |
|--------------------|-------------------|
| 固定長期金鑰，手動管理 | 自動短期臨時憑證（15分鐘~1小時），自動輪換 |
| 金鑰可能外洩或意外提交到 GitHub | 程式碼無任何金鑰，無法外洩 |
| 需要在每台機器設定 `aws configure` | EC2 綁定 Role 後自動生效，無需額外設定 |
| 撤銷需要手動刪除金鑰 | 直接修改或移除 Role 即可立即撤銷 |

---

## PART 6：GitHub PR 流程 — 回報與 Merge

### 6.1 建立 report branch 並推送

```bash
# 確保在最新的 main 分支上
git checkout main
git pull origin main

# 建立 report branch
git checkout -b report/docker-deployment

# 新增或更新 README.md / REPORT.md
git add .
git commit -m "report: Docker deployment test completed"
git push -u origin report/docker-deployment
```

---

### 6.2 在 GitHub 建立 Pull Request

1. 進入 GitHub repo
2. 點擊黃色提示列的「**Compare & pull request**」
3. 填寫 Title 和 Description（說明本次部署的內容與測試結果）
4. 在 **Assignees** 指定主管審核
5. 點「**Create pull request**」

---

### 6.3 主管審核 & Merge

- 主管審核後點「**Merge pull request**」→「**Confirm merge**」
- Merge 完成後點「**Delete branch**」刪除遠端分支

---

### 6.4 本地清理分支

```bash
git checkout main
git pull origin main
git branch -d report/docker-deployment            # 刪除本地分支
git push origin --delete report/docker-deployment  # 刪除遠端分支（若網頁未刪）
```

---

## PART 7：快速指令速查表

### 7.1 AWS CLI

| 指令 | 說明 |
|------|------|
| `aws configure` | 設定金鑰、Region、Output 格式 |
| `aws sts get-caller-identity` | 確認目前登入身份（含 IAM Role 驗證） |
| `aws s3 ls` | 列出所有 Bucket |
| `aws s3 mb s3://名稱 --region 區域` | 建立 Bucket |
| `aws s3 ls s3://bucket/` | 列出 Bucket 內容 |
| `aws s3 cp 本地檔案 s3://bucket/` | 上傳檔案 |
| `aws s3 cp s3://bucket/檔案 ./` | 下載檔案 |
| `aws s3 sync ./folder s3://bucket/folder/` | 同步資料夾 |
| `aws s3 rm s3://bucket/檔案` | 刪除 S3 上的檔案 |
| `aws s3api get-bucket-location --bucket 名稱` | 查詢 Bucket 所在 Region |

---

### 7.2 Docker 指令

| 指令 | 說明 |
|------|------|
| `docker build -t 名稱:版本 .` | Build Image |
| `docker images` | 查看所有 Image |
| `docker-compose build` | 用 compose 建立 Image |
| `docker-compose up -d` | 背景執行容器 |
| `docker-compose up --build -d` | 重新 Build 並背景執行 |
| `docker-compose down` | 停止並移除容器 |
| `docker ps` | 查看執行中的容器 |
| `docker ps -a` | 查看所有容器（含已停止） |
| `docker logs 容器名稱` | 查看容器 Log |
| `docker stop 容器名稱` | 停止容器（優雅關閉，等 SIGTERM） |
| `docker kill 容器名稱` | 強制終止容器（直接 SIGKILL） |
| `docker rm 容器名稱` | 刪除容器 |
| `docker pull 帳號/名稱:版本` | 從 Docker Hub 下載 Image |
| `docker push 帳號/名稱:版本` | 推送 Image 到 Docker Hub |
| `docker login` | 登入 Docker Hub |
| `docker tag 原名稱 帳號/新名稱:版本` | 重新標記 Image（推送前必做） |
| `docker run -d -p 外:內 --name 名稱 image` | 背景執行並指定 port 映射 |

---

### 7.3 Git 指令

| 指令 | 說明 |
|------|------|
| `git checkout -b 分支名稱` | 建立並切換到新分支 |
| `git checkout 分支名稱` | 切換分支 |
| `git add .` | 加入所有變更到暫存區 |
| `git commit -m "訊息"` | Commit |
| `git push -u origin 分支名稱` | 首次推送並設定遠端追蹤 |
| `git pull origin main` | 拉取遠端 main 最新狀態 |
| `git branch -d 分支名稱` | 刪除本地分支（已 merge） |
| `git push origin --delete 分支名稱` | 刪除遠端分支 |
| `git log --oneline` | 查看精簡版 commit 紀錄 |
| `git status` | 查看目前工作目錄狀態 |

---

## ⚠️ 重要注意事項整理

| 分類 | 注意事項 |
|------|---------|
| 金鑰安全 | 程式碼中絕對不要寫死 Access Key ID 或 Secret Access Key |
| 金鑰安全 | S3 Bucket 絕對不要存放 `.pem` 金鑰檔案（任何人下載都能登入 EC2） |
| 金鑰安全 | 金鑰不要提交到 GitHub，`.gitignore` 要排除 `.env`、`.aws/`、`*.pem` |
| Region | boto3 client 要明確指定 `region_name`，避免 Presigned URL Region 錯誤 |
| Region | 建立 Bucket 時要指定 `--region`，與程式碼保持一致 |
| Docker | Dockerfile 的 `EXPOSE` port 要和 Flask 實際監聽的 port 一致 |
| Docker | `docker tag` 的帳號名稱要和 Docker Hub 帳號完全一致 |
| Docker | EC2 安裝完 docker 後必須 `exit` 重新登入，群組設定才會生效 |
| IAM Role | EC2 綁定 IAM Role 後，容器內 boto3 自動取得臨時憑證，不需傳入環境變數 |
| Security Group | AWS Security Group Inbound Rules 要開放對應 port，外部才能連線 |
| Git | `git` 指令必須在專案目錄下執行，否則出現 `not a git repository` 錯誤 |
| Git | `LF will be replaced by CRLF` 是 Windows 正常行為，不影響功能 |

---

## 📝 實驗總結

**前半段（雲端開發）**：先將開發人員透過 IAM 新增 S3 權限並取得 Access Key，接著安裝 AWS CLI，透過金鑰登入後建立 S3 Bucket。

**後半段（功能開發 & 部署）**：新增 Feature 3 功能並測試完畢後，打包成 Docker Image 推送到 Docker Hub。接著去 IAM 新增給 EC2 用的政策與角色，在 EC2 新增虛擬主機並綁定 IAM Role（賦予短期臨時憑證）。進入虛擬主機後拉取 Image 執行，確認外部可連線且 S3 存取正常，代表 IAM Role 設定成功。最後建立新分支推送 GitHub，主管審核後 Merge 到 main 並刪除分支。
