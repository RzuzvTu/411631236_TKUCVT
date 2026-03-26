## VM vs Container 對照表（四個維度）


| 維度 | 虛擬機 (VM) | 容器 (Container / Docker) |
|---|---|---|
| 核心架構 | 硬體層級虛擬化：包含完整的作業系統 (Guest OS)。 | 作業系統層級虛擬化：共享宿主機的核心 (Kernel)。 |
| 啟動速度 | 較慢：需要數分鐘（因為要跑完完整的開機流程）。 | 極快：秒級啟動（只需開啟一個處理程序）。 |
| 資源消耗 | 較高：每個 VM 都要預扣固定的記憶體與硬碟空間。 |極低：僅佔用執行程式所需的資源，空間極小。 |
| 隔離性 | 強：具有完全獨立的核心，安全性較高。 | 中：共享核心，若核心受攻擊可能影響所有容器。 |

## 本課選擇「VM 裡跑 Docker」的理由

「我的理由是：為了兼顧『環境隔離』與『快速復原』。」

雖然 Docker 已經很方便，但直接在 Mac 上跑 Docker Desktop 有時候會污染宿主機環境。選擇在 VM 裡跑 Docker，最大的好處是具備 「雙重保險」：

快照 (Snapshot) 功能： 像我剛才實驗弄壞了 Docker 軟體源，只要有點快照，幾秒鐘就能還原整個 Linux 環境，這在實體機或單純 Docker 上很難做到這麼徹底。

環境一致性： 在 VM 裡建立的 Ubuntu 環境，其操作邏輯（如 apt 管理、systemd 服務）與雲端伺服器完全一致，這讓我們能提早練習標準的 Linux 維運技能。

## Hypervisor Type 1 vs Type 2 的差異與本課的選擇

Type 1 (Bare Metal / 原生型)：

定義： 直接安裝在硬體上（裸機），不需要先裝作業系統。

代表： VMware ESXi、Microsoft Hyper-V、Proxmox (KVM)。

優點： 效能損耗極低，適合企業級伺服器。

Type 2 (Hosted / 寄居型)：

定義： 安裝在既有的作業系統（如 macOS, Windows）之上的一個應用程式。

代表： VMware Fusion (我們用的)、Oracle VirtualBox、UTM。

優點： 設定簡單，方便我們一邊看網頁、一邊在 Mac 上操作實驗。

## 我們選擇的是 Type 2 Hypervisor (VMware Fusion)。
理由： 因為我們需要在一台電腦（Mac）上同時處理多種任務（寫報告、查資料、跑實驗）。Type 2 讓我們能在不重啟電腦的情況下，像開 App 一樣開啟 Linux 系統，這對教學與開發初期的實驗是最有效率的選擇。