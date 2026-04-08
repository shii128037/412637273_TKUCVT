# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | （填入）192.168.11.133 | 上網 |
| dev-a | NIC 2 | Host-only | （填入）192.168.75.128 | 內網互連 |
| server-b | NIC 1 | Host-only | （填入）192.168.75.129 | 內網互連 |

## 連線驗證紀錄

- [ ] dev-a NAT 可上網：`ping google.com` 輸出
      ping -c 4 google.com
PING google.com (142.250.196.206) 56(84) bytes of data.
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=1 ttl=128 time=208 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=2 ttl=128 time=16.1 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=3 ttl=128 time=35.7 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=4 ttl=128 time=14.3 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 14.270/68.562/208.166/81.034 ms
- [ ] 雙向互 ping 成功：貼上雙方 `ping` 輸出
在 dev-a 上 ping server-b
      ping -c 4 192.168.75.129
PING 192.168.75.129 (192.168.75.129) 56(84) bytes of data.
64 bytes from 192.168.75.129: icmp_seq=1 ttl=64 time=1.18 ms
64 bytes from 192.168.75.129: icmp_seq=2 ttl=64 time=0.881 ms
64 bytes from 192.168.75.129: icmp_seq=3 ttl=64 time=5.28 ms
64 bytes from 192.168.75.129: icmp_seq=4 ttl=64 time=1.32 ms

--- 192.168.75.129 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.881/2.165/5.275/1.802 ms

在 server-b 上 ping dev-a
ping -c 4 192.168.75.128
PING 192.168.75.128 (192.168.75.128) 56(84) bytes of data.
64 bytes from 192.168.75.128: icmp_seq=1 ttl=64 time=0.846 ms
64 bytes from 192.168.75.128: icmp_seq=2 ttl=64 time=1.13 ms
64 bytes from 192.168.75.128: icmp_seq=3 ttl=64 time=1.62ms
64 bytes from 192.168.75.128: icmp_seq=4 ttl=64 time=0.915 ms

--- 192.168.75.128 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3044ms
rtt min/avg/max/mdev = 0.846/1.129/1.623/0.304 ms

- [ ] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
      gina@server-b:~$ hostname
server-b

- [ ] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
      gina@server-b:~$ cat /tmp/test-from-dev.txt
Hello from dev-a

- [ ] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
gina@server-b:~$ ping -c 4 8.8.8.8
ping: connect: Network is unreachable

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
- 症狀：
- 診斷：（你首先查了什麼？用了哪個命令？）
- 修正：（做了什麼改動？）
- 驗證：（如何確認修正有效？）

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 server-b 只設 Host-only 不給 NAT？）
