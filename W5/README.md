# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：runc

驗證指令：
```bash
docker info 2>/dev/null | grep -E "Storage Driver|Cgroup Driver|Cgroup Version|Runtime"
```

---

## Namespace 觀察

### 六種 namespace 用途（用自己的話）

- **PID**：每個容器有自己的 PID 編號空間，容器內的第一個 process 看到自己是 PID 1，但在 host 上它其實是一個普通的大號 PID（本次實驗是 6024）。兩個容器互看不到彼此的 process。
- **NET**：容器有自己獨立的網路介面、IP 位址和路由表。容器內只看到 `lo` 和 `eth0`，看不到 host 的 `enp2s0`、`docker0` 等介面，網路協定堆疊完全分開。
- **MNT**：容器有自己的根目錄 `/`，掛載點視角與 host 完全不同。容器內的 `/etc`、`/var` 是 image 的檔案系統，看不到 host 的真實磁碟掛載。
- **UTS**：容器有自己的 hostname（預設是 container id 前 12 碼），和 host 的 hostname（`app`）完全獨立，互相修改不影響對方。
- **IPC**：容器有獨立的 System V IPC 資源（共享記憶體、semaphore、message queue），容器之間無法透過 IPC 互相通訊。
- **USER**：UID/GID 的對映空間。Docker 預設不啟用 user namespace（本次實驗 user inode 相同），啟用後容器內的 root (uid=0) 可以對應到 host 的普通使用者，降低逃脫風險。

### Host vs 容器 inode 對照

| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | 4026531836 | 4026532772 | 否 |
| net | 4026531833 | 4026532774 | 否 |
| mnt | 4026531832 | 4026532769 | 否 |
| uts | 4026531838 | 4026532770 | 否 |
| ipc | 4026531839 | 4026532771 | 否 |
| user | 4026531837 | 4026531837 | 是 |

觀察方式：
```bash
CPID=$(sudo docker inspect -f '{{.State.Pid}}' ns-demo)
sudo ls -la /proc/$CPID/ns/   # 容器的 namespace
sudo ls -la /proc/1/ns/        # host PID 1 的 namespace
```

### 容器內 `ps aux` 輸出

```
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
    7 root      0:00 ps aux
```

容器內只看到 2 支 process，而 host 上有上百支。原因：PID namespace 把容器的 PID 空間切開，容器只能看到同一個 PID namespace 下的 process。`sleep` 在容器內是 PID 1（init 角色），但在 host 視角它是 PID 6024——同一支 process，兩個視角。

---

## Cgroups 實驗

### 容器內讀到的限制

```bash
docker exec cg-demo sh -c 'cat /sys/fs/cgroup/memory.max; echo "---"; cat /sys/fs/cgroup/cpu.max'
```

```
268435456
---
50000 100000
```

- `memory.max = 268435456` bytes = 256 MiB（對應 `--memory=256m`）
- `cpu.max = 50000 100000`：每 100 ms 最多用 50 ms CPU = 0.5 核（對應 `--cpus=0.5`）

### Host 端對照

```bash
CID=$(sudo docker inspect -f '{{.Id}}' cg-demo)
CGPATH=/sys/fs/cgroup/system.slice/docker-${CID}.scope
sudo cat $CGPATH/memory.max      # → 268435456
sudo cat $CGPATH/cpu.max         # → 50000 100000
sudo cat $CGPATH/memory.current  # → 950272（執行某一刻的實際用量）
```

| 檔案 | 容器內值 | Host 端值 |
|---|---|---|
| memory.max | 268435456 | 268435456 |
| cpu.max | 50000 100000 | 50000 100000 |
| memory.current | - | 950272（動態） |

容器內外讀到的是**同一份 kernel 資料**，`--memory=256m` 這個 flag 最終就是寫進 `/sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max` 這個檔案，kernel 看到這個值就會強制執行上限。

### OOM 故障三階段

| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m + shm-size=220m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | `Memory cgroup out of memory: Killed process (dd)` | 無 OOM |

```bash
# 故障注入（memory 限 32m，寫 200m 到 tmpfs）
sudo docker run --name oom-demo --memory=32m alpine \
    sh -c 'dd if=/dev/zero of=/dev/shm/big bs=1M count=200'
# exit code 137，OOMKilled=true

# 回復驗證（放寬到 256m，同時加大 shm-size）
sudo docker run --name oom-ok --memory=256m --shm-size=220m alpine \
    sh -c 'dd if=/dev/zero of=/dev/shm/big bs=1M count=200 && echo DONE'
# DONE，exit code 0，OOMKilled=false
```

> **exit code 137 = 128 + 9（SIGKILL）**，kernel OOM killer 直接殺掉 PID 1 的 dd process，整個容器終止。
> `/dev/shm` 是 tmpfs，寫入會計入 memory cgroup；若寫到可寫層（磁碟）則不會觸發 OOM。

---

## Image 分層

### `docker image inspect nginx:latest` layer 數量

共 **7 層**：

```
sha256:dbd35b2200dce25964b5371e8221a0b6c8638a6d86d76e2b1795b7584c5d4428
sha256:e7e3fdfd83a1a823b55f64bc384fe91d4e3d2236346b2eeeb7d0b85d23500533
sha256:63a5a76ebb78e59dfbdcb378dc16ae4990e26c182dd9db6095c1ae9288de3164
sha256:601efc4dbc723b4128936c32a3c7a8c65a6385c6582d4e289b9d5c945a2ed9e8
sha256:8107e256d8e26896a22cbbc67892da3d8294bb904781f5f59c6dd5a5bd7be86d
sha256:1836b43e2fbe05c71aa7d33ca08dbb37cef09c42379ca750634c906903115fbe
sha256:bb08c6805c7b40d9e83600c9e41af2b6bf125cea3652657be3335aa472caf5d6
```

`alpine:latest` 只有 **1 層**（極簡設計）：

```
sha256:45f3ea5848e8a25ca27718b640a21ffd8c8745d342a24e1d4ddfc8c449b0a724
```

### 兩個同源 image 共享 layer 的證據

本環境無外網，無法拉 nginx:1.27-alpine 與 nginx:1.26-alpine，以磁碟用量變化驗證共享機制：

- `/var/lib/docker/` 初始大小：**19M**
- 啟動 nginx:latest 容器後新增量極小（只多出可寫層）
- 再啟動 alpine:latest 容器後：**28M**（新增量遠小於 image 本身大小）

Layer 用內容的 sha256 定址——內容一樣就是同一層，不會重複儲存。多個容器共用同一組 lowerdir（唯讀層），只有各自的 upperdir（可寫層）是獨立的。

### `docker diff` 輸出範例與解讀

```bash
docker exec fs-demo sh -c 'echo hello > /tmp/hello.txt; rm /etc/nginx/conf.d/default.conf; echo custom > /etc/nginx/conf.d/custom.conf'
docker diff fs-demo
```

輸出：
```
C /etc
C /etc/nginx
C /etc/nginx/conf.d
A /etc/nginx/conf.d/custom.conf
D /etc/nginx/conf.d/default.conf
C /tmp
A /tmp/hello.txt
```

| 符號 | 意義 | 範例 |
|---|---|---|
| A | Added（新增） | `/tmp/hello.txt`、`custom.conf` |
| C | Changed（目錄內容有變動） | `/etc`、`/tmp` |
| D | Deleted（刪除） | `default.conf` |

這些變更都只在容器的 upperdir（可寫層），唯讀的 lowerdir 完全不動——其他使用同一 image 的容器看到的還是原版。

---

## OCI 呼叫鏈

```
docker CLI → dockerd → containerd → containerd-shim → runc → 容器 process
```

| 元件 | 角色 |
|---|---|
| **docker CLI** | 使用者介面，把指令透過 Unix socket 送給 dockerd |
| **dockerd** | 提供使用者友善的 API（build、volume、network），底層委託給 containerd |
| **containerd** | 管理映像下載、快照、容器生命週期，透過 gRPC 與 dockerd 溝通 |
| **containerd-shim** | 每個容器一支，在 runc 啟動後接住容器的 stdio 和 exit code；讓 containerd 重啟時容器不跟著死 |
| **runc** | OCI Runtime Spec 的參考實作，真正呼叫 `clone()` 建 namespace、寫 cgroup、exec 容器 process |

Process tree 驗證（本次實驗）：
```
containerd (PID 1760)
dockerd    (PID 1824)
containerd-shim-runc-v2 (PID 7578)  ← 每容器一支
  └── sleep 3600 (PID 7605)          ← 容器 process
```

### OCI Runtime Spec `config.json` 關鍵欄位

**namespaces**（告訴 runc 要建哪些 namespace）：
```json
"namespaces": [
    {"type": "mount"},
    {"type": "network"},
    {"type": "uts"},
    {"type": "pid"},
    {"type": "ipc"},
    {"type": "cgroup"}
]
```

**resources**（cgroup 限制，由 `--memory`、`--cpus` 等 flag 轉換而來）：
```json
"resources": {
    "devices": [...],
    "memory": {...},
    "cpu": {...}
}
```

一份 `config.json` 完整描述一個容器的 namespace、cgroup、rootfs、process 參數——任何符合 OCI 規範的 runtime（runc、crun、kata-runtime）看到這份 JSON 都能跑出相同的容器，這就是 Docker image 能在 Podman / K8s / containerd 上直接使用的原因。

---

## 排錯紀錄

### 案例一：docker 指令 Permission denied

- **症狀**：`docker info` 回傳 `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock`
- **診斷**：使用者 `tt` 不在 `docker` group，無法直接存取 `/var/run/docker.sock`
- **修正**：改用 `sudo docker ...`（或將使用者加入 docker group：`sudo usermod -aG docker tt`，需重新登入）
- **驗證**：`sudo docker ps` 成功列出容器

### 案例二：OOM 故障注入沒有觸發（dd 跑完沒死）

- **症狀**：`docker run --memory=32m alpine sh -c 'dd if=/dev/zero of=/tmp/big bs=1M count=200'` 跑完，exit code 0
- **診斷**：`/tmp` 在容器內是 overlay 可寫層（磁碟），不計入 memory cgroup，所以不會 OOM
- **修正**：改寫到 `/dev/shm/big`（tmpfs，計入 memory cgroup）
- **驗證**：`docker inspect oom-demo --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` 輸出 `true 137`

### 案例三：回復實驗 dd 寫 200m 到 /dev/shm 失敗（No space left on device）

- **症狀**：`docker run --memory=256m alpine sh -c 'dd ... of=/dev/shm/big count=200'` 寫到 64MB 就失敗
- **診斷**：Docker 預設 `/dev/shm` 大小為 64MB，與 `--memory` 上限無關
- **修正**：加上 `--shm-size=220m` 擴大 shm 空間
- **驗證**：容器正常印出 `DONE`，exit code 0，`OOMKilled=false`

---

## 想一想（回答 3 題）

**1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？**

不是同一支。host PID 1 是 `systemd`，容器內 PID 1 是我們跑的 `sleep 3600`——兩者在不同的 PID namespace，只是 inode 不同的獨立空間。在容器內執行 `kill -9 1` 會殺掉容器的 PID 1（即 sleep），導致整個容器退出，但 host 的 systemd 完全不受影響。Docker 建議加 `--init` 是因為容器 PID 1 若不處理 signal 和 zombie reaping，可能造成子 process 殭屍化。

**2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？**

共用。overlayfs 用 sha256 內容定址 layer，相同內容只存一份 lowerdir，兩個容器的 upperdir 才是各自獨立的。驗證方式：`sudo du -sh /var/lib/docker/` 在啟動第二個同源容器前後幾乎沒有增加（只多出一個可寫層的大小，約幾 KB）；用 `docker image inspect` 對照兩個 image 的 `RootFS.Layers` 可看到相同的 sha256。

**3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？**

不能完全稱為隔離。容器共用 host kernel，kernel 的 privilege escalation 漏洞（如 Dirty COW）可以讓容器內的惡意程式逃脫 namespace 限制，直接攻擊 host。VM 每個 Guest 有自己的 kernel，攻擊者要先突破 Hypervisor 才能影響 host，隔離邊界更深。這就是 Kata Containers / Firecracker 存在的原因：它們在每個容器底下再墊一層輕量 VM，用硬體虛擬化做隔離邊界，兼具容器的啟動速度與 VM 的強隔離。

---

## 可重跑最小命令鏈

```bash
docker info | grep -E "Storage Driver|Cgroup|Runtime"
sudo docker run -d --name chk --memory=256m alpine sleep 60
sudo docker exec chk cat /sys/fs/cgroup/memory.max
sudo docker rm -f chk
```

預期輸出：`268435456`（= 256 MiB）
