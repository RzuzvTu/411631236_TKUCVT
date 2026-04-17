# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔 | daemon.json（Docker daemon 設定），目前不存在代表使用預設值 |
| /var/lib/docker/ | 程式的持久性狀態資料 | 映像、容器、volumes、overlay2 層資料 |
| /usr/bin/docker | 使用者可執行檔 | Docker CLI 工具（42MB ELF 二進位） |
| /run/docker.sock | 執行期暫存（socket） | Docker daemon 的 Unix socket，開機建立關機消失 |

## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取 nginx:latest 前 /var/lib/docker/ 大小：232K
- 拉取 nginx:latest 後 /var/lib/docker/ 大小：232K（du 因權限限制未能完整統計，實際映像層存於 overlayfs 子目錄）

```
$ sudo docker info（關鍵欄位）
 Containers: 0
 Images: 3
 Storage Driver: overlayfs
 Docker Root Dir: /var/lib/docker
```

## 權限結構

### Docker Socket 權限解讀

```
srw-rw---- 1 root docker 0 Apr 17 19:28 /var/run/docker.sock
```

- `s`：檔案類型是 socket（不是普通檔案 `-` 也不是目錄 `d`）
- `rw-`（owner = root）：root 有讀寫權限
- `rw-`（group = docker）：docker 群組成員有讀寫權限
- `---`（others）：其他使用者完全無法存取

### 使用者群組

```
$ id
uid=1000(tt) gid=1000(tt) groups=1000(tt),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),116(lpadmin),981(docker)
```

加入 docker group 前：groups 不含 docker，`docker ps` 回傳 `permission denied`。  
執行 `sudo usermod -aG docker tt` 後重新登入，groups 包含 docker（gid=981），`docker ps` 不需 sudo 即可執行。

### 安全意涵

docker group 大約等於 root。原因：Docker daemon 以 root 身份運行，能存取 /var/run/docker.sock 就等於能對 daemon 下任意指令。實際示範：

```
$ docker run --rm -v /etc/shadow:/host-shadow:ro alpine cat /host-shadow
root:*:20368:0:99999:7:::
daemon:*:20368:0:99999:7:::
...
```

在不加 sudo 的情況下，docker group 成員可透過容器掛載 Host 的任意目錄並讀取敏感檔案（如 /etc/shadow）。生產環境不應隨意將使用者加入 docker group。

## 程序與服務管理

### systemctl status docker

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-04-18 03:23:02 CST
     Main PID: 1847 (dockerd)
     Memory: 131.4M
```

`enabled` 代表開機會自動啟動。Main PID 與 `ps aux | grep dockerd` 結果一致（PID 1847）。

### journalctl 日誌分析

```
Apr 18 03:23:01  Starting docker.service...
Apr 18 03:23:01  Starting up
Apr 18 03:23:02  Loading containers: start.
Apr 18 03:23:02  Restoring containers: start.
Apr 18 03:23:02  Docker daemon commit=f78c987 storage-driver=overlayfs version=29.3.1
Apr 18 03:23:02  Daemon has completed initialization
Apr 18 03:23:02  API listen on /run/docker.sock
Apr 18 03:23:02  Started docker.service
```

重點事件：daemon 啟動 → 載入容器 → 初始化完成 → 開始監聽 socket。最後一行「API listen on /run/docker.sock」代表 daemon 準備好接受請求。

### CLI vs Daemon 差異

Docker CLI（/usr/bin/docker）是一般的可執行檔，執行時只是一個短暫的 process，負責把指令轉成 API 請求送到 socket。Docker daemon（dockerd）是由 systemd 管理的長期背景服務，實際執行所有容器操作。

`docker --version` 只是 CLI 自己印版本，完全不需要跟 daemon 溝通——因此版本正常不代表 Docker 能用。必須用 `systemctl status docker` 確認 daemon 是否在跑。

## 環境變數

- $PATH：`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin`
- which docker：`/usr/bin/docker`（在 $PATH 搜尋順序中第 4 個目錄 /usr/bin 找到）

容器內外環境變數差異觀察：
- Host: `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin`、`HOME=/home/tt`、`USER=tt`
- Container (alpine): `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`、`HOME=/root`（沒有 USER 變數）
- 容器有自己獨立的環境，不繼承 Host 的 shell 設定，$PATH 也較短（沒有 /snap/bin 等 Host 特有路徑）

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active (running) | inactive (dead) | active (running) |
| docker --version | Docker version 29.3.1 | Docker version 29.3.1（正常） | Docker version 29.3.1 |
| docker ps | 正常（空列表） | Cannot connect to the Docker daemon at unix:///var/run/docker.sock | 正常（空列表） |
| ps aux \| grep dockerd | 有 process（PID 1847） | 無結果（process 不存在） | 有 process（新 PID） |

關鍵觀察：daemon 停止後 `docker --version` 仍然正常，確認 CLI 是獨立的執行檔，不依賴 daemon。

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- (0660) | srw------- (0600) | srw-rw---- (0660) |
| docker ps（不加 sudo） | 正常 | permission denied while trying to connect to the docker API | 正常 |
| sudo docker ps | 正常 | 正常（root 是 owner，有讀寫權限） | 正常 |
| systemctl status docker | active (running) | active (running) | active (running) |

關鍵觀察：daemon 全程在跑，`sudo docker ps` 仍可用，確認問題在 socket 權限而非 daemon。

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon at unix:///var/run/docker.sock | daemon 沒在跑，socket 不存在或無人監聽 | `systemctl status docker` → 若 inactive 則 `systemctl start docker` |
| permission denied while trying to connect to the docker API | daemon 在跑但 socket 權限不足，使用者無法存取 | `ls -la /var/run/docker.sock` 看權限，`id` 確認是否在 docker group |

兩種錯誤的根因不同，`sudo docker ps` 是最快的分辨方法：若 sudo 可用代表 daemon 在跑但是權限問題；若 sudo 也失敗代表 daemon 根本沒跑。

## 排錯紀錄

**症狀**：執行 `docker ps` 出現 `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

**診斷**：
1. 先確認 daemon 狀態：`systemctl status docker` → `active (running)`，代表 daemon 沒問題
2. 確認 socket 權限：`ls -la /var/run/docker.sock` → `srw------- root docker`，group 的 rw 權限被拿掉了
3. 確認自身群組：`id` → 使用者在 docker group（gid=981），但 socket 的 group 位元是 `---`，所以群組成員也無法存取

**修正**：
```bash
sudo chmod 660 /var/run/docker.sock
sudo chown root:docker /var/run/docker.sock
```

**驗證**：
```bash
ls -la /var/run/docker.sock   # 確認權限回到 srw-rw----
docker ps                      # 不加 sudo 正常執行
```

## 設計決策

**為什麼教學環境用 `usermod -aG docker $USER` 而不是每次 sudo？**

在教學和開發環境中，加入 docker group 可以減少每次操作的摩擦，讓學習者專注在容器本身的概念而不是每次都要輸入密碼。取捨是：docker group 成員等同 root 權限（本週安全示範已證明可讀取 /etc/shadow），一旦帳號被入侵，攻擊者等同取得 root 存取。生產環境的正確做法是每次用 sudo（保留操作記錄和明確授權），或改用 rootless Docker（daemon 以非 root 身份運行，根本消除這個問題）。

## 可重跑最小命令鏈

```bash
which docker
systemctl status docker --no-pager | head -5
ls -la /var/run/docker.sock
docker ps
docker run --rm hello-world
```
