# Namespace 對照表

| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | 4026531836 | 4026532772 | 否 |
| net | 4026531833 | 4026532774 | 否 |
| mnt | 4026531832 | 4026532769 | 否 |
| uts | 4026531838 | 4026532770 | 否 |
| ipc | 4026531839 | 4026532771 | 否 |
| user | 4026531837 | 4026531837 | 是 |

> 備註：Docker 預設不啟用 user namespace，所以 user inode 相同。
> pid / net / mnt / uts / ipc 全部不同，代表這五種資源視角已完全隔離。
