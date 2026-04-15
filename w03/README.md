# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.11.133 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.75.128 | — |
| app | 應用層 | NIC 1 | Host-only | 192.168.75.129 | SSH from 192.168.56.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.75.131 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：（例：ed25519）
- 公鑰部署到：（例：app 和 db 的 ~/.ssh/authorized_keys）
- 免密碼登入驗證：
  - bastion → app：（貼上輸出）
  - bastion → db：（貼上輸出）
  <img width="633" height="89" alt="image" src="https://github.com/user-attachments/assets/c79596e0-c6d9-4d61-9d82-712e1b69ea38" />

## 防火牆規則

### app 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
<img width="605" height="201" alt="image" src="https://github.com/user-attachments/assets/6e31ee06-c9d0-4d5c-8704-d5336a3a8913" />

### db 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
<img width="605" height="228" alt="image" src="https://github.com/user-attachments/assets/6feeec41-1e02-4e92-bb6b-2a916e6b3906" />

### 防火牆確實在擋的證據
（貼上步驟 13 的 curl 8080 失敗輸出）
<img width="814" height="66" alt="image" src="https://github.com/user-attachments/assets/830cc172-91d9-454c-b31e-8c39ec7a7084" />

## ProxyJump 跳板連線
- 指令：（貼上你使用的 ssh -J 或 ssh config 設定）ssh -J gina@192.168.75.128 gina@192.168.75.129 "hostname"
- 驗證輸出：（貼上連線成功的 hostname 輸出）
<img width="827" height="71" alt="image" src="https://github.com/user-attachments/assets/cb6a67ab-16eb-4c77-b8b3-aca8f19f0f7f" />
<img width="817" height="461" alt="image" src="https://github.com/user-attachments/assets/06390c57-9822-4ce8-8654-ad8e5444c77d" />

- SCP 傳檔驗證：（貼上結果）
<img width="776" height="281" alt="image" src="https://github.com/user-attachments/assets/4e7bfffb-6214-49ae-b997-6977bf379214" />

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | （填入）active + rules |
| bastion ping app | 成功 | （填入）Connection timed out | （填入）成功 |
| bastion SSH app | 成功 | **timed out** | （填入）成功 |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （填入）有監聽 |
| bastion ping app | 成功 | 成功 | （填入）成功 |
| bastion SSH app | 成功 | **refused** | （填入）成功 |

## timeout vs refused 差異
（用自己的話說明timeout（逾時）
timeout（逾時）代表封包沒有回應
常見原因：防火牆（UFW / security group）擋掉 、網路不通（route / subnet / NAT）
排錯方向： 查「網路層 / 防火牆 / IP / route」

connection refused（連線被拒絕）有到達主機，但服務不接受
常見原因：SSH / service 沒開（sshd 沒啟動）、 port 沒監聽 、防火牆允許但服務不存在
排錯方向： 查「服務層（daemon / port / process）」兩種錯誤的差異、各自指向什麼排錯方向）

## 網路拓樸圖
（嵌入或連結 network-diagram）
<img width="806" height="981" alt="network-diagram" src="https://github.com/user-attachments/assets/fd2db6bc-e16a-405c-b898-e889a89b73eb" />

## 排錯紀錄
- 症狀：bastion 無法 SSH 進 app、 出現 Permission denied (publickey) 或 timeout 、SCP ProxyJump 傳檔失敗
- 診斷：（你首先查了什麼？）
先測 bastion → app：ssh gina@192.168.75.129
檢查 UFW：sudo ufw status verbose
檢查 SSH key：
ls ~/.ssh
cat ~/.ssh/authorized_keys
- 修正：（做了什麼改動？）
修正 UFW 網段（192.168.75.0/24）
sudo ufw allow from 192.168.75.0/24 to any port 22 proto tcp

修正 SSH key 權限：
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

重新設定 key：
ssh-copy-id gina@192.168.75.129
- 驗證：（如何確認修正有效？）
ssh -J gina@192.168.75.128 gina@192.168.75.129 "hostname"
或：

scp -o ProxyJump=gina@192.168.75.128 file.txt app:/tmp/
成功看到 hostname / file content

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 db 允許 bastion 直連而不是只允許從 app 跳？）
- 決策：使用 Host-only 網路分離內網（app / db）與外部存取（bastion）
- 理由：
網路分層更清楚
bastion（192.168.75.128）負責對外入口
app / db（192.168.75.129 / .130）只存在內網
避免直接暴露服務到外部

降低攻擊面
外部只能 SSH 到 bastion
app / db 無法直接從外部存取
即使被掃描 IP，也無法直接連線

符合企業架構（Jump Server）
bastion = 唯一進入點
所有管理流量集中
可記錄與控管登入行為

- 取捨：
| 方案 | 優點 | 缺點 |
|---|---|---|
|Host-only + bastion|安全性高、架構清楚|設定較複雜|
|全部 NAT / 直接連線|設定簡單|安全性低|
