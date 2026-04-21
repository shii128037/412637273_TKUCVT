# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | （填入）系統設定檔目錄 | （填入）存放 Docker daemon 設定（daemon.json） |
| /var/lib/docker/ | （填入）可變資料（應用程式資料） | （填入）存放 image、container、volume 等實際資料 |
| /usr/bin/docker | （填入）使用者執行的二進位檔 | （填入）Docker CLI 指令位置 |
| /run/docker.sock | （填入）runtime socket（暫存） | （填入）CLI 與 Docker daemon 溝通的 socket |

## Docker 系統資訊

- Storage Driver：（貼上 `docker info` 中的值）overlayfs
<img width="272" height="25" alt="image" src="https://github.com/user-attachments/assets/bcef7e8b-2fc0-4689-89ee-f7c7c93ab853" />
- Docker Root Dir：（貼上 `docker info` 中的值）/var/lib/docker
<img width="333" height="24" alt="image" src="https://github.com/user-attachments/assets/59f0f1dd-c109-4532-8da9-be153f2c8ede" />
- 拉取映像前 /var/lib/docker/ 大小：（填入）232K
- 拉取映像後 /var/lib/docker/ 大小：（填入）236K

## 權限結構

### Docker Socket 權限解讀
（貼上 `ls -la /var/run/docker.sock` 輸出，逐欄說明 owner/group/others 的權限）
<img width="615" height="43" alt="image" src="https://github.com/user-attachments/assets/4931e1f6-4d7f-41d4-9ba0-3023bed19830" />

| 欄位        | 值                    | 說明                 |
| --------- | -------------------- | ------------------ |
| 檔案類型      | s                    | socket 檔案（用來程序間通訊） |
| owner 權限  | rw-                  | root 可讀寫           |
| group 權限  | rw-                  | docker 群組可讀寫       |
| others 權限 | ---                  | 其他人無權限             |
| 連結數       | 1                    | 檔案連結數              |
| 擁有者       | root                 | 系統管理者              |
| 群組        | docker               | Docker 使用群組        |
| 大小        | 0                    | socket 不佔實際檔案空間    |
| 時間        | Apr 21 23:01         | 最後修改時間             |
| 路徑        | /var/run/docker.sock | Docker 通訊 socket   |

### 使用者群組
（貼上 `id` 輸出，說明是否包含 docker 群組）
<img width="806" height="67" alt="image" src="https://github.com/user-attachments/assets/fa4b8a2b-823e-4263-a316-7176c4bd50b8" />
984(docker) 代表使用者 gina 有加入 docker 群組

### 安全意涵
（用自己的話說明為什麼 docker group ≈ root，安全示範的觀察結果）
由於使用者屬於 docker 群組，因此擁有對 /var/run/docker.sock 的讀寫權限，可以直接控制 Docker daemon。這代表該使用者具備接近 root 的權限（例如可透過掛載主機目錄來存取系統檔案），因此 docker 群組實際上等同於高權限群組，需謹慎使用。

## 程序與服務管理

### systemctl status docker
（貼上 `systemctl status docker` 輸出）
<img width="809" height="546" alt="image" src="https://github.com/user-attachments/assets/35249020-2ad1-47b6-8905-538518f45aa7" />

### journalctl 日誌分析
（貼上 `journalctl -u docker --since "1 hour ago"` 的重點摘錄，說明看到什麼事件）
<img width="806" height="330" alt="image" src="https://github.com/user-attachments/assets/a9384c99-b65a-44bd-8815-171b23899f56" />
在最近一小時的 Docker 日誌中，可以觀察到容器加入 bridge 網路（sbJoin）以及 container 被刪除的事件（task-delete）。這表示系統中有容器被建立並執行，隨後又被停止或移除，且 Docker daemon 能正常處理容器生命週期與網路配置，整體運作正常。

### CLI vs Daemon 差異
（用自己的話說明兩者的差異，為什麼 `docker --version` 正常不代表 Docker 能用）
Docker CLI：使用者操作的指令工具（例如 docker ps、docker run）
Docker Daemon（dockerd）：背景服務，負責真正建立、執行與管理 container
docker --version 只是檢查：
✔ Docker CLI 程式是否存在
✔ /usr/bin/docker 是否可以執行
但它不會連線 Docker daemon
即使 docker --version 顯示正常，也只代表 Docker CLI 安裝成功，並不代表 Docker daemon 正在運行。只有當 CLI 能成功透過 /var/run/docker.sock 與 daemon 溝通時，Docker 才算真正可用。

## 環境變數

- $PATH：（貼上內容）
<img width="810" height="68" alt="image" src="https://github.com/user-attachments/assets/a9d0356d-3b29-4d74-a9b4-49973360c285" />

- which docker：（填入路徑）/usr/bin/docker                                                    
- 容器內外環境變數差異觀察：（簡述）
<img width="808" height="307" alt="image" src="https://github.com/user-attachments/assets/16b07d50-10e5-4519-a98f-43709658ac4b" />
容器與主機的環境變數是隔離的
→ 容器不會直接繼承主機的 USER、HOME 等資訊

容器預設使用 root 身份執行
→ HOME 為 /root

容器的 PATH 較精簡
→ 只包含基本執行路徑（沒有 snap、games 等）

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | （填入）inactive (dead) | （填入）active |
| docker --version | 正常 | （填入）正常 | （填入）正常 |
| docker ps | 正常 | Cannot connect | （填入）正常 |
| ps aux grep dockerd | 有 process | （填入）無 process | （填入）有 process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | （填入）srw------- | （填入）srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | （填入）正常 |
| sudo docker ps | 正常 | （填入）正常 | （填入）正常 |
| systemctl status docker | active | （填入）active | （填入）active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | （填入）daemon 沒啟動 | （填入）systemctl status docker → 啟動 daemon |
| permission denied…docker.sock | （填入）daemon 在跑但無權存取 socket | （填入）ls -la /var/run/docker.sock + id → 檢查權限/群組 |

（用自己的話說明兩種錯誤的差異，各自指向什麼排錯方向）
Cannot connect
→ Docker 沒開

permission denied
→ Docker 有開，但你沒權限

## 排錯紀錄
- 症狀：docker ps 無法執行
- 診斷：（你首先查了什麼？）先檢查 systemctl status docker
- 修正：（做了什麼改動？）sudo systemctl start docker
- 驗證：（如何確認修正有效？）docker ps

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼教學環境用 `usermod` 加 group 而不是每次 sudo？這個選擇的風險是什麼？）
技術選擇：將使用者加入 docker 群組（usermod -aG docker）

在本週實作中，我選擇將使用者加入 docker 群組，使其可以直接執行 Docker 指令，而不需要每次都使用 sudo。

優點（Benefit）
使用方便：可直接執行 docker ps、docker run
提高開發效率：減少頻繁輸入 sudo
適合教學與開發環境

缺點（Trade-off）
安全風險高：docker 群組擁有對 /var/run/docker.sock 的讀寫權限
使用者可透過 Docker 取得主機完整控制權（等同 root）
違反最小權限原則（Principle of Least Privilege）
