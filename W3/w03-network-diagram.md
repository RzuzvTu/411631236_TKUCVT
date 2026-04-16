# W03｜網路拓樸圖

```mermaid
flowchart TB
    subgraph MAC["Host OS（Mac）"]
        mackey["SSH Key: mac-key"]
    end

    subgraph BASTION["bastion（跳板機）"]
        b_nat["enp2s0 NAT\n192.168.103.128"]
        b_ho["enp26s0 Host-only\n192.168.213.128"]
        b_key["SSH Key: bastion-key"]
    end

    subgraph APP["app（應用層）"]
        a_ho["enp2s0 Host-only\n192.168.213.133"]
        a_fw["ufw: deny all\nallow 22 from 192.168.213.0/24"]
    end

    subgraph DB["db（資料層）"]
        d_ho["enp2s0 Host-only\n192.168.213.132"]
        d_fw["ufw: deny all\nallow 22 from .133 + .128"]
    end

    INET["Internet"]

    b_nat -->|"上網 ✅"| INET
    MAC -->|"SSH ProxyJump ✅"| b_ho
    b_ho -->|"SSH ✅"| a_ho
    b_ho -->|"SSH ✅"| d_ho
    a_ho -->|"SSH ✅"| d_ho
    MAC -. "禁止直連 ❌" .-> a_ho
    MAC -. "禁止直連 ❌" .-> d_ho

    style BASTION fill:#fef3c7,stroke:#f59e0b,color:#000000
    style APP fill:#dbeafe,stroke:#3b82f6,color:#000000
    style DB fill:#e0e7ff,stroke:#6366f1,color:#000000
    style MAC fill:#d1fae5,stroke:#10b981,color:#000000
    style INET fill:#f3f4f6,stroke:#6b7280,color:#000000
    style b_nat color:#000000
    style b_ho color:#000000
    style b_key color:#000000
    style a_ho color:#000000
    style a_fw color:#000000
    style d_ho color:#000000
    style d_fw color:#000000
    style mackey color:#000000
```

## 說明

| VM | 角色 | 網卡 | 模式 | IP | 防火牆 |
|---|---|---|---|---|---|
| bastion | 跳板機 | enp2s0 | NAT | 192.168.103.128 | 無限制 |
| bastion | 跳板機 | enp26s0 | Host-only | 192.168.213.128 | 無限制 |
| app | 應用層 | enp2s0 | Host-only | 192.168.213.133 | allow 22 from 192.168.213.0/24 |
| db | 資料層 | enp2s0 | Host-only | 192.168.213.132 | allow 22 from .133 + .128 only |

**流量方向：**
- `bastion enp2s0` → Internet：可通（NAT）
- `Mac` → `bastion`：可通（SSH）
- `Mac` → `app` / `db`：透過 ProxyJump 跳板可通，禁止直連
- `bastion` ↔ `app`：可通（Host-only 同網段 + ufw 允許）
- `bastion` ↔ `db`：可通（Host-only 同網段 + ufw 允許）
- `app` → `db`：可通（ufw 允許 app IP）
- `app` / `db` → Internet：不通（無 NAT 網卡、無預設路由）
```
