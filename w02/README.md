# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | （填入）192.168.11.133 | 上網 |
| dev-a | NIC 2 | Host-only | （填入）192.168.75.128 | 內網互連 |
| server-b | NIC 1 | Host-only | （填入）192.168.75.129 | 內網互連 |

## 連線驗證紀錄

- [ ] dev-a NAT 可上網：`ping google.com` 輸出
<img width="804" height="311" alt="image" src="https://github.com/user-attachments/assets/7aee26e3-c921-4aa4-b04e-c5311327f7f2" />

- [ ] 雙向互 ping 成功：貼上雙方 `ping` 輸出

在 dev-a 上 ping server-b
<img width="637" height="217" alt="image" src="https://github.com/user-attachments/assets/191c20b6-6075-4fd4-9f60-3e988cd7af6a" />

在 server-b 上 ping dev-a
<img width="651" height="220" alt="image" src="https://github.com/user-attachments/assets/f1cf9340-cb57-4645-84c8-01b53cdc75c1" />

- [ ] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
<img width="509" height="88" alt="image" src="https://github.com/user-attachments/assets/b2bf1f7d-b969-4b63-9082-4cd956eb2f89" />

- [ ] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
<img width="455" height="38" alt="image" src="https://github.com/user-attachments/assets/8d0ba72d-3366-43ba-934b-7784d45d0566" />

- [ ] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
<img width="794" height="113" alt="image" src="https://github.com/user-attachments/assets/1d868ea1-3c82-4e72-b6f7-5d48f7f65845" />

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | （填入）UP |
| dev-a ping server-b | 成功 | 失敗 | （填入）成功 |
| dev-a SSH server-b | 成功 | 失敗 | （填入）成功 |
<img width="778" height="528" alt="image" src="https://github.com/user-attachments/assets/7ec26659-67fe-477b-995a-e5bba7471f5d" />
<img width="804" height="657" alt="image" src="https://github.com/user-attachments/assets/429b6fad-0cbe-4a85-82ff-0496db44a1fa" />
<img width="805" height="240" alt="image" src="https://github.com/user-attachments/assets/f53ef495-8aed-4d78-b506-852fd34c409a" />
<img width="664" height="326" alt="image" src="https://github.com/user-attachments/assets/7bbfea63-6f83-4b4e-aefb-fda713511938" />

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （填入）有監聽 |
| dev-a ping server-b | 成功 | 成功 | （填入）成功 |
| dev-a SSH server-b | 成功 | Connection refused | （填入）成功 |
<img width="683" height="196" alt="image" src="https://github.com/user-attachments/assets/d0a3e587-890b-471a-9d9d-9749e26b7c59" />
<img width="773" height="216" alt="image" src="https://github.com/user-attachments/assets/df995d00-ee77-48fc-921f-dc014cf747c6" />
<img width="496" height="67" alt="image" src="https://github.com/user-attachments/assets/42c2166a-5319-43f2-9755-3e16e247da95" />

## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）
L2（Data Link Layer）
先確認網卡、交換器、虛擬網卡是否正常

指令：
ip link
ip addr
nmcli device status
ethtool ens33
ping 127.0.0.1

檢查重點：
- 網卡是否為 UP 狀態
- 是否有 MAC Address
- 是否有拿到 IP
- 線路或虛擬網卡是否斷開


L3（Network Layer）
確認 IP、路由、Gateway 是否正常

指令：
ip route
ping <gateway IP>
ping 8.8.8.8
traceroute 8.8.8.8
netstat -rn

檢查重點：
- 是否有正確的預設閘道
- 是否能 ping 到 Gateway
- 是否能連到外部 IP
- 路由是否正確


L4（Transport Layer）
確認 TCP/UDP Port 是否有開啟、服務是否有監聽

指令：
ss -tulnp
netstat -tulnp
telnet <IP> 22
nc -zv <IP> 22
sudo systemctl status ssh

檢查重點：
- Port 22 是否正在 Listen
- SSH 服務是否啟動
- 防火牆是否阻擋
- TCP 連線是否成功

## 網路拓樸圖
（嵌入或連結 network-diagram.png）
<img width="481" height="491" alt="network-diagram" src="https://github.com/user-attachments/assets/d11ae772-5f80-4e6c-935a-52a6040fcc28" />

## 排錯紀錄
- 症狀：dev-a 原本可以 ping 與 SSH 到 server-b，但在故障演練後突然無法連線。
- 診斷：（你首先查了什麼？用了哪個命令？）
我先從 L2 開始檢查 server-b 的網卡狀態，使用：
ip link
ip addr
nmcli device status
發現 Host-only 網卡介面狀態變成 DOWN，因此 dev-a 無法 ping 到 192.168.75.129。
接著在 SSH 故障演練中，我使用：
ss -tlnp | grep :22
sudo systemctl status ssh
發現 Port 22 沒有在 Listen，表示 SSH 服務被停止。
- 修正：（做了什麼改動？）針對網卡停用問題，重新啟用介面：
sudo ip link set ens34 up
或使用：
sudo nmcli device connect ens34
針對 SSH 問題，重新啟動 SSH 服務：
sudo systemctl start ssh
- 驗證：（如何確認修正有效？）
重新確認：
ping -c 4 192.168.75.129
ssh gina@192.168.75.129 "hostname"
ss -tlnp | grep :22
結果顯示 ping 成功、SSH 可正常登入、Port 22 重新開始監聽，代表問題已修正。
## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 server-b 只設 Host-only 不給 NAT？）

選擇讓 server-b 只使用 Host-only 網路，不額外配置 NAT，原因如下：
server-b 的角色是內部伺服器，只需要提供 SSH 與檔案傳輸給 dev-a 使用，不需要直接連上網際網路。
只使用 Host-only 可以降低暴露風險，避免伺服器直接對外連線或被外部存取。
若未來需要模擬公司內網、跳板機、資料庫伺服器等情境，Host-only 比較符合真實環境。
NAT 只配置在 dev-a，代表 dev-a 可以同時扮演「可上網的工作站」與「內網管理端」，架構較清楚。
因此這次配置形成：
dev-a：同時有 NAT + Host-only，可上網也可管理內網
server-b：只有 Host-only，只能被內網存取，不能直接上網
這樣的設計兼顧了功能性與安全性。
