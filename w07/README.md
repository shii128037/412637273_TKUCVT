# W07｜Docker Compose 與資料持久化

## 拓樸圖
（mermaid 或 ASCII，標出 app、db、default network、db-data volume）

## 從 docker run 到 compose.yaml
（自己的話：你最有感的一個改善是什麼？）
最有感的改善是「宣告式管理的重現性」與「內建服務發現（DNS）」。
以前用 docker run 部署時，必須小心翼翼地手動建立網路、建立 Volume，還要依序敲出一長串帶有 -e、-v、--network 的指令。只要漏抄一個參數或順序顛倒，學弟妹重現時就會直接炸開。
換成 compose.yaml 後，整個基礎架構直接變成「程式碼（IaC）」。我只需要描述最終狀態，Compose 就會自動幫我把網路建好，並把服務名稱直接當成內建的 DNS（App 只要指定 DB_HOST=db 就能通）。這種一鍵 docker compose up -d 就能完美重現的輕盈感，完全解決了環境不一致的痛點。

## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume | /var/lib/docker/volumes/w07_db-data/_data | 在 | 完整保留 | 生產環境的資料庫 |
| bind mount | ./app | 在 | 完整保留 | 本地開發階段 |
| tmpfs | 不落地 | 不在 | 完全消失 | 敏感暫存、高速快取 |

## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on | 503 | 503 | 503 | 200 |
| service_healthy | Connection refused | Connection refused | Connection refused | 200 |

觀察（自己的話）：

只用 depends_on 的慘劇（步驟 17）：
當 db 容器一轉換為 Running 狀態，Compose 就會立刻把 app 容器也拉起來。但此時 Postgres 內部實際上還在執行人為的 sleep 8 與資料庫初始化，根本還沒準備好對外連線。這導致 app 在前幾秒（t=1s~8s）頻繁打進來的請求都會因為連不上資料庫，而在前端噴出大堆 503 db unreachable 錯誤代碼，並在日誌裡塞滿 connection refused 的噪音。

改用 service_healthy 的改變（步驟 19）：
這次 app 容器被強迫進入「等待狀態」，直到 db 容器內的 pg_isready 健康檢查成功回報為 healthy（約在第 8、9 秒後），Compose 才會正式啟動 app。
因此在 t=1s~5s 時，由於 app 根本還沒啟動、沒有任何程式在監聽 8080 埠口，我們 curl 會直接得到 Connection refused（拒絕連線）。但這種狀態非常優雅，因為只要 app 一順利開起來，第一發打進去的 curl 就絕對是健康的 200 ok，徹底解決了微服務之間因為啟動時間差所造成的短暫失控。

## 排錯紀錄
- 症狀：執行 docker compose up -d 啟動服務後，立刻對網頁發送請求 curl http://localhost:8080/healthz，前幾秒會連續收到 503 db unreachable 錯誤回應，直到約 10 秒後才成功返回 200 ok。
- 診斷：檢查 compose.yaml 發現第一版只寫了單純的 depends_on: - db。這只能確保 db 容器轉換為 Running 狀態就立刻啟動 app，但此時 Postgres 內部仍在執行 sleep 8 與資料庫初始化，尚未準備好接受 TCP 連線。app 太早嘗試連線，導致在初始化窗口期拋出大量連線失敗的 Log 與 503 噪音。
- 修正：在 db 服務中加入 healthcheck（使用 pg_isready 指令檢查），並將 app 的依賴條件修改為宣告式的進階寫法：  
- depends_on:
  db:
    condition: service_healthy
- 驗證：執行 docker compose down 完整關閉後重新啟動，並使用 Shell 迴圈測試。在前 8 秒內，由於 app 被強迫等待 db 就緒而不啟動，curl 直接返回 Connection refused（拒絕連線）；而當第 9 秒 app 正式開起來後，第一發打入的請求便直接回傳 200 ok，順利消除 503 錯誤。

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）
1. 為什麼 db 用 named volume 而不是 bind mount？
效能與平台相容性：
Postgres 等主流資料庫在底層讀寫時，極度依賴作業系統的檔案鎖定（File Locking）與記憶體對映（mmap）。Named Volume 是在 Docker 專屬管理的 Host 檔案系統路徑下（Linux 原生為 /var/lib/docker/volumes/）進行原生讀寫，能提供最穩定的高 I/O 效能。若改用 bind mount，資料庫將直接受制於 Host 端的檔案權限、SELinux，或在 Mac/Windows 環境中面臨虛擬化文件共享層（如 gRPC FUSE / VirtioFS）的效能極速下降與檔案鎖定失效，極易導致資料損壞。

權限與安全性（職責分離）：
bind mount 會把 Host 端的特定目錄權限直接帶入容器。資料庫容器（常以特定 UID 如 999 執行）若沒有 Host 該目錄的完整權限，會直接拋出 Permission denied 無法啟動；另外，維運人員本就不該、也不需要直接去 Host 檔案系統裡用編輯器手動修改資料庫的二進位檔案。交由 Docker 封裝管理的 named volume 是最安全的生產實踐。

2. 為什麼不能在生產用 tmpfs 存資料庫？
徹底背離 ACID 持久性（Durability）原則：
tmpfs 是直接將資料擺在 Host 主機的記憶體（RAM）中，資料完全不落地。生產環境的資料庫必須保證交易提交後的永久儲存。如果把資料庫的 data 目錄掛載為 tmpfs，只要遇到以下任何一個情況，資料庫內的所有正式資料與使用者軌跡都會在一瞬間灰飛煙滅且完全無法復原：

實體主機無預警斷電。

容器遭遇記憶體不足（OOM）被作業系統核心強制殺掉。

維運人員單純重啟容器（docker compose restart）。

容量與資源競爭：
生產環境的資料量通常會隨著時間增長，遠遠大於伺服器的實體記憶體總量。使用 tmpfs 不僅會迅速榨乾系統的 RAM 資源、觸發系統崩潰，其高昂的記憶體成本也完全不符合儲存海量數據的經濟效益。
