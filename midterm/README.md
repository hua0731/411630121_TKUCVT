# 期中實作 — <411630121> <曾若嬅>

## 1. 架構與 IP 表

- IP 表:

| VM | 網卡 | 模式 | IP |
|---|---|---|---|
| bastion | ens33 | NAT | 192.168.220.129 |
| bastion | ens37 | Host-only | 192.168.17.128 |
| app | ens33 | Host-only | 192.168.17.129 |

- 網路拓樸圖:

<img width="561" height="361" alt="network-diagram" src="https://github.com/user-attachments/assets/d1a3b320-54be-4f67-bf8a-02d471e4be64" />

## 2. Part A：VM 與網路

- 從 bastion ping app `ping -c 3 192.168.17.129`

<img width="653" height="232" alt="螢幕擷取畫面 2026-04-16 164455" src="https://github.com/user-attachments/assets/ade85eab-ff65-42b6-8547-cf1cafad7fff" />

- 從 app ping bastion `ping -c 3 192.168.17.128`

<img width="647" height="233" alt="螢幕擷取畫面 2026-04-16 164559" src="https://github.com/user-attachments/assets/950807ec-45a5-4473-b0da-fa5f24c8ef59" />

## 3. Part B：金鑰、ufw、ProxyJump

- 防火牆規則表:

<img width="628" height="497" alt="螢幕擷取畫面 2026-04-16 174509" src="https://github.com/user-attachments/assets/fb0b78ee-41aa-4028-811f-12fd0ea60de8" />

<img width="626" height="534" alt="image" src="https://github.com/user-attachments/assets/4bb547d2-8a7d-42f6-8987-f50bb07d4ecc" />

- `ssh app` 成功截圖:

<img width="1451" height="520" alt="image" src="https://github.com/user-attachments/assets/ca04be7f-e0b3-452a-a4e0-637551775a02" />

## 4. Part C：Docker 服務

- systemctl status docker 輸出:

<img width="1595" height="618" alt="image" src="https://github.com/user-attachments/assets/d11a38ef-b064-4a63-a8d5-e479570bae00" />

- curl 輸出:

<img width="540" height="259" alt="image" src="https://github.com/user-attachments/assets/b4a44c33-b5ba-4bd3-9d9f-ce3b7705a2bf" />

## 5. Part D：故障演練
### 故障 1：F2：防火牆封鎖 SSH
- 注入方式：在 app 上移除允許 bastion 存取 SSH 的規則：

| 階段 | 命令 | 關鍵輸出 / 現象 |
|---|---|---|
| 故障前 | `ssh app "hostname"` | `app` |
| 故障前 | `sudo ufw status verbose` | `22/tcp ALLOW IN 192.168.17.128` |
| 故障中 | `sudo ufw delete allow from 192.168.17.128 to any port 22 proto tcp` | SSH 規則被刪除 |
| 故障中 | `ssh app` | `Connection timed out` |
| 故障中 | `ping 192.168.17.129` | 仍可 ping 通 |
| 回復後 | `sudo ufw allow from 192.168.17.128 to any port 22 proto tcp` | 規則恢復 |
| 回復後 | `ssh app "hostname"` | `app` |

- 診斷推論：雖然 SSH 連線 timeout，但 ping 仍然成功，表示網路連通性正常（L3 沒問題），因此可以排除網卡或路由問題。
SSH 無法連線的原因為防火牆阻擋 22 port，因此判斷為防火牆設定問題，而非網路層問題。

### 故障 2：F3：停止 Docker daemon
- 注入方式：在 app 上停止 Docker 服務：

| 階段 | 命令 | 關鍵輸出 / 現象 |
|---|---|---|
| 故障前 | `systemctl status docker --no-pager` | `Active: active (running)` |
| 故障前 | `sudo docker ps` | 正常列出 container |
| 故障前 | `ssh app "hostname"` | `app` |
| 故障中 | `sudo systemctl stop docker` | Docker 停止 |
| 故障中 | `systemctl status docker --no-pager` | `Active: inactive (dead)` |
| 故障中 | `sudo docker ps` | `Cannot connect to the Docker daemon` |
| 故障中 | `ssh app "hostname"` | `app` |
| 回復後 | `sudo systemctl start docker` | Docker 重新啟動 |
| 回復後 | `systemctl status docker --no-pager` | `Active: active (running)` |
| 回復後 | `sudo docker ps` | 恢復正常 |

- 診斷推論：在故障中仍可透過 SSH 登入 app，表示網路與 SSH 服務正常。
但 docker 指令無法使用，且顯示無法連線到 Docker daemon，表示問題出在 Docker service，而不是整台主機或網路。
因此可判斷為應用服務（Docker daemon）層級的故障，而非系統或網路問題。

### 症狀辨識（若選 F1+F2 必答）
在 F2 中，Host 端看到的症狀是 `ssh timeout`，但同時 `ping` 仍然能通，代表 app 的網路介面與 L3 連通性正常，因此問題不是網卡或路由，而是 SSH 流量被防火牆阻擋。

在 F3 中，Host 端仍可正常 SSH 到 app，表示網路與 SSH 服務都正常；但在 app 上執行 `docker ps` 出現 `Cannot connect to the Docker daemon`，因此可以判斷故障點在 Docker daemon，而不是整台主機或網路。

## 6. 反思（200 字）
這次做完最大的感受是，以前看到 timeout 都會直覺覺得「壞掉了」，但其實不一定是整個系統掛掉，有可能只是某一層出問題。像這次在做防火牆那題時，SSH 連不上會 timeout，但其實 ping 還是通的，才發現原來網路本身是正常的，只是被防火牆擋住而已。這讓我比較能理解「分層」的概念，不是只看一個現象就下結論，而是要用不同工具去確認是哪一層出問題。

另外像 Docker 那題，雖然服務停掉了，但 SSH 還是可以連，代表主機本身沒壞，只是某個 service 出問題。這也讓我學到排錯時應該一步一步拆開來看，而不是一開始就亂猜。整體來說，這次練習讓我比較有系統地去理解問題發生在哪一層，也比較不會被 timeout 這種訊息誤導。

## 7. Bonus（選做）
