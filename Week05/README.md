# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver: `overlayfs`
- Cgroup Driver: `systemd`
- Cgroup Version: `2`
- Default Runtime: `runc`

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：
把 process 隔離開來，容器內只能看到自己的 process，看不到 host 或其他 container 的 process。

- NET：
把網路環境隔離開來，每個 container 可以有自己的 IP、port、routing 與網卡設定。

- MNT：
把檔案系統掛載點隔離開來，container 看到的是自己的 filesystem，不會直接看到 host 的完整目錄。

- UTS：
隔離 hostname 與 domain name，container 可以有自己的 hostname。

- IPC：
隔離 process 之間的 IPC（Inter-Process Communication）機制，例如 shared memory、message queue 等。

- USER：
隔離使用者與群組 ID，可以讓 container 內的 root 不一定等於 host 的 root。

### Host vs 容器 inode 對照

<img width="1084" height="505" alt="image" src="https://github.com/user-attachments/assets/c1490261-608b-4156-a967-f8781216e9d2" />

<img width="803" height="372" alt="image" src="https://github.com/user-attachments/assets/0c559f32-cd9d-4959-b449-1917a24c5227" />

### 容器內 `ps aux` 輸出
（只看到幾支 process？為什麼？）

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：
- cpu.max：

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：
- cpu.max：
- memory.current（執行時某一刻）：

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | （填入）| （填入）|
| OOMKilled | - | （填入）| （填入）|
| dmesg 關鍵字 | 無 OOM | （填入）| 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
（填入）

### 兩個同源 image 共享 layer 的證據
（前幾個 sha256 是否相同？）

### `docker diff` 輸出範例與解讀
（貼上 A/C/D 實例並說明）

## OCI 呼叫鏈

（用自己的話說明 dockerd → containerd → containerd-shim → runc 各自負責什麼，以及 OCI Runtime Spec `config.json` 裡哪些欄位對應到 namespace / cgroup 設定）

## 排錯紀錄
- 症狀：
- 診斷：
- 修正：
- 驗證：

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？
2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？
3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？
