# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：
  Layers 是 Docker image 一層一層堆疊起來的唯讀檔案系統。每做一次修改（例如安裝套件、複製檔案）通常就會新增一層。不同 image 如果有相同 layer，可以共用，不需要重複儲存。

- Config 是什麼：
  Config 是 image 的設定資訊，裡面會記錄 container 啟動時要用的內容，例如環境變數、預設指令、工作目錄、開放 port 等。

- Manifest 是什麼：
  Manifest 是 image 的描述清單，用來記錄這個 image 包含哪些 layer，以及對應的 config 是哪一個。Docker pull image 時，會先讀 manifest，再去下載需要的 layer。
  
## python:3.12-slim inspect 摘錄
- Config.Cmd：
`["python3"]`

- Config.Env：
```
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=C.UTF-8
GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305
PYTHON_VERSION=3.12.13
PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf052076684
```

- Config.WorkingDir：
未設定

- RootFS.Layers 數量：
4

## Layer 快取實驗

| 情境 | build 時間 |
|---|---|
| v1 首次 build | 2m30s |
| v1 改 app.py 後 rebuild | 2m25s |
| v2 首次 build | 2m30s |
| v2 改 app.py 後 rebuild | 5s |

觀察：

v2 的 rebuild 會明顯比較快，因為 Dockerfile.v2 先複製 `requirements.txt` 並執行 `pip install`，最後才複製 `app.py`。  
因此當我只修改 `app.py` 時，前面的套件安裝 layer 沒有改變，Docker 就可以直接使用 cache，不需要重新安裝 Flask 套件，所以 build 很快就完成。

相反地，v1 是先 `COPY app/ .` 再執行 `pip install`，只要 `app.py` 有修改，前面的 COPY layer 就會改變，導致後面的 `pip install` 也必須重新執行，因此 rebuild 還是很慢。

這也代表 Docker layer cache 可以有效減少重複安裝套件的時間，提高 build 效率。

## CMD vs ENTRYPOINT 實驗

<img width="1087" height="451" alt="螢幕擷取畫面 2026-05-19 172542" src="https://github.com/user-attachments/assets/d9b69ddd-2458-43fe-9a1d-dafca64fa788" />

<img width="1604" height="424" alt="螢幕擷取畫面 2026-05-19 172624" src="https://github.com/user-attachments/assets/a4b721fc-953c-4f39-979c-2e4a34ecabfb" />

| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | `argv = ['show_args.py', 'default1', 'default2']`<br>`PID = 7` | `exec: "extra1": executable file not found` |
| CMD exec form | `argv = ['show_args.py', 'default1', 'default2']`<br>`PID = 1` | `exec: "extra1": executable file not found` |
| ENTRYPOINT + CMD | `argv = ['show_args.py', 'default1', 'default2']`<br>`PID = 1` | `argv = ['show_args.py', 'extra1', 'extra2']`<br>`PID = 1` |

### 結論

shell form 的 CMD 其實會透過 `/bin/sh -c` 執行，因此 PID 1 通常是 shell，再由 shell 啟動 python。

exec form 則是直接執行 python，所以 PID 1 就是 python 本身。

當額外加入參數時，shell form 與 exec form 的 CMD 都會被整條覆蓋，因此 Docker 會嘗試直接執行 `extra1`，最後出現 executable not found 的錯誤。

而 ENTRYPOINT + CMD 的組合會保留主程式 `python show_args.py`，再把 `extra1 extra2` 當成附加參數，因此能正常執行。

因此 production 環境通常會使用 exec form 的 ENTRYPOINT + CMD，因為：
- PID 1 正確
- signal 處理比較正常
- 參數追加行為穩定
- 較適合正式環境部署

## Multi-stage 大小對照

| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.02GB |
| python:3.12-slim（runtime base） | 179MB |
| myapp:v2（單階段） | 191MB |
| myapp:multi（多階段） | 183MB |

解釋：

Multi-stage build 會把建置階段和執行階段分開。builder stage 使用 `python:3.12`，主要負責安裝套件與準備環境；runtime stage 則只保留真正執行程式需要的內容，因此最後的 image 不會包含 builder 階段的編譯工具、pip cache 與中間 layer。

builder stage 的 layer 並沒有真的消失，而是只留在本機的 build cache 中，方便下次 build 重用，但不會被包進最終的 runtime image。

## .dockerignore 故障注入

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 24K | 101M | 24K |
| build context 傳輸大小 | 1.2kB | 101MB | 1.2kB |
| build 時間 | 5s | 21s | 5s |

## 排錯紀錄

- 症狀：
  在 build image 時，發現 build context 傳輸大小突然變大，build 時間也明顯增加。

- 診斷：
  檢查專案目錄後，發現有大型測試檔案與暫存資料被一起送進 build context。Docker 在 build 前會先把目前目錄傳給 daemon，因此即使 Dockerfile 沒有使用那些檔案，也會影響 build 效率。

- 修正：
  新增 `.dockerignore`，將不需要打包進 image 的檔案排除，例如大型測試檔、cache、log 與暫存資料。

- 驗證：
  加入 `.dockerignore` 後，再次 build，build context 傳輸大小恢復正常，build 時間也明顯縮短。

## 設計決策

本週 runtime image 選擇 `python:3.12-slim`，而不是 `alpine`，主要是因為 slim 版本在 image 大小與相容性之間取得較好的平衡。

雖然 alpine 更小，但它使用 musl libc，有些 Python 套件或 C extension 可能需要額外編譯與相容性處理，容易增加除錯成本。相較之下，`python:3.12-slim` 使用 Debian 環境，與大部分 Python 套件的相容性更好，也比較接近實際 production 環境。

另外，透過 multi-stage build，可以把 builder 階段的編譯工具與暫存檔排除在最終 image 外，讓 image 更小、更乾淨，也降低安全風險。
