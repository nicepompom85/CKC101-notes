# 專案 Docker 化完整流程筆記

> Flask 專案 → Docker Image → Docker Hub → AWS EC2 → GitHub PR
>
> 日期：2026-05-28

---

## 📋 課程概述

本次課程完成以下任務：

- 將 Python Flask 專案 Docker 化，建立 Dockerfile 與 docker-compose.yml
- Build Docker Image 並推送到 Docker Hub
- 在 AWS EC2 建立 Ubuntu 虛擬機器，模擬其他台電腦的環境
- 在 EC2 上安裝 Docker，從 Docker Hub 下載 Image 並執行
- 測試從外部瀏覽器連接到 Flask 網站成功
- 建立新分支回報主管，完成 Pull Request 與 Merge，最後刪除分支

### 為什麼要 Docker 化？

| 問題 | Docker 的解決方式 |
|------|-----------------|
| 「在我電腦可以跑，別人的不行」 | Image 封裝所有依賴，任何有 Docker 的機器都能執行 |
| 環境設定繁瑣 | 一個 `docker run` 指令即可啟動，無需手動安裝依賴 |
| 跨平台差異（Windows/Linux/macOS） | 容器內部環境一致，不受宿主機 OS 影響 |
| 部署步驟複雜 | 推送到 Docker Hub，任何地方都能 `docker pull` 取得 |

---

## PHASE 1：專案 Docker 化

### Step 1 — 確認 requirements.txt

列出所有 Python 套件依賴，讓 Docker 在容器內重新安裝時有依據：

```bash
pip freeze > requirements.txt
```

> 📌 建議定期更新此檔案，確保版本與本地開發環境一致

---

### Step 2 — 建立 Dockerfile

在專案根目錄建立 `Dockerfile`，定義如何建立 Image：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 19191

CMD ["python", "SRC/app.py"]
```

**每一行的說明：**

| 指令 | 說明 |
|------|------|
| `FROM python:3.12-slim` | 以官方 Python 3.12 精簡版作為基底 Image |
| `WORKDIR /app` | 設定容器內的工作目錄 |
| `COPY requirements.txt .` | 先只複製 requirements，利用 Docker 層快取加速 |
| `RUN pip install ...` | 安裝所有 Python 依賴（`--no-cache-dir` 減少 Image 大小） |
| `COPY . .` | 複製所有專案程式碼進容器 |
| `EXPOSE 19191` | 宣告容器監聽的 port（文件用途，不實際開放） |
| `CMD [...]` | 容器啟動時執行的預設指令 |

> ⚠️ EXPOSE 的 port 要和 Flask 程式實際使用的 port 一致

---

### Step 3 — 建立 .dockerignore

排除不需要進入 Image 的檔案，縮小 Image 大小並提升安全性：

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

> 📌 `.pem` 和 `.aws/` 加入 dockerignore，防止金鑰意外打包進 Image

---

### Step 4 — 建立 docker-compose.yml

定義如何執行容器，方便一鍵啟動：

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

**ports 格式說明：`"宿主機port:容器port"`**

> 📌 `PYTHONUNBUFFERED=1` 讓 Python 的 print/log 即時輸出，不會被緩衝，方便 `docker logs` 查看

---

### 完成後的專案結構

```
MyProject/
├── SRC/                  # 主要程式碼
│   └── app.py
├── TEST/                 # 測試程式碼
├── Dockerfile            # 定義如何建立 Image
├── docker-compose.yml    # 定義如何執行容器
├── .dockerignore         # 排除不需要的檔案
├── .gitignore            # 排除不需要進 Git 的檔案
├── requirements.txt      # Python 套件清單
└── README.md             # 專案說明文件
```

---

## PHASE 2：Build Image 並推送到 Docker Hub

### Step 5 — Build Docker Image

在專案根目錄執行（需要先開啟 Docker Desktop）：

```bash
# 方式一：使用 docker-compose
docker-compose build

# 方式二：直接使用 docker build
docker build -t myproject-web:latest .
```

---

### Step 6 — 確認 Image 建立成功

```bash
docker images
```

應看到 `myproject-web:latest` 出現在清單中：

```
REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
myproject-web     latest    abc123def456   2 minutes ago   180MB
```

---

### Step 7 — 登入 Docker Hub

先至 [hub.docker.com](https://hub.docker.com) 註冊帳號，再執行：

```bash
docker login
# 輸入 Docker Hub 的帳號和密碼
# 顯示 Login Succeeded 即成功
```

---

### Step 8 — 標記 Image

推送前必須將 Image 重新標記為 `帳號/專案名稱:版本` 格式：

```bash
# 格式：docker tag 原本名稱 帳號/專案名稱:版本
docker tag myproject-web:latest pompom85/myproject:latest
```

> ⚠️ 帳號名稱必須與 Docker Hub 帳號完全一致，否則推送會失敗

---

### Step 9 — 推送到 Docker Hub

```bash
docker push pompom85/myproject:latest
```

完成後至 [hub.docker.com](https://hub.docker.com) 確認 Repository 已出現。

---

## PHASE 3：Push Docker 檔案到 GitHub

### Step 10 — 建立 docker branch 並推送

```bash
git checkout -b docker
git add .
git commit -m "Add Docker deployment"
git push -u origin docker
```

> 📌 出現 `LF will be replaced by CRLF` 的 warning 是正常的，Windows 換行符號轉換，不影響功能

---

## PHASE 4：AWS EC2 建立 Ubuntu 虛擬機器

### Step 11 — 開啟 EC2 Instance

登入 AWS Console → EC2 → **Launch Instance**，設定如下：

| 設定項目 | 選擇值 | 說明 |
|---------|--------|------|
| Name | my-docker-server（自訂） | 辨識用名稱 |
| AMI | Ubuntu Server 24.04 LTS | 輕量、常用的 Linux 發行版 |
| Instance type | t2.micro | 免費方案（Free Tier） |
| Key pair | 建立新的 .pem 檔並下載保存 | SSH 登入用，只能下載一次 |
| Security Group - SSH | Port 22，來源：你的 IP | 限制只有你能 SSH 登入 |
| Security Group - App | Port 19191，來源：0.0.0.0/0 | 開放外部瀏覽器連線 |

> ⚠️ `.pem` 金鑰檔案務必妥善保存，遺失後無法重新下載，只能重新建立 Key pair

---

### Step 12 — SSH 連線進入 EC2

```bash
# Linux / macOS
chmod 400 你的金鑰.pem
ssh -i 你的金鑰.pem ubuntu@EC2公開IP

# Windows PowerShell — 先設定 .pem 權限
icacls 你的金鑰.pem /inheritance:r /grant:r "%username%:R"
ssh -i 你的金鑰.pem ubuntu@EC2公開IP
```

---

### Step 13 — 在 EC2 安裝 Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker        # 開機自動啟動
sudo usermod -aG docker ubuntu      # 讓 ubuntu 用戶可執行 docker
exit                                # 重新登入讓群組設定生效
```

> ⚠️ 必須 `exit` 重新 SSH 登入，`usermod` 的群組設定才會生效，否則執行 docker 指令會出現 `Permission denied`

---

## PHASE 5：從 Docker Hub 下載並執行

### Step 14 — Pull Image

```bash
docker pull pompom85/myproject:latest
```

### Step 15 — 執行容器

```bash
docker run -d -p 19191:19191 --name myproject pompom85/myproject:latest
```

**參數說明：**

| 參數 | 說明 |
|------|------|
| `-d` | Detached mode，背景執行 |
| `-p 19191:19191` | 宿主機 port 映射到容器 port |
| `--name myproject` | 指定容器名稱，方便後續管理 |

---

### Step 16 — 確認容器運作

```bash
docker ps               # 確認容器狀態為 Up
docker logs myproject   # 查看 Flask 啟動日誌
```

---

### Step 17 — 瀏覽器測試

在本機瀏覽器輸入：

```
http://EC2公開IP:19191
```

> 📌 如果連不上，先確認 AWS Security Group 是否已開放對應 port
> 📌 用 `docker logs` 確認 Flask 實際運行的 port 號

---

## PHASE 6：回報主管 & GitHub PR 流程

### Step 18 — 在本機建立 report branch

```bash
cd Desktop\MyProject    # 確認在專案目錄
git checkout main
git pull origin main
git checkout -b report/docker-deployment
```

---

### Step 19 — 新增回報文件並推送

```bash
# 新增或修改 README.md 或 REPORT.md
git add .
git commit -m "report: Docker deployment test completed"
git push -u origin report/docker-deployment
```

---

### Step 20 — 在 GitHub 建立 Pull Request

1. 進入 GitHub repo
2. 點擊黃色提示列的「**Compare & pull request**」
3. 填寫 Title 和 Description（說明部署結果與測試情況）
4. 在 **Assignees** 指定主管審核
5. 點「**Create pull request**」

---

### Step 21 — 主管 Merge（主管操作）

1. 主管審核後點「**Merge pull request**」
2. 點「**Confirm merge**」

---

### Step 22 — 刪除分支

GitHub 網頁上（merge 後出現提示）：
- 點「**Delete branch**」按鈕

或在本地執行：

```bash
git checkout main
git pull origin main
git branch -d report/docker-deployment
git push origin --delete report/docker-deployment
```

---

## 📊 整體流程總覽

| 階段 | 動作 | 工具 |
|------|------|------|
| PHASE 1 | 建立 Dockerfile / docker-compose.yml / .dockerignore | VS Code |
| PHASE 2 | Build Image → 標記 → 推送到 Docker Hub | Docker Desktop / CLI |
| PHASE 3 | 建立 docker branch → Push 到 GitHub | Git / GitHub |
| PHASE 4 | AWS EC2 建立 Ubuntu 虛擬機器 | AWS Console |
| PHASE 5 | EC2 安裝 Docker → Pull Image → 執行容器 → 測試 | SSH / Docker CLI |
| PHASE 6 | 建立 report branch → PR → Merge → 刪除分支 | Git / GitHub |

---

## ⚡ 常用指令速查表

### Docker 指令

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
| `docker stop 容器名稱` | 停止容器（優雅關閉） |
| `docker kill 容器名稱` | 強制終止容器 |
| `docker rm 容器名稱` | 刪除容器 |
| `docker pull 帳號/名稱:版本` | 從 Docker Hub 下載 Image |
| `docker push 帳號/名稱:版本` | 推送 Image 到 Docker Hub |
| `docker login` | 登入 Docker Hub |
| `docker tag 原名稱 新名稱` | 重新標記 Image |
| `docker run -d -p 外:內 --name 名稱 image` | 背景執行並指定 port 映射 |

### Git 指令

| 指令 | 說明 |
|------|------|
| `git checkout -b 分支名稱` | 建立並切換到新分支 |
| `git checkout 分支名稱` | 切換分支 |
| `git add .` | 加入所有變更到暫存區 |
| `git commit -m "訊息"` | Commit |
| `git push -u origin 分支名稱` | 首次推送並設定追蹤 |
| `git pull origin main` | 拉取遠端 main 最新狀態 |
| `git branch -d 分支名稱` | 刪除本地分支 |
| `git push origin --delete 分支名稱` | 刪除遠端分支 |
| `git log --oneline` | 查看精簡版 commit 紀錄 |
| `git status` | 查看目前工作目錄狀態 |

---

## ⚠️ 注意事項

- Dockerfile 的 `EXPOSE` port 要和 Flask 程式實際的 port 一致
- `docker tag` 的帳號名稱要和 Docker Hub 帳號完全一致
- AWS Security Group 的 Inbound Rules 要開放對應 port，來源設 `0.0.0.0/0`
- `git` 指令必須在專案目錄下執行，否則出現 `not a git repository` 錯誤
- `LF will be replaced by CRLF` 是 Windows 正常行為，不是錯誤
- EC2 安裝完 docker 要 `exit` 重新登入，`usermod` 的群組設定才會生效
- `.pem` 金鑰不要存入 S3 Bucket 或上傳到 GitHub

---

## 📝 實驗總結

第一個實驗核心流程：

1. **本機**：Flask 專案 Docker 化 → Build Image → 推送到 Docker Hub
2. **GitHub**：建立 `docker` branch → Push 程式碼
3. **AWS EC2**：開一台 Ubuntu 虛擬機 → 安裝 Docker → 從 Docker Hub 拉取 Image → 執行容器
4. **驗證**：從外部瀏覽器成功連到 Flask 網站
5. **回報**：建立 `report/docker-deployment` branch → PR → 主管 Merge → 刪除分支
