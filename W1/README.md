# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：MacOS 26.3.1
- VM 名稱：vct-w01-411631236
- Ubuntu 版本：

        Distributor ID:	Ubuntu
        Description:	Ubuntu 25.10
        Release:	25.10
        Codename:	questing

- Docker 版本：

        Docker version 29.3.0, build 5927d80
- Docker Compose 版本：

        Docker Compose version v5.1.0


## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 4 vCPU | `lscpu \| grep "^CPU(s)"` | 4 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | 3.3Gi |
| 磁碟 | 64 GB | `df -h /` | 62G |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` |  |

## 四層驗收證據
- [ ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出: 
        
        deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg]   https://download.docker.com/linux/ubuntu   questing stable
- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出:  

        ii
        docker-ce                                     5:29.3.0-1~ubuntu.25.10~questing           arm64        Docker: the open-source application container engine
        ii  docker-ce-cli                                 5:29.3.0-1~ubuntu.25.10~questing           arm64        Docker CLI: the open-source application container engine
        ii  docker-ce-rootless-extras                     5:29.3.0-1~ubuntu.25.10~questing           arm64        Rootless support for Docker.

- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
- [ ] Compose：`docker compose version` 可執行

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出

        <!DOCTYPE html>
        <html>
        <head>
        <title>Welcome to nginx!</title>
        <style>
        html { color-scheme: light dark; }
        body { width: 35em; margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif; }
        </style>
        </head>
        <body>
        <h1>Welcome to nginx!</h1>
        <p>If you see this page, nginx is successfully installed and working.
        Further configuration is required for the web server, reverse proxy, 
        API gateway, load balancer, content cache, or other features.</p>

        <p>For online documentation and support please refer to
        <a href="https://nginx.org/">nginx.org</a>.<br/>
        To engage with the community please visit
        <a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
        For enterprise grade support, professional services, additional 
        security features and capabilities please refer to
        <a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

        <p><em>Thank you for using nginx.</em></p>
        </body>
        </html>

- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出

        / # hostname
        d59298cc94b7
        / # cat /etc/os-release
        NAME="Alpine Linux"
        ID=alpine
        VERSION_ID=3.23.3
        PRETTY_NAME="Alpine Linux v3.23"
        HOME_URL="https://alpinelinux.org/"
        BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
        / # 
        / # ls /
        bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
        / # whoami
        root
        / # exit

- [ ] 映像列表：`sudo docker images` 輸出

        IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
        alpine:latest        25109184c71b       13.6MB         4.28MB        
        hello-world:latest   85404b3c5395       22.5kB         10.2kB        
        nginx:latest         dec7a90bd097        258MB         64.1MB     

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | 2026/03/26 03:38 | 包含最基礎的 Ubuntu 系統 | hostnamectl ,ip route ,sudo docker --version ,docker compose version ,sudo systemctl status docker --no-pager ,sudo docker run --rm hello-world |
| docker-ready | 2026/03/26 03:39 | 已安裝 Docker 與 Docker Compose | sudo systemctl status docker --no-pager ,sudo docker run --rm hello-world ,sudo docker images |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 60秒 | 20秒 |
| 適用情境 | 錯誤簡單知道問題在哪 | 無法判斷錯誤並且難以修復 |
| 風險 | 可能會留下未知的隱患 | 會失去沒有設定在快照中的工作進度 |

## Snapshot 保留策略
- 新增條件：每次安裝新工具或大改設定前，且當前狀態已驗證通過時。
- 保留上限：最多 3 個活躍 snapshot。
- 刪除條件：已有更新節點且舊節點確認不再需要時，刪最舊的。

## 最小可重現命令鏈
- 注入故障：

        sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
        sudo apt update

- 回覆驗證：

        ls /etc/apt/sources.list.d/
        cat /etc/apt/sources.list.d/docker.list
        sudo apt update

        sudo systemctl status docker --no-pager
        sudo docker --version
        docker compose version
        sudo docker run --rm hello-world
        sudo docker images

        free -h
        df -h /

## 排錯紀錄
- 症狀：
        執行 sudo apt update 時出現警告訊息：Notice: Ignoring file 'docker.list.broken'... as it has an invalid filename extension且嘗試執行 apt-cache policy docker-ce 時，發現 Candidate (候選版本) 失去遠端倉庫 (Repo) 連結，僅剩下本地已安裝的版本資訊
- 診斷：
        1.首先檢查 /etc/apt/sources.list.d/ 目錄下的文件列表，確認軟體源清單是否存在
        2.觀察 apt update 的輸出日誌，定位是否有文件被跳過或解析失敗
        3.使用 apt-cache policy 指令確認 docker-ce 的來源表格（Version table），發現網路來源 (https://download.docker.com/...) 消失
- 修正：
        將原本被重新命名為 .broken 的無效設定檔恢復為標準的 .list 副檔名，指令如下：
        sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
- 驗證：
        1.重新執行 sudo apt update，確認輸出日誌中恢復了對 download.docker.com 的讀取 (Hit:1)
        2.再次執行 apt-cache policy docker-ce，確認 Version table 重新出現優先級為 500 的遠端下載來源
        3.執行 sudo docker run --rm hello-world 成功輸出 "Hello from Docker!"，確認容器引擎通訊與映像檔拉取功能皆正常

## 設計決策
- 技術選擇： 使用獨立的 /etc/apt/sources.list.d/docker.list 管理第三方套件源，而非直接修改主系統的 sources.list
- 決策背景： 為了確保系統穩定性並方便故障排除（Troubleshooting）。
- 取捨：

        優點： 當 Docker 更新出現衝突或需要暫時停用時，只需透過重新命名單一文件即可達成（如實驗中的 .broken 注入），不會影響到 Ubuntu 官方核心套件的更新路徑。

        缺點： 增加了一點管理成本（需要多檢查一個目錄），但大幅降低了誤刪系統主軟體源的風險。