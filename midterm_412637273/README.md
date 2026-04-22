# 期中實作 — <412637273> <鍾珺如>

## 1. 架構與 IP 表
<Mermaid 圖 + 表格>

```mermaid
graph TD

Internet((Internet))

subgraph Bastion
    B1[NAT\n192.168.11.133]
    B2[Host-only\n192.168.75.128]
end

subgraph App
    A1[Host-only\n192.168.75.129]
end

Internet --> B1
B2 --> A1
```


| VM          | 網卡            | 對外           | 對內           | 角色     |
| ----------- | --------------- | -------------- | -------------- | -------- |
| `bastion` | NAT + Host-only | 收 Host 的 SSH | 轉送到 `app` | 唯一入口 |
| `app`     | Host-only only  | 無             | 跑 nginx 容器  | 實際服務 |

## 2. Part A：VM 與網路
<命令 + 關鍵輸出>

bastion ip
<img width="810" height="461" alt="image" src="https://github.com/user-attachments/assets/a32d0954-d1ab-44a3-9d7d-51708a60aaa8" />
app ip
<img width="807" height="327" alt="image" src="https://github.com/user-attachments/assets/00ec1f91-410f-4fe8-8710-d279d594dadc" />
bastion ping
<img width="685" height="457" alt="image" src="https://github.com/user-attachments/assets/987ceb41-a9e8-4ea3-aa02-4ac75a32dca1" />
app ping
<img width="641" height="176" alt="image" src="https://github.com/user-attachments/assets/8ca98775-9b0c-4cf1-92a2-b193ea4d1d0f" />

## 3. Part B：金鑰、ufw、ProxyJump
<防火牆規則表 + ssh app 成功證據>
| VM      | 規則                                  | 說明          |
| ------- | ----------------------------------- | ----------- |
| bastion | allow 22/tcp                        | 允許外部 SSH    |
| bastion | deny all                            | 拒絕其他連線      |
| app     | allow from 192.168.75.128 to 22/tcp | 僅允許 bastion |
| app     | deny all                            | 拒絕其他        |
<img width="1470" height="748" alt="image" src="https://github.com/user-attachments/assets/62f1ad8b-4044-4344-9bce-92c60764023f" />

## 4. Part C：Docker 服務
<systemctl status docker + curl 輸出>
<img width="813" height="552" alt="image" src="https://github.com/user-attachments/assets/354df15b-bd9d-4343-880e-a567cc2329a1" />
<img width="569" height="227" alt="image" src="https://github.com/user-attachments/assets/e9695751-9189-4595-80a1-da168b8692b0" />

## 5. Part D：故障演練
### 故障 1：<F1/F2/F3 擇一>
F1：網路介面禁用
- 注入方式：app：sudo ip link set ens33 down
- 故障前：  
  檢查命令：ip link show ens33  
  輸出：顯示 state UP，且 Host 端可正常 ssh  
  app  
  <img width="814" height="151" alt="image" src="https://github.com/user-attachments/assets/4030b449-979e-43d0-b315-2e2a641287d4" />
  Host  
  <img width="1474" height="759" alt="image" src="https://github.com/user-attachments/assets/a96254a4-8340-4435-aa68-2cc6462e9b7e" />

- 故障中：  
  故障注入：sudo ip link set ens33 down  
  檢查命令：ip link show ens33  
  輸出：顯示 state DOWN  
  症狀：Host 端執行 ssh app 出現 Connection timed out，ping 無回應  
  app  
  <img width="807" height="184" alt="image" src="https://github.com/user-attachments/assets/1fcdb165-c550-4438-8877-4e1b880b4a6e" />
  Host  
  <img width="1477" height="314" alt="image" src="https://github.com/user-attachments/assets/8accea82-24fd-4d34-8fa1-2572cb6222ef" />

- 回復後：  
  回復注入：sudo ip link set ens33 up  
  檢查命令：ip link show ens33  
  輸出：顯示 state UP  
  驗證：Host 端 SSH 恢復連線  
  app  
  <img width="802" height="174" alt="image" src="https://github.com/user-attachments/assets/e2e6a34b-a0cb-4d14-ac69-2337f8c9ae36" />
  Host  
  <img width="1480" height="760" alt="image" src="https://github.com/user-attachments/assets/00e271d6-b6b9-4749-8f73-04a7dc70f03b" />

- 診斷推論：  
  Host 端 ping 不通，代表基礎網路層 (L3) 斷聯。  
  檢查 App 端 ip link 發現狀態為 DOWN，確認為實體介面被禁用。  
  此故障導致封包無法從網卡進出，故呈現連線逾時。  

### 故障 2：<另一個>
F3：Docker 服務停止
- 注入方式：sudo systemctl stop docker/sudo systemctl stop docker docker.socket  
- 故障前：  
  檢查命令：systemctl status docker  
  輸出：顯示 Active: active (running)  
  驗證：執行 docker ps 可列出運行中的容器  
  app  
  <img width="810" height="373" alt="image" src="https://github.com/user-attachments/assets/4e661f22-6fc6-42db-8046-77431dd5b819" />
  Host  
  <img width="1475" height="728" alt="image" src="https://github.com/user-attachments/assets/307ad378-a9f3-4749-ae2b-913e8d82c4fd" />

- 故障中：  
  故障注入：sudo systemctl stop docker/sudo systemctl stop docker docker.socket  
  檢查命令：sudo docker ps  
  輸出：Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?  
  症狀：Host 端 可以正常 SSH 登入，但無法操作 Docker  
  sudo systemctl stop docker  
  app  
  <img width="805" height="156" alt="image" src="https://github.com/user-attachments/assets/8e13f9ee-9ab0-41d7-b5b2-714127b6d4d1" />
  Host  
  <img width="1479" height="761" alt="image" src="https://github.com/user-attachments/assets/2ebb0936-e46a-4997-8192-17de3aed9170" />
  
  sudo systemctl stop docker docker.socket  
  app  
  <img width="827" height="196" alt="image" src="https://github.com/user-attachments/assets/c0b4c34e-5117-4fa9-bfa3-cfae8f74f6fb" />
  Host  
  <img width="1478" height="799" alt="image" src="https://github.com/user-attachments/assets/82aa2b8c-c3f5-44ca-bce4-21b85f5bc953" />

- 回復後：  
  命令：sudo systemctl start docker/sudo systemctl start docker docker.socket  
  輸出：systemctl status docker 恢復為 active (running)  
  驗證：docker ps 恢復正常  
  sudo systemctl start docker  
  app  
  <img width="807" height="175" alt="image" src="https://github.com/user-attachments/assets/1bea686d-dd2d-4e04-948d-b7a1ac38c342" />
  Host  
  <img width="1478" height="729" alt="image" src="https://github.com/user-attachments/assets/1706ed88-64f4-4fb4-9e6e-9310e29cec79" />
  
  sudo systemctl start docker docker.socket  
  app  
  <img width="809" height="219" alt="image" src="https://github.com/user-attachments/assets/96d026be-f0fc-47ae-b1ad-27475332abb9" />
  Host  
  <img width="1480" height="798" alt="image" src="https://github.com/user-attachments/assets/981be8c7-a571-4dc0-a382-ccaab9cbda6c" />

- 診斷推論：  
  Host 端能透過 SSH 登入，代表 L2/L3/L4 網路層與防火牆完全正常。  
  執行 docker ps 報錯，指向特定的應用服務 (Service) 失效。  
  透過 systemctl 發現服務為 inactive，確認故障層級在應用層而非網路層。  

### 症狀辨識（若選 F1+F2 必答）
兩個都 timeout，我怎麼分？

## 6. 反思（200 字）
這次做完，對「分層隔離」或「timeout 不等於壞了」的理解有什麼改變？  
網路排錯中 「Timeout 不代表機器壞了，而是溝通在某一層斷了」。  
在 F1 演練中，介面關閉導致 ssh timeout，這時主機在網路上徹底消失，連基礎的 ping 都無法回應。  
但在 F3 演練中，我發現即便 docker ps 報錯，我依然能透過 SSH 順利登入。  
這讓我明白「分層隔離」的診斷價值：SSH 的通暢代表了 L2 物理層到 L4 傳輸層都是好的，問題被限縮在「應用服務層」。  
這次經驗改變了我過往的排錯習慣。現在我知道，當遇到連線問題時，應先用 ping 檢查路徑（L3），再用 telnet/nc 測試端口（L4），最後才進系統看 systemctl 狀態（Service）。  
理解這些層級的差異，讓我能從混亂的報錯訊息中，快速定位出到底是「路不通」還是「冰箱沒電」。  
## 7. Bonus（選做）
