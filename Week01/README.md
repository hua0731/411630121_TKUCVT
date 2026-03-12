# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：Ubuntu 64-bit (2)
- Ubuntu 版本：
```
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.4 LTS
Release:	24.04
Codename:	noble
```
- Docker 版本： `Docker version 29.3.0, build 5927d80` 
- Docker Compose 版本：`Docker Compose version v5.1.0
` 

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s):   4|
| 記憶體 | 4 GB | `free -h \| grep Mem` | Mem:           3.8Gi       1.3Gi       180Mi        34Mi       2.5Gi       2.4Gi |
| 磁碟 | 40 GB | `df -h /` | /dev/sda2        49G   11G   36G   23% /|
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor 供應商：     VMware |

## 四層驗收證據
- [ ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg]   https://download.docker.com/linux/ubuntu   noble stable
```

- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出
```
ii  docker-ce                                      5:29.3.0-1~ubuntu.24.04~noble            amd64        Docker: the open-source application container engine
ii  docker-ce-cli                                  5:29.3.0-1~ubuntu.24.04~noble            amd64        Docker CLI: the open-source application container engine
ii  docker-ce-rootless-extras                      5:29.3.0-1~ubuntu.24.04~noble            amd64        Rootless support for Docker.
```

- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
<img width="868" height="625" alt="螢幕擷取畫面 2026-03-12 143135" src="https://github.com/user-attachments/assets/55f19a80-e87b-4ca7-8bf4-4145c034383d" />

- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

- [ ] Compose：`docker compose version` 可執行
```
Docker Compose version v5.1.0
```

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
```
ba1797d533be3b43887c680815f4abea4a222273646502a79a1d2f61213b76b7
```
```
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
```

- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
```
/ # 
```

- [ ] 映像列表：`sudo docker images` 輸出
```
i Info →   U  In Use
 
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:latest        25109184c71b       13.1MB         3.95MB        
hello-world:latest   85404b3c5395       25.9kB         9.52kB    U   
nginx:latest         bc45d248c4e1        240MB         65.8MB    U   
```

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | Docker 安裝完成後 | 建立第一個環境健康的可回復基線 | hostnamectl、ip route、docker --version、docker compose version、systemctl status docker、docker run --rm hello-world |
| docker-ready | 完成 container 測試後 | Docker 環境與 container 功能驗證完成，之後可以繼續做 | sudo systemctl status、docker --no-pager、sudo docker run --rm hello-world、sudo docker images  |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是|
| hello-world 成功 | 是 | N/A | 是|
| nginx curl 成功 | 是 | N/A | 是|

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 幾秒而已(大概30)，因為上課練習的錯誤很容易修復，但如果是很難的錯誤應該會用很久 | 一樣幾秒而已(大概10-15)，因為就是關機然後選snapshot再開機而已 |
| 適用情境 | 知道問題原因且是很容易修復的錯誤時| 系統嚴重損壞或不確定問題來源時 |
| 風險 | 若操作錯誤可能造成更多問題 | 會遺失snapshot之後所有改變 |

## Snapshot 保留策略
- 新增條件：在系統環境健康或完成重要設定後建立，例如完成docker安裝並確認服務正常
- 保留上限：3–5個snapshot，避免磁碟空間被大量占用
- 刪除條件：當新的健康snapshot建立後，可刪除較舊且不再需要回復的snapshot

## 最小可重現命令鏈
```
# 故障前基線
echo "=== 故障前 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10

# 注入故障
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

echo "=== 故障中 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10
sudo apt -y install docker-ce 2>&1 | tail -5

sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
sudo apt update
apt-cache policy docker-ce | head -5

sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

sudo poweroff

echo "=== 回復後 ==="
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
```

## 排錯紀錄
- 症狀：執行 `apt-cache policy docker-ce` 時，只顯示 `/var/lib/dpkg/status`，Docker repository 消失
- 診斷：（你首先查了什麼？）檢查 `/etc/apt/sources.list.d/` 發現 `docker.list` 被改名為 `docker.list.broken`，APT因副檔名不正確而忽略repository
- 修正：（做了什麼改動？）將 `docker.list.broken` 改回 `docker.list` 並重新執行 `apt update`
- 驗證：（如何確認修正有效？）重新執行 `apt-cache policy docker-ce`，確認再次出現 `https://download.docker.com/linux/ubuntu noble/stable`

## 設計決策
這次實驗使用修改repository檔名作為故障注入方式，因為這個方法不會破壞docker已安裝的套件，只會影響APT套件來源，因此能安全的模擬repository設定錯誤的情境。
而且老師說這個錯誤很容易回復，只需將檔名改回原本名稱即可恢復系統狀態。
