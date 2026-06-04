# W08｜容器生產實踐

## Healthcheck 故障測試
- 停 db 後幾秒被標 unhealthy：約 30 秒
- 對應的 log 訊息：healthcheck failed: connect to db failed curl: (7) Failed to connect to db port 5432 container marked as unhealthy

## Log 失控估算
- noisy 容器 30s log 大小：約 12 MB
- 預估 24h 大小：約 34 GB
（12 MB / 30 秒 × 2880）
- 套 rotation 後穩定上限：約 50 MB
（max-size=10m、max-file=5）

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137, OOMKilled=true | memory.max | 134217728 |
| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 50% | cpu.max | 50000 100000 |

## 權限四階對照
| 階梯 | id | CapEff | NoNewPrivs | curl /healthz |
|---|---|---|---|---|
| 0 | uid=0(root) | 00000000a80425fb | 0 | 200 |
| 1 | uid=1000(appuser) | 0000000000000000 ← 切非 root 後就是 0 | 0 | 200 |
| 2 | uid=1000(appuser) | 0000000000000000 | 0 | 200 |
| 3 | uid=1000(appuser) | 0000000000000000 ← 還是 0，但多一層 defense-in-depth | 0 | 200 |
| 4 | uid=1000(appuser) | 0000000000000000 | 1 | 200 |

## 排錯紀錄
- 症狀 / 診斷 / 修正 / 驗證  

症狀：
啟用 read_only: true 後，App 容器啟動失敗或崩潰，查看 docker compose logs app 出現 OSError: [Errno 30] Read-only file system，且指向 /home/appuser/.cache/。

診斷：
Python 框架或特定套件在執行時，會嘗試向預設的 HOME 或快取路徑寫入暫存檔案（如 .pyc 檔案或套件快取），但在唯讀根檔案系統下該操作會被拒絕。

修正：
在 compose.yaml 的 app 服務中掛載額外的 tmpfs 暫存區：  
tmpfs:
  - /tmp:size=32M
  - /home/appuser/.cache:size=16M

驗證：
重新執行 docker compose up -d --build app，容器順利維持 Running 狀態，且執行 curl http://localhost:8080/healthz 回傳 200。

## 設計決策
（你選的 mem_limit / cpus 數值理由是什麼？read_only 之後你補了哪些 tmpfs，為什麼？）  
1.資源限制（mem_limit / cpus）的數值理由：  
透過 docker stats 觀察日常真實運作，App 平均消耗約 100-150 MB 記憶體、CPU 峰值不超過 0.2 核。依據「真實用量之 1.5 倍」原則：  
App 給予 mem_limit: 256m 與 cpus: "0.5"，既能滿足突發流量，又能在程式陷入無窮迴圈或記憶體洩漏（Memory Leak）時，及時被 OOM killer 終止，避免拖垮整台宿主機（Host）。  
DB（PostgreSQL）屬於重量級服務，給予較寬裕的 mem_limit: 512m 與 cpus: "1.0" 確保快取與查詢效能。
  
2.read_only 後補上 tmpfs 的選擇與理由：  
為了達到極致的安全防禦，我們開啟了 read_only: true 阻止惡意網頁 Shell（Web Shell）塞入 /usr/bin 等目錄。但應用程式仍需寫入暫存檔，因此掛載了：  
/tmp:size=32M：供 Python 內建 tempfile 模組或 Shell 腳本寫入基本暫存。  
/home/appuser/.cache:size=16M：解決特定 Python 依賴庫在執行期間需要快取寫入權限的問題。  
皆使用 tmpfs 限制大小，既保證資料只存在於記憶體中、重啟即消失，也防止暫存檔把記憶體吃光。
