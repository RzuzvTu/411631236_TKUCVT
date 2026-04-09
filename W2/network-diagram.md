# W02｜網路拓樸圖

```mermaid
flowchart TB
    subgraph dev["dev-a（雙網卡）"]
        nic1["enp2s0\nNAT\n192.168.103.128"]
        nic2["enp26s0\nHost-only\n192.168.213.128"]
    end

    subgraph srv["server-b（單網卡）"]
        nic3["enp2s0\nHost-only\n192.168.213.131"]
    end

    INET["Internet\n（外網）"]
    HNET["Host-only 網段\n192.168.213.0/24"]
    HOST["Host（Mac）"]

    nic1 -->|"可上網 ✅"| INET
    nic2 --> HNET
    nic3 --> HNET
    HNET <-->|"雙向互通 ✅\nping / SSH / SCP"| HOST

    nic3 -. "無法上網 ❌\n（無預設路由）" .-> INET

    style dev fill:#dbeafe,stroke:#3b82f6,color:#000000
    style srv fill:#e0e7ff,stroke:#6366f1,color:#000000
    style INET fill:#d1fae5,stroke:#10b981,color:#000000
    style HNET fill:#fef3c7,stroke:#f59e0b,color:#000000
    style HOST fill:#f3f4f6,stroke:#6b7280,color:#000000
    style nic1 color:#000000
    style nic2 color:#000000
    style nic3 color:#000000
```

## 說明

| VM | 網卡 | 模式 | IP | 可上網 |
|---|---|---|---|---|
| dev-a | enp2s0 | NAT | 192.168.103.128 | ✅ |
| dev-a | enp26s0 | Host-only | 192.168.213.128 | ❌ |
| server-b | enp2s0 | Host-only | 192.168.213.131 | ❌ |

**流量方向：**
- `dev-a enp2s0` → Internet：可通（NAT）
- `dev-a` ↔ `server-b`：可通（同屬 192.168.213.0/24 Host-only 網段）
- `server-b` → Internet：不通（無 NAT 網卡、無預設路由）
