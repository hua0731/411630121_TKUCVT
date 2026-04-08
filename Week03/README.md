# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.220.129 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.17.128 | — |
| app | 應用層 | NIC 1 | Host-only | 192.168.17.129 | SSH from 192.168.17.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.17.130 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：ed25519
- 公鑰部署到：app 和 db 的 ~/.ssh/authorized_keys
<img width="1185" height="240" alt="螢幕擷取畫面 2026-04-08 104626" src="https://github.com/user-attachments/assets/d3d791a7-1225-48ab-9b13-dd49e528656e" />
<img width="1167" height="232" alt="螢幕擷取畫面 2026-04-08 104856" src="https://github.com/user-attachments/assets/0c5816e7-9f0b-48ed-b4fd-2338bfa752e9" />

- 免密碼登入驗證：
  - bastion → app：（貼上輸出）
```
hua0731@bastion:~$ ssh hua0731@192.168.17.129
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-19-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

對 Applications 延伸的安全維護未啟用

49 個更新可立即套用。
26 個標準安全更新。
要檢查這些額外的更新，請執行 apt list --upgradable

7 的安全更新可以 ESM Apps 來套用。關於啟用 ESM Apps 服務可見 at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Wed Apr  8 10:57:52 2026 from 192.168.17.128
hua0731@app:~$ cat ~/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJMPjTtMgJly3Q672O6P/yvQzekp8mEHewVF0DpdRkFY bastion-key
```

  - bastion → db：（貼上輸出）
```
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-19-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

對 Applications 延伸的安全維護未啟用

49 個更新可立即套用。
26 個標準安全更新。
要檢查這些額外的更新，請執行 apt list --upgradable

7 的安全更新可以 ESM Apps 來套用。關於啟用 ESM Apps 服務可見 at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Wed Apr  8 10:58:26 2026 from 192.168.17.128
hua0731@db:~$ cat ~/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJMPjTtMgJly3Q672O6P/yvQzekp8mEHewVF0DpdRkFY bastion-key
```

## 防火牆規則

### app 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
```
hua0731@app:~$ sudo ufw status verbose
[sudo] hua0731 的密碼： 
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
-                          --          --
22/tcp                     ALLOW IN    192.168.17.128      
```

### db 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
```
hua0731@db:~$ sudo ufw status verbose
[sudo] hua0731 的密碼： 
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
-                          --          --
22/tcp                     ALLOW IN    192.168.17.129            
22/tcp                     ALLOW IN    192.168.17.128            
```

### 防火牆確實在擋的證據
（貼上步驟 13 的 curl 8080 失敗輸出）
```
hua0731@bastion:~$ curl -m 5 http://192.168.17.129:8080 2>&1
curl: (28) Connection timed out after 5003 milliseconds
```

## ProxyJump 跳板連線
- 指令：（貼上你使用的 ssh -J 或 ssh config 設定）
```
hua0731@bastion:~$ ssh hua0731@192.168.17.129 "hostname"
app
hua0731@bastion:~$ ssh -J hua0731@192.168.17.129 hua0731@192.168.17.130 "hostname"
db
```

- 驗證輸出：（貼上連線成功的 hostname 輸出）
```
hua0731@bastion:~$ ssh app "hostname"
ssh db "hostname"
app
db
```

- SCP 傳檔驗證：（貼上結果）
```
hua0731@bastion:~$ echo "Test file via ProxyJump" > /tmp/proxy-test.txt
hua0731@bastion:~$ scp /tmp/proxy-test.txt app:/tmp/
ssh app "cat /tmp/proxy-test.txt"
proxy-test.txt                                                                                                              100%   24    31.5KB/s   00:00    
Test file via ProxyJump
hua0731@bastion:~$ scp /tmp/proxy-test.txt db:/tmp/
ssh db "cat /tmp/proxy-test.txt"
proxy-test.txt                                                                                                              100%   24    16.0KB/s   00:00    
Test file via ProxyJump
```

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | active + allow SSH from bastion |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **timed out** | 成功 |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp \| grep :22 | 有監聽 | 無監聽 | 有監聽 |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **refused** | 成功 |

## timeout vs refused 差異

timeout 表示封包無法到達目標服務，通常是被防火牆阻擋或網路路徑中斷，屬於較底層的問題。  
refused 則表示已成功連到主機，但目標 port 沒有服務在監聽，通常是服務未啟動或被關閉，屬於應用層問題。

## 網路拓樸圖
<img width="522" height="691" alt="network-diagram" src="https://github.com/user-attachments/assets/6a7e9290-cd3d-4df5-a20a-d3c76f2ecb99" />

## 排錯紀錄

- 症狀：
  在設定防火牆後，bastion 無法透過 SSH 連入 app，出現 connection timed out 的情況。

- 診斷：
  先使用 `ping` 確認 bastion 與 app 之間網路仍可通，排除網路問題；再確認 SSH 服務狀態與 port 22 監聽情況。接著使用 `sudo ufw status` 檢查防火牆規則，發現允許的來源網段設定錯誤。

- 修正：
  修正 app 上的防火牆規則，將原本錯誤的 192.168.56.0/24 改為正確的 192.168.17.0/24，並重新載入防火牆設定。

- 驗證：
  再次從 bastion 使用 SSH 連線至 app，確認可以正常登入，並使用 `ssh hua0731@192.168.17.129 "hostname"` 成功取得回應，證明問題已排除。

## 設計決策

在這次的lab中，db 僅允許 bastion 與 app 透過 SSH 存取，而非對整個網段開放。這樣的設計可以降低暴露面，確保只有受信任的管理端（bastion）與應用層（app）能夠連入資料層（db），提升整體安全性。

此外，在跳板設計上選擇使用 bastion 作為入口，並透過 ProxyJump 連入內部主機，使內部機器不需直接對外開放 SSH，兼顧安全性與管理便利性。這種架構常見於實際企業環境中。
