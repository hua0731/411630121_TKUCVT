# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.220.129 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.17.128 | 內網互連 |
| server-b | NIC 1 | Host-only | 192.168.17.129 | 內網互連 |

## 連線驗證紀錄

- [ ] dev-a NAT 可上網：`ping google.com` 輸出
<img width="935" height="258" alt="螢幕擷取畫面 2026-03-19 112934" src="https://github.com/user-attachments/assets/e81b3744-b9d7-4549-9ba2-46b7d0762177" />

- [ ] 雙向互 ping 成功：貼上雙方 `ping` 輸出
<img width="702" height="257" alt="螢幕擷取畫面 2026-03-19 113432" src="https://github.com/user-attachments/assets/c3f260d3-8b09-4c87-b432-eb9de411a212" />
<img width="658" height="266" alt="螢幕擷取畫面 2026-03-19 113721" src="https://github.com/user-attachments/assets/54eeefc3-58cf-4e72-abb2-c3ae64846561" />

- [ ] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
<img width="1018" height="519" alt="螢幕擷取畫面 2026-03-19 114642" src="https://github.com/user-attachments/assets/a25f65ca-1e10-4f97-8c91-e843831d308f" />

- [ ] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
<img width="762" height="186" alt="螢幕擷取畫面 2026-03-23 135941" src="https://github.com/user-attachments/assets/8faf49b1-2cbc-4a05-935c-45f4f7528a2d" />

- [ ] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
<img width="848" height="134" alt="螢幕擷取畫面 2026-03-23 140247" src="https://github.com/user-attachments/assets/f965efe2-49b0-4d96-bff2-2cd2eae32660" />

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | UP |
| dev-a ping server-b | 成功 | 失敗 | 成功 |
| dev-a SSH server-b | 成功 | 失敗 | 成功 |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| dev-a ping server-b | 成功 | 成功 | 成功 |
| dev-a SSH server-b | 成功 | Connection refused | 成功 |

## 排錯順序

L2：
先檢查網路介面是否正常啟用
使用指令：
`ip link`

L3：
確認 IP 位址是否正確，並測試網路連通性
使用指令：
`ip address show`
`ping -c 4 192.168.17.129`

L4：
檢查 SSH 服務是否啟動，以及 port 22 是否有監聽。
使用指令：
`ss -tlnp | grep :22`
`sudo systemctl status ssh`

流程簡短說明：
我選擇由下而上進行排錯，先確認網卡狀態，再確認 IP 與連通性，最後檢查服務與連接埠，以避免在高層問題時忽略底層錯誤

## 網路拓樸圖
<img width="491" height="602" alt="network-diagram" src="https://github.com/user-attachments/assets/3bcdd73c-3382-446b-a58e-743064e55c22" />

## 排錯紀錄

- 症狀：
  在進行步驟 7 時，server-b 執行 `sudo apt update` 會一直停在連線畫面，顯示無法正常下載套件。另外，在初期設定時，也需要確認 dev-a 與 server-b 是否能透過 Host-only 網路互相連線，以及 SSH 是否正常運作

- 診斷：（你首先查了什麼？用了哪個命令？）
  我先使用 `ip address show` 檢查兩台 VM 的 IP 位址與網卡狀態，確認 dev-a 具有 NAT 與 Host-only 兩張網卡，而 server-b 只有 Host-only。之後使用 `ping -c 4 192.168.17.129` 與 `ping -c 4 192.168.17.128` 測試雙方 Host-only 網路是否可互通。  
  在 SSH 連線部分，我先使用 `sudo systemctl status ssh --no-pager` 檢查 server-b 的 SSH 服務狀態，再使用 `ss -tlnp | grep :22` 確認 port 22 是否有在監聽。  
  針對 `apt update` 卡住的問題，從 IP 配置判斷出 server-b 只有 Host-only 網卡，因此無法直接連上外部網路

- 修正：（做了什麼改動？）
  我先確認並啟動 server-b 的 SSH 服務，使用 `sudo systemctl enable ssh` 設定開機自動啟動，並以 `sudo systemctl start ssh` 讓服務立即啟動。  
  之後在 dev-a 上使用 `ssh hua0731@192.168.17.129` 成功連入 server-b，確認 SSH 設定正確。  
  對於套件安裝問題，因為 server-b 只有 Host-only 無法上網，所以需暫時切換成 NAT 或額外增加 NAT 網卡，讓系統可執行 `sudo apt update` 與安裝套件；完成後再切回 Host-only，以符合實驗設計

- 驗證：（如何確認修正有效？）
  我使用 `ping -c 4 192.168.17.129` 驗證 dev-a 可連到 server-b，並以 `ssh hua0731@192.168.17.129 "hostname && ip address show && uptime"` 驗證遠端 SSH 連線、主機名稱、IP 位址與系統運作狀態皆正常。  
  此外，我也使用 `scp /tmp/test-from-dev.txt hua0731@192.168.17.129:/tmp/` 傳送檔案，再用 `ssh hua0731@192.168.17.129 "cat /tmp/test-from-dev.txt"` 驗證檔案已成功傳送到 server-b。這些結果都表示修正後系統可正常運作

## 設計決策

本週的技術選擇中，我讓 dev-a 同時配置 NAT 與 Host-only 兩張網卡，而 server-b 只配置 Host-only。

dev-a 需要 NAT，因為它扮演管理端與工作端的角色，必須能連上外部網路下載套件、更新系統，並同時透過 Host-only 與內部主機溝通。server-b 只保留 Host-only，目的是模擬內部伺服器環境，使其只能被內網中的 dev-a 存取，而不能直接連上 Internet。

這種配置的優點是安全性較高，因為 server-b 不會直接暴露在外部網路上，較符合伺服器僅開放內部管理的情境；缺點是若 server-b 需要安裝套件或更新系統，就不能直接執行 `apt update`，必須暫時切換成 NAT 或額外增加 NAT 網卡。因此這是一種在「安全性」與「操作方便性」之間的取捨。本實驗選擇讓 server-b 僅使用 Host-only，主要是為了更清楚呈現內外網分離的概念，也方便觀察不同網路模式對連線行為的影響。
