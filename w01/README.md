# 412637273_TKUCVT
# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：vct-w01-412637273

- Ubuntu 版本：
<img width="583" height="128" alt="image" src="https://github.com/user-attachments/assets/9b35aee2-970d-4f85-a731-e90ce5cf603e" />

- Docker 版本：
<img width="615" height="60" alt="image" src="https://github.com/user-attachments/assets/f8f378b1-5c69-431f-86c1-80eb055b2bf8" />

- Docker Compose 版本：
<img width="603" height="39" alt="image" src="https://github.com/user-attachments/assets/fce8a0ef-d7cd-4d52-bc66-97d333e8d9c5" />

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s): 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | Men: 3.8Gi  1.4Gi  525Mi  35Mi  2.2Gi  2.4Gi |
| 磁碟 | 40 GB | `df -h /` | /dev/sda2   40G  11G  27G 30%  / |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor vendor:   VMare |

## 四層驗收證據
- [ ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
<img width="806" height="63" alt="image" src="https://github.com/user-attachments/assets/d97a374c-4011-496a-82d4-a9782237bb6a" />

- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出
<img width="803" height="153" alt="image" src="https://github.com/user-attachments/assets/9394587c-610e-475d-bc17-8765d6eb618d" />

- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
<img width="806" height="530" alt="image" src="https://github.com/user-attachments/assets/4465e342-6ceb-493e-bca5-e78a6f8bc2cc" />

- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
<img width="795" height="492" alt="image" src="https://github.com/user-attachments/assets/09007205-e6cf-475c-8a80-0981f33ec958" />

- [ ] Compose：`docker compose version` 可執行
<img width="610" height="43" alt="image" src="https://github.com/user-attachments/assets/6207b962-a8fe-4687-aa6a-eb981e236487" />

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
<img width="733" height="667" alt="image" src="https://github.com/user-attachments/assets/10990ecc-90bf-4b41-9830-92fdb8ab716a" />

- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
<img width="793" height="419" alt="image" src="https://github.com/user-attachments/assets/148cdcaf-8c48-484a-806b-ba30da3ee38a" />

- [ ] 映像列表：`sudo docker images` 輸出
<img width="812" height="127" alt="image" src="https://github.com/user-attachments/assets/92b8b5e8-1001-4ec0-a86f-8c6df3b12a49" />

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | （時間點）2026/3/22 21:57:10 | （此節點代表的狀態）建立第一個可回復基線。建 snapshot 之前必須先確認環境健康。 
| （列出建點前做了哪些驗證）hostnamectl
ip route
sudo docker --version
docker compose version
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world |
| docker-ready | （時間點）2026/3/22 22:05:28 | （此節點代表的狀態）節省重複部署的時間 
| （列出建點前做了哪些驗證）sudo systemctl status docker --no-pager
sudo docker run --rm hello-world
sudo docker images    # 確認 nginx、alpine 映像都在 |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | （填入）是 |
| apt-cache policy 有候選版本 | 是 | 否 | （填入）是 |
| docker 重裝可行 | 是 | 否 | （填入）是 |
| hello-world 成功 | 是 | N/A | （填入）是 |
| nginx curl 成功 | 是 | N/A | （填入）是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | （你的實測）約 30 秒 | （你的實測）約 1～2 分鐘（含開關機） |
| 適用情境 | （你的判斷）小錯誤（單一檔案、設定明確） | （你的判斷）大規模錯誤、不確定改動內容 |
| 風險 | （你的判斷）修錯或漏修設定，可能造成更大問題 | （你的判斷）幾乎無風險，但會遺失未保存變更 |

## Snapshot 保留策略
- 新增條件：每次重大變更（安裝 Docker、修改系統設定）且功能驗證成功後建立
- 保留上限：最多保留 3 個 Snapshot
- 刪除條件：新 Snapshot 建立後，且舊 Snapshot 已無回復需求時刪除最舊版本

## 最小可重現命令鏈
（列出讓他人能重現故障注入與回復驗證的命令序列）
/# 基線確認
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10

/# 注入故障（移除 docker repo）
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

/# 驗證故障
apt-cache policy docker-ce | head -10
sudo apt -y install docker-ce

/# 手動修復
sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
sudo apt update

/# Snapshot 回復（操作：VM → Revert）

/# 回復後驗證
sudo docker run --rm hello-world
sudo docker run -d -p 8080:80 nginx
curl http://localhost:8080

## 排錯紀錄
- 症狀：無法安裝 docker-ce，apt 顯示找不到候選版本
- 診斷：（你首先查了什麼？）
  使用 ls /etc/apt/sources.list.d/ 發現 docker.list 消失
  使用 apt-cache policy docker-ce 確認沒有 repository
- 修正：（做了什麼改動？）
  將 docker.list.broken 改回 docker.list 並重新 apt update
- 驗證：（如何確認修正有效？）
  apt-cache policy docker-ce 出現 Candidate
  docker run hello-world 成功
  curl localhost:8080 正常回應 nginx 頁面

## 設計決策
（說明本週至少 1 個技術選擇與取捨）
本次選擇使用 Snapshot 作為主要回復機制，而非完全依賴手動修復。

取捨說明：
優點：
可快速回復到「已驗證正確」的狀態
避免人為操作錯誤累積
適合教學與實驗環境反覆測試
缺點：
需要額外儲存空間
回復會遺失回復點之後的變更
