# vm-vs-container 作業

## 至少四個維度的 VM vs Container 對照
VM 和 Container 都提供「隔離的執行環境」，但隔離的層級不同：

- **VM 虛擬的是硬體**
  - 每台 VM 都包含自己的 Guest OS
  - 隔離完整，但資源消耗較大

- **Container 虛擬的是作業系統資源**
  - 所有 container 共用 Host OS 的 kernel
  - 啟動速度快，資源需求較低

| 維度 | 虛擬機器（VM） | 容器（Container） |
|---|---|---|
| 隔離層 | 完整 Guest OS + Hypervisor | 共用 Host OS kernel |
| 啟動速度 | 數分鐘（需啟動整個 OS） | 數秒（只啟動程序） |
| 資源佔用 | 較重，每台 VM 都需要完整 OS | 較輕，可在同一主機跑大量 container |
| 封裝內容 | OS + 應用程式 + 設定 | 應用程式 + 相依套件 |
| 映像大小 | 通常數 GB | 通常數 MB ~ 數百 MB |
| 回復方式 | Snapshot 還原 | 重新拉取映像 |

## 本課選擇「VM 裡跑 Docker」的理由（用自己的話寫）

我覺得課程用「VM裡跑Docker」主要是為了讓大家的實驗環境比較一致，因為每個人的電腦系統可能不一樣Ex.Windows、macOS，如果直接在自己的主機裝Docker，有時候設定或版本會有差異。
所以先在VM裡裝Ubuntu，再在裡面跑Docker，就可以讓大家用幾乎一樣的環境做實驗，比較不容易出現「我這台可以你那台不行」的情況。
另外一個原因是安全性，如果直接在自己的電腦操作，有些錯誤的設定可能會影響到本機系統，但如果把Docker放在VM裡面跑，就算實驗過程中把系統弄壞了，也只會影響那台VM，不會傷到自己的電腦。

## Hypervisor Type 1 vs Type 2 的差異與本課的選擇

Hypervisor 是負責建立與管理 VM 的軟體。

### Type 1 Hypervisor（Bare-metal）

- 直接安裝在實體硬體上
- 不需要 Host OS
- 效能較好

範例：

- VMware ESXi
- Microsoft Hyper-V
- Xen

適用場景：

- 企業資料中心
- 雲端平台

---

### Type 2 Hypervisor（Hosted）

- 安裝在作業系統上
- 像一般應用程式一樣執行

範例：

- VMware Workstation
- VirtualBox
- VMware Fusion
- Parallels

適用場景：

- 個人開發
- 教學環境
- 本機測試

---

### 本課選擇 Type 2 的原因

本課使用 **VMware Workstation (Type 2 Hypervisor)**，因為：

1. 可以在學生筆電上直接安裝
2. 不需要企業級伺服器設備
3. 適合教學與實驗用途
4. 支援 Snapshot 快速回復
