# W08｜容器生產實踐

## Healthcheck 故障測試

* 停 db 後幾秒被標 unhealthy：30 秒內（約 25~30 秒）
* 對應的 log 訊息：

```text
db unreachable
connection refused
database connection failed
```

說明：

停止 db 後，app 容器本身仍然維持 running 狀態，但 healthcheck 持續失敗，在 retries 次數累積完成後被標記為 unhealthy。Docker 不會因為 unhealthy 自動重啟容器。

---

## Log 失控估算

* noisy 容器 30s log 大小：35 MB
* 預估 24h 大小：100.8 GB
* 套 rotation 後穩定上限：6 MB（2 MB × 3 檔）

計算：

```text
35 MB / 30s
≈ 1.17 MB/s

1.17 × 86400
≈ 100.8 GB/day
```

加上：

```yaml
logging:
  driver: json-file
  options:
    max-size: "2m"
    max-file: "3"
```

後，log 大小會穩定維持在約 6 MB。

---

## 資源限制實驗

| 實驗           | 命令                               | 觀察結果                    | 對應 cgroup 檔 | 值            |
| ------------ | -------------------------------- | ----------------------- | ----------- | ------------ |
| OOM          | stress-ng --vm 1 --vm-bytes 200m | exit 137，OOMKilled=true | memory.max  | 134217728    |
| CPU throttle | stress-ng --cpu 4                | docker stats CPU% ≈ 50% | cpu.max     | 50000 100000 |

說明：

OOM 實驗中，容器記憶體上限設定為 128 MB，但程式一次配置 256 MB 記憶體，因此被 Linux OOM Killer 強制終止，Exit Code 為 137。

CPU 實驗中，即使 stress-ng 建立 4 個 CPU worker，由於 cgroup quota 限制為 0.5 CPU，因此 docker stats 顯示 CPU 使用率維持在約 50%。

---

## 權限四階對照

| 階梯 | id                | CapEff           | NoNewPrivs | curl /healthz |
| -- | ----------------- | ---------------- | ---------- | ------------- |
| 0  | uid=0(root)       | 00000000a80425fb | 0          | 200           |
| 1  | uid=1000(appuser) | 0000000000000000 | 0          | 200           |
| 2  | uid=1000(appuser) | 0000000000000000 | 0          | 200           |
| 3  | uid=1000(appuser) | 0000000000000000 | 0          | 200           |
| 4  | uid=1000(appuser) | 0000000000000000 | 1          | 200           |

說明：

* 階梯 1：改用非 root 身分執行。
* 階梯 2：root filesystem 改成唯讀。
* 階梯 3：移除所有 Linux capabilities。
* 階梯 4：啟用 no-new-privileges，防止透過 setuid 程式再次升權。

---

## 排錯紀錄

### 症狀

啟用 `read_only: true` 後，app 無法正常啟動。

### 診斷

查看：

```bash
docker compose logs app
```

發現 Python 套件嘗試寫入暫存目錄，產生：

```text
OSError: [Errno 30] Read-only file system
```

### 修正

在 compose.yaml 中新增：

```yaml
tmpfs:
  - /tmp:size=32M
  - /home/appuser/.cache:size=16M
```

提供可寫入的暫存空間。

### 驗證

重新：

```bash
docker compose up -d --build
```

之後：

```bash
curl http://localhost:8080/healthz
```

成功回傳：

```text
200 OK
```

表示服務恢復正常。

---

## 設計決策

### 為什麼 mem_limit 設定 256m？

根據教材範例 Flask + PostgreSQL 的需求，平時記憶體用量遠低於 256 MB。

設定：

```yaml
mem_limit: 256m
```

可以避免程式發生記憶體洩漏時拖垮整台主機，同時保留足夠的緩衝空間。

### 為什麼 cpus 設定 0.5？

這個服務屬於輕量 Web API，不需要長時間佔滿 CPU。

設定：

```yaml
cpus: "0.5"
```

可以避免無限迴圈或異常程式將 Host CPU 打滿，確保其他服務仍能正常運作。

### 為什麼 read_only 之後要補 tmpfs？

啟用：

```yaml
read_only: true
```

後，容器內所有檔案系統都變成唯讀。

但許多 Python 套件與 tempfile 模組仍需要：

```text
/tmp
/home/appuser/.cache
```

作為暫存空間。

因此補上：

```yaml
tmpfs:
  - /tmp:size=32M
  - /home/appuser/.cache:size=16M
```

讓程式可以正常執行，同時維持唯讀 root filesystem 的安全性。

### 為什麼使用 cap_drop + no-new-privileges？

即使攻擊者取得容器存取權：

* 無法透過 Linux Capability 操作系統功能
* 無法利用 setuid 程式升權
* 無法修改 root filesystem

達到最小權限（Least Privilege）原則，降低容器被入侵後的影響範圍。
