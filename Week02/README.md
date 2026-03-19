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
- [ ] server-b 不能上網：`ping 8.8.8.8` 失敗輸出

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | （填入） |
| dev-a ping server-b | 成功 | 失敗 | （填入） |
| dev-a SSH server-b | 成功 | 失敗 | （填入） |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （填入） |
| dev-a ping server-b | 成功 | 成功 | （填入） |
| dev-a SSH server-b | 成功 | Connection refused | （填入） |

## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）

## 網路拓樸圖
（嵌入或連結 network-diagram.png）

## 排錯紀錄
- 症狀：
- 診斷：（你首先查了什麼？用了哪個命令？）
- 修正：（做了什麼改動？）
- 驗證：（如何確認修正有效？）

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 server-b 只設 Host-only 不給 NAT？）
