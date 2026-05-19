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
對照表請見 `namespace-table.md`

<img width="1084" height="505" alt="image" src="https://github.com/user-attachments/assets/c1490261-608b-4156-a967-f8781216e9d2" />

<img width="803" height="372" alt="image" src="https://github.com/user-attachments/assets/0c559f32-cd9d-4959-b449-1917a24c5227" />

### 容器內 `ps aux` 輸出
容器內只看到少量 process，主要是 container 自己的 process（sleep、sh、ps）。

看不到 host 上的大量 kernel process，因為 PID namespace 會把 process 視角隔離開來，container 只能看到自己的 process。

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：`268435456`
- cpu.max：`50000 100000`

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：`268435456`
- cpu.max：`50000 100000`
- memory.current（執行時某一刻）：`417792`

### OOM 故障三階段

| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | oom-kill / Out of memory / Killed process | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
大約 5～7 層

### 兩個同源 image 共享 layer 的證據
Docker image 採用 layer 機制。

若兩個 image 都基於相同的 base image（例如 alpine），前幾層 sha256 digest 會相同，代表 layer 被共享，而不是重複下載兩份。

這也是 Docker image 體積較小、建置較快的重要原因。

### `docker diff` 輸出範例與解讀

<img width="1417" height="651" alt="image" src="https://github.com/user-attachments/assets/f2dcc89e-4a2b-4539-9a91-46c27b3b27b3" />

說明 :
- A（Added）代表新增檔案，例如：
  - `/tmp/hello.txt`
  - `/etc/nginx/conf.d/custom.conf`
- C（Changed）代表原本 image 裡的目錄或檔案內容被修改，例如：
  - `/etc/nginx/conf.d`
  - `/tmp`
- D（Deleted）代表 image 原本存在的檔案被刪除，例如：
  - `/etc/nginx/conf.d/default.conf`

這代表 container 並不會直接修改原本 image 的唯讀 layer，而是把所有修改記錄在 container 自己的 writable layer 中。

## OCI 呼叫鏈

Docker 在建立 container 時，實際上不是只有單一程式在運作，而是由多個元件合作完成。

大致流程如下：

dockerd → containerd → containerd-shim → runc

- dockerd：
  Docker 的主要 daemon，負責接收使用者的 `docker run`、`docker ps` 等指令，也負責 image、network、volume 等高階管理功能。

- containerd：
  負責真正管理 container lifecycle，例如建立、啟動、停止 container，也會管理 image pull 與 snapshot。

- containerd-shim：
  負責讓 container 即使在 dockerd 重啟後也能繼續存活，並協助管理 container 的 stdin/stdout 與 process 狀態。

- runc：
  真正建立 container 的低階 runtime。它會依照 OCI Runtime Spec 的 `config.json` 去建立 namespace、cgroup、mount 等隔離環境。

OCI Runtime Spec 的 `config.json` 中：
- `namespaces` 欄位對應 PID、NET、MNT、UTS、IPC 等 namespace 隔離設定。
- `linux.resources` 對應 cgroup 的 memory、cpu、pids 等限制設定。

## 排錯紀錄
- 症狀：執行 `docker ps`、`docker run` 等指令時出現：

`permission denied while trying to connect to the Docker API socket`
  
- 診斷：目前使用者沒有加入 docker 群組，因此無法直接存取 `/var/run/docker.sock`。
- 修正：在所有 docker 指令前加上 `sudo`
- 驗證：加上 `sudo` 後，container 可以正常建立與執行，`docker diff`、`docker inspect` 等指令也能正常輸出。

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？

container 有自己的 PID namespace，所以 container 看到的 PID 1，其實只是 container 內的 init process；但在 host 看，會是另一個較大的 PID。

如果在 container 內執行：`kill -9 1`

container 的主 process 會被強制終止，container 也會直接停止。

2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？

大部分 layer 是共用的，不會完整吃兩份空間。

Docker image 採用 layer 機制，相同的 base image layer 只會儲存一份，不同 container 共享唯讀 layer，只有各自的 writable layer 會另外增加空間。

可以透過：

```
docker image inspect
docker system df
```

或比較 layer 的 sha256 digest 是否相同來驗證。

3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？

容器的隔離其實是建立在 host kernel 之上，因此如果 host kernel 本身有漏洞，container 也可能被突破。

也就是說：

container 是「共享 host kernel」的隔離。

而 VM 則是透過 Hypervisor 模擬出完整硬體，每台 VM 都有自己的 kernel，因此隔離性通常比 container 更強。

所以：

- Container：效能較好、較輕量，但隔離較弱。
- VM：隔離較完整，但資源消耗較大。
