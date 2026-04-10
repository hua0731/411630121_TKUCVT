# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統層級設定檔目錄，用於存放服務的設定 | 存放 Docker daemon |
| /var/lib/docker/ | 可變動資料目錄，用於存放應用程式執行時產生的資料 | 儲存 image、container、volume 等實際資料 |
| /usr/bin/docker | 使用者可執行的二進位檔位置 | Docker CLI 指令所在 |
| /run/docker.sock | 系統執行期間的暫存通訊檔（Unix socket） | Docker client 與 daemon 溝通的通道 |

## Docker 系統資訊

- Storage Driver：`overlayfs`
- Docker Root Dir：`/var/lib/docker`
- 拉取映像前 /var/lib/docker/ 大小：320K
- 拉取映像後 /var/lib/docker/ 大小：324K

## 權限結構

### Docker Socket 權限解讀
<img width="740" height="293" alt="螢幕擷取畫面 2026-04-10 204939" src="https://github.com/user-attachments/assets/0833a85e-d54c-4a98-b3d5-3cd64cbfd152" />

- s：表示為 socket 檔案
- rw-（owner）：擁有者可讀寫
- rw-（group）：群組可讀寫
- ---（others）：其他人無權限

### 使用者群組
<img width="1280" height="159" alt="image" src="https://github.com/user-attachments/assets/597206a9-9cef-4ed7-9d0b-9d5f5c7a2595" />

目前使用者未包含於 docker 群組，因此需使用 sudo 執行 Docker 指令。
若加入 docker 群組，則可直接操作 Docker，但也代表具有類似 root 權限。

<img width="1400" height="155" alt="image" src="https://github.com/user-attachments/assets/8f8fc14f-7d36-41cc-ac56-fda891cc4d71" />

群組變更新的 session ，groups 現在包含 docker，docker ps 不加 sudo 也能執行。

### 安全意涵
docker group 之所以可以視為接近 root 權限，是因為加入這個群組的使用者可以直接透過 Docker 操作系統資源，例如建立 container、掛載主機目錄等。

在本次安全示範中，透過執行 docker run 並將主機的 /etc/shadow 掛載到 container 中，可以直接讀取主機上的密碼雜湊資料。這代表即使沒有使用 sudo，只要能操作 Docker，就等同可以存取系統的敏感檔案。

因此 docker group 的權限實際上非常高，幾乎等同於 root。在實務環境中，不應隨意將使用者加入 docker 群組，以避免系統安全風險。

## 程序與服務管理

### systemctl status docker
<img width="1591" height="622" alt="image" src="https://github.com/user-attachments/assets/5df21fe3-76c1-4383-8dcb-2eeeccc33bf5" />

### journalctl 日誌分析
```
hua0731@bastion:~$ journalctl -u docker --since "1 hour ago" --no-pager | tail -30
 4月 10 20:34:35 bastion systemd[1]: Starting docker.service - Docker Application Container Engine...
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.192914362+08:00" level=info msg="Starting up"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.193685860+08:00" level=info msg="OTEL tracing is not configured, using no-op tracer provider"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.193795768+08:00" level=info msg="CDI directory does not exist, skipping" dir=/var/run/cdi
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.193819069+08:00" level=info msg="CDI directory does not exist, skipping" dir=/etc/cdi
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.194043910+08:00" level=info msg="detected 127.0.0.53 nameserver, assuming systemd-resolved, so using resolv.conf: /run/systemd/resolve/resolv.conf"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.340296200+08:00" level=info msg="Creating a containerd client" address=/run/containerd/containerd.sock timeout=1m0s
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.349808015+08:00" level=info msg="Loading containers: start."
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.353155036+08:00" level=info msg="NRI is disabled"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.353181266+08:00" level=info msg="Starting daemon with containerd snapshotter integration enabled"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.361497483+08:00" level=info msg="Restoring containers: start."
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.440917793+08:00" level=info msg="Deleting nftables IPv4 rules" error="exit status 1" output="Error: Could not process rule: No such file or directory\ndelete table ip docker-bridges"
 4月 10 20:34:36 bastion dockerd[1723]: time="2026-04-10T20:34:36.460380630+08:00" level=info msg="Deleting nftables IPv6 rules" error="exit status 1" output="Error: Could not process rule: No such file or directory\ndelete table ip6 docker-bridges"
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.017166065+08:00" level=info msg="Loading containers: done."
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.036688375+08:00" level=info msg="Docker daemon" commit=83bca51 containerd-snapshotter=true storage-driver=overlayfs version=29.3.0
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.037193956+08:00" level=info msg="Initializing buildkit"
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.064731297+08:00" level=info msg="Completed buildkit initialization"
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.069317305+08:00" level=info msg="Daemon has completed initialization"
 4月 10 20:34:37 bastion dockerd[1723]: time="2026-04-10T20:34:37.069410994+08:00" level=info msg="API listen on /run/docker.sock"
 4月 10 20:34:37 bastion systemd[1]: Started docker.service - Docker Application Container Engine.
 4月 10 20:42:48 bastion dockerd[1723]: time="2026-04-10T20:42:48.204767579+08:00" level=info msg="image pulled" digest="sha256:7f0adca1fc6c29c8dc49a2e90037a10ba20dc266baaed0988e9fb4d0d8b85ba0" remote="docker.io/library/nginx:latest"
 4月 10 21:12:24 bastion dockerd[1723]: time="2026-04-10T21:12:24.599056774+08:00" level=info msg="sbJoin: gwep4 ''->'fa32df944725', gwep6 ''->''" eid=fa32df944725 ep=funny_wing net=bridge nid=998e96c8b3a6
 4月 10 21:12:24 bastion dockerd[1723]: time="2026-04-10T21:12:24.697643023+08:00" level=info msg="received task-delete event from containerd" container=e45ad24431d578a4a5f4c18e05a9531bbbd058a37f298b7beee4d8dc5fbd8844 module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
 4月 10 21:12:38 bastion dockerd[1723]: time="2026-04-10T21:12:38.276135763+08:00" level=info msg="sbJoin: gwep4 ''->'387540ea1d70', gwep6 ''->''" eid=387540ea1d70 ep=elastic_benz net=bridge nid=998e96c8b3a6
 4月 10 21:12:38 bastion dockerd[1723]: time="2026-04-10T21:12:38.327632340+08:00" level=info msg="received task-delete event from containerd" container=22c6954261d1d350634b10a7f320128e1b8d0bf5adddd733df484f2466912cd4 module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
```
1. Docker 服務啟動

- `Starting docker.service - Docker Application Container Engine...`
- `Starting up`
- `Started docker.service - Docker Application Container Engine.`

這表示 Docker 服務在 20:34 左右開始啟動，並成功由 systemd 啟動完成。

2. Docker daemon 初始化過程

- `Creating a containerd client`
- `Loading containers: start.`
- `Restoring containers: start.`
- `Loading containers: done.`
- `Docker daemon ... storage-driver=overlayfs version=29.3.0`
- `Initializing buildkit`
- `Completed buildkit initialization`
- `Daemon has completed initialization`

這些訊息顯示 Docker daemon 啟動時，會先連接 containerd、載入既有 container 狀態、確認 storage driver，最後完成 buildkit 與 daemon 初始化。

3. Socket 監聽

- `API listen on /run/docker.sock`

表示 Docker daemon 已開始監聽 /run/docker.sock，代表 Docker CLI 可以透過此 socket 與 daemon 溝通。

4. 映像檔拉取事件

- `image pulled ... remote="docker.io/library/nginx:latest"`

表示在 20:42 成功從 Docker Hub 拉取 nginx:latest 映像檔。

5. Container 網路與刪除事件

- `sbJoin ... net=bridge`
- `received task-delete event from containerd`

這代表之後有 container 加入 bridge 網路，並且有 container 執行結束後被刪除。可推測當時有執行過短生命週期的 container 測試。

6. 啟動過程中的 nftables 訊息

- `Deleting nftables IPv4 rules`
- `Deleting nftables IPv6 rules`

這些訊息顯示 Docker 啟動時有嘗試處理既有的 nftables 規則，雖然顯示找不到舊規則，但後續 daemon 仍成功完成初始化，因此未影響 Docker 正常啟動。

### CLI vs Daemon 差異

Docker CLI（Command Line Interface）是使用者操作 Docker 的工具，例如輸入 `docker run`、`docker ps` 等指令時，其實是在呼叫 CLI 程式。

Docker Daemon 則是實際在背景執行的服務（dockerd），負責建立 container、管理 image、處理網路與儲存等功能。CLI 本身不會執行這些動作，而是透過 `/run/docker.sock` 與 daemon 溝通，由 daemon 來完成操作。

因此，`docker --version` 只是在確認 CLI 程式是否存在與版本資訊，即使 daemon 沒有啟動，這個指令仍然可以正常執行。但當執行像 `docker run` 或 `docker ps` 這類需要實際操作 Docker 的指令時，就必須能成功連到 daemon，否則會出現無法連線或 permission denied 的錯誤。

## 環境變數

- $PATH：`PATH:  /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin`
- which docker：`/usr/bin/docker`
- 容器內外環境變數差異觀察：
<img width="1147" height="335" alt="image" src="https://github.com/user-attachments/assets/be51e494-8c15-4a98-8c7e-c1ab9fac8b33" />

差異說明
1. PATH 不同
- Host 多了 `/usr/games`、`/snap/bin` 等路徑
- Container 的 PATH 較精簡，只保留基本系統指令路徑

2. 使用者不同
- Host：`USER=hua0731`
- Container：預設為 `/root`

3. HOME 不同
- Host：`/home/hua0731`
- Container：`/root`

4. Container 特有變數
- `HOSTNAME` 為 container ID，用來識別容器

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive (dead) | active (running) |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux \| grep dockerd | 有 process | 無 process | 有 process |


## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |


## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | Docker daemon 未啟動或無法連線 | 檢查 systemctl status docker、dockerd process |
| permission denied…docker.sock | 使用者沒有存取 socket 權限 | 檢查 /var/run/docker.sock 權限與 docker 群組 |

- 差異說明

  - Cannot connect 表示 Docker daemon 沒有在運作，或 client 無法連到 daemon，屬於服務層問題（daemon 沒開）。  
  - permission denied 則表示 daemon 是正常運作的，但使用者沒有權限操作 socket，屬於權限問題。


## 排錯紀錄

- 症狀：
  執行 docker ps 時出現 Cannot connect 或 permission denied 錯誤。

- 診斷：
  先使用 `systemctl status docker` 確認 Docker 服務是否啟動，並用 `ps aux | grep dockerd` 檢查 daemon process。若服務正常，則進一步檢查 `/var/run/docker.sock` 權限與使用者群組。

- 修正：
  若 daemon 未啟動，使用 `sudo systemctl start docker` 啟動服務；  
  若為權限問題，則修正 socket 權限或將使用者加入 docker 群組。

- 驗證：
  再次執行 `docker ps`，確認可以正常列出 container，且不再出現錯誤訊息。

---

## 設計決策

本週實驗中，選擇使用 `usermod -aG docker` 將使用者加入 docker 群組，使其可不使用 sudo 執行 Docker 指令，提升操作便利性。

但此設計的風險在於 docker 群組擁有接近 root 的權限，使用者可以透過 container 存取主機檔案（如 /etc/shadow），因此在實務環境中應謹慎控管 docker 群組成員，避免安全風險。
