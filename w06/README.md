# W06｜Docker Image 與 Dockerfile

## 映像組成
- **Layers 是什麼**：Layers（鏡像層）是唯讀的檔案系統差異包（Tarball）。在 Dockerfile 中，每執行一個會變動檔案系統的指令（如 RUN, COPY）就會造出一層 Layer。它們以堆疊的方式組合（透過 overlay2），並且可以在不同的 Image 之間共享，用以節省硬碟空間。
- **Config 是什麼**：Config 是一份 JSON 格式的元數據（Metadata）文件。它紀錄了容器啟動時的預設指令（CMD/ENTRYPOINT）、工作目錄（WORKDIR）、環境變數（ENV）、執行用戶（USER）等設定，它不包含實際的檔案系統，只決定容器如何運作。
- **Manifest 是什麼**：Manifest（清單）是 Image 的核心索引檔案。它負責把所有的唯讀 Layers 和那份 Config JSON 綁在一起，裡面記錄了 Config 的參照以及每一層 Layer 的下載雜湊值（Digest）與檔案大小，讓 Docker 知道如何完整下載並組合這個 Image。

## python:3.12-slim inspect 摘錄
- Config.Cmd：["python3"]
- Config.Env：["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "LANG=C.UTF-8",
                "GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305",
                "PYTHON_VERSION=3.12.13",
                "PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684"]
- Config.WorkingDir：""
- RootFS.Layers 數量：4

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 0m0.232s |
| v1 改 app.py 後 rebuild | 0m5.444s |
| v2 首次 build | 0m5.004s |
| v2 改 app.py 後 rebuild | 0m0.293s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？
因為 Docker 的 Layer 快取機制遵循「一旦某層 Miss，其後所有層皆 Miss」的殘酷規則。在 v1 中，`COPY app/ .` 被放在 `pip install` 之前，只要改了 `app.py` 就會導致 COPY 層快取失效，連帶逼迫後面的 `pip install` 必須整個重跑。
而在 v2 中，我們調整了順序，先單獨 `COPY app/requirements.txt .` 並進行 `pip install`。因為 requirements 檔案沒變，這兩層牢牢鎖定在 `CACHED` 狀態，只有最後複製 `app.py` 的那一層重新計算，因而省下了大量重複下載與安裝套件的時間。
## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | argv = ['show_args.py', 'default1', 'default2'] PID  = 8 | docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: exec: "extra1": executable file not found in $PATHRun 'docker run --help' for more information |
| CMD exec form | argv = ['show_args.py', 'default1', 'default2'] PID  = 1 | docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: exec: "extra1": executable file not found in $PATHRun 'docker run --help' for more information |
| ENTRYPOINT + CMD | argv = ['show_args.py', 'default1', 'default2'] PID  = 1 | argv = ['show_args.py', 'default1', 'default2'] PID  = 1 |

結論（用自己的話寫）：
1. **Shell Form vs Exec Form**：Shell form 會預先呼叫 `/bin/sh -c` 來執行命令，導致應用程式變成 PID 1 的子進程，無法直接接收到系統的 SIGTERM 訊號進而優雅關閉（Graceful Shutdown）。Exec form（JSON 陣列）則會直接將應用程式拉起為 PID 1。因此生產環境推薦一律使用 **Exec Form**。
2. **參數覆蓋機制**：如果只使用 `CMD`，只要在 `docker run` 後面接任何參數，舊的 `CMD` 就會被**整條完全覆蓋**。而採用 `ENTRYPOINT` (主程式) + `CMD` (預設參數) 的黃金組合時，`docker run` 後面接的參數只會**覆蓋 CMD 的部分**，並漂亮地當作引數附加在主程式後面。這提供了極高的彈性與穩定性。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.62GB |
| python:3.12-slim（runtime base） | 179MB |
| myapp:v2（單階段） | 197MB |
| myapp:multi（多階段） | 184MB |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？
Builder stage 的 Layers 並沒有消失，它們依然完好地保存在我們本地的 Docker Build Cache（快取）中，因此當我們下一次 rebuild 時，仍然可以享受到編譯快取的好處（可以使用 `docker images -a` 看到這些 `<none>:<none>` 的中間層）。
然而，當 Docker 生成最終的 `myapp:multi` 鏡像標籤，或是我們將其 Push 到 Registry 時，**只有 Runtime stage 的 Layers 會被打包進去**。那些巨大的編譯工具（如 GCC、Make）與標頭檔成功被留在第一階段，不會佔用最終生產環境的部署空間與頻寬。

## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 44K | 151M | 48K |
| build context 傳輸大小 | 419B  | 129B | 129B |
| build 時間 | 0m0.265s | 0m0.283s | 0m0.262s |

## 排錯紀錄
- **症狀**：在切換為 `USER appuser` 安全設定後，啟動容器遭遇 Crash，並從 `docker logs` 中觀察到錯誤訊息 `Permission denied: '/app/app.py'`。
- **診斷**：由於 Dockerfile 中 `COPY app/ .` 預設是以 `root` 權限將檔案複製進容器，隨後直接切換為非特權用戶 `appuser` 執行，導致 Python 嘗試讀取與執行檔案時，因權限不足而被系統阻擋。
- **修正**：在 `USER appuser` 指令上方，加入修改權限的指令：`RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app`（或者在 COPY 時直接指定 `--chown=appuser:appuser`）。
- **驗證**：重新 build 鏡像，並執行容器。使用 `curl` 能夠成功打入首頁，且進入容器內部執行 `docker exec <container-id> whoami`，正確回傳 `appuser`，證明排錯成功。

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）
在生產環境的 Runtime Stage 選擇中，我們選擇了 `python:3.12-slim` 而非體積更小的 `python:3.12-alpine`。
雖然 Alpine 基於 `musl libc` 能帶來極致的輕量化，但在 Python 的生態系中，社群絕大多數的預編譯第三方套件（Wheel 檔）都是基於 `glibc`（Linux 標準 C 函式庫）建置的。如果選擇 Alpine，未來只要遇到需要編譯的資料分析庫（如 NumPy、Pandas）或加密庫，容器就必須在 build 時重新下載 gcc 從頭編譯，這不僅會讓 build 時間暴增數十分鐘，更常常遇到 C 語言依賴缺失的噴錯地獄。
因此，選擇基於 Debian 的 `slim` 版本，雖然多了數十 MB 的體積，卻能完美兼容各類預編譯套件，在「映像檔體積」與「維運與建置穩定性」之間取得了最佳的工程取捨。
