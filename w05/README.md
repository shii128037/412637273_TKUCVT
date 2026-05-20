# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：systemd
- Cgroup Driver：2
- Default Runtime：runc
<img width="841" height="175" alt="image" src="https://github.com/user-attachments/assets/55be3400-abac-4f6a-9903-b56810259741" />

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：看得到的進程不一樣。讓容器擁有自己獨立的進程編號系統，所以容器裡的啟動程式可以自豪地當 PID 1（老大），看不到 Host 上的其他程式。
- NET：用的網路不一樣。每個容器都有自己獨立的虛擬網卡、IP 位址、路由表和防火牆（iptables），互不干擾，也不會搶佔 Host 的連接埠。
- MNT：看到的檔案系統不一樣。檔案掛載點是隔離的。容器有自己獨立的根目錄（/），它在裡面怎麼切換、掛載硬碟，Host 和其他容器都看不到。
- UTS：名字和網域不一樣。隔離了主機名稱（Hostname）和網域名稱（Domainname）。你可以把容器的主機名改成 web-server，而不會影響到 Host 原本的名字。
- IPC：溝通的管道不一樣。隔離進程間的通訊管道（如共享記憶體、信號佇列）。容器內的程式只能跟同一個容器內的同伴「講悄悄話」，無法與外部進程共享記憶體。
- USER：認定的身份不一樣。隔離使用者與群組 ID。最厲害的是，容器內的 root 使用者（UID 0），在 Host 視角看來可能只是一個沒有特權的普通小職員，安全防護極佳。

### Host vs 容器 inode 對照
（貼上或連結 `namespace-table.md`）
| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | 4026531836 | 4026532675 | 不一樣 (已隔離) |
| net | 4026531833 | 4026532677 | 不一樣 (已隔離) |
| mnt | 4026531832 | 4026532634 | 不一樣 (已隔離) |
| uts | 4026531838 | 4026532671 | 不一樣 (已隔離) |
| ipc | 4026531839 | 4026532674 | 不一樣 (已隔離) |
| user | 4026531837 | 4026531837 | 一樣 (未隔離) |

### 容器內 `ps aux` 輸出
（只看到幾支 process？為什麼？）
<img width="865" height="724" alt="image" src="https://github.com/user-attachments/assets/90379f51-c187-4921-8fa0-c1a7f8245a03" />
- 3 支
- 1. 獨立的進程視角（PID Namespace 隔離）    
     .看不見 Host：核心（Kernel）為這個容器切換了全新的 PID Namespace。這意味著 Host 上的幾百支系統進程（如 systemd、sshd 等）在容器內被完全隱藏。  
     .自己當老大：容器啟動時的初始指令 sleep 3600，在容器世界裡被核心賦予了最高地位的 PID 1。  
  2. 獨立的 /proc 檔案系統（MNT Namespace 隔離）  
     .Linux 的 ps 指令本質上是去讀取 /proc 目錄下的進程資訊（例如 /proc/1/、/proc/7/）。  
     .因為容器同時做了 Mount (MNT) Namespace 隔離，容器擁有自己獨立的 /proc 掛載點。  
     .當您在容器內執行 ps aux 時，它只會讀到該容器內部專屬的 /proc 檔案，因此絕對不會看到任何外部主機的進程活動。  
## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：401408

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | OOM Killed 或 oom-kill | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
3 層  

### 兩個同源 image 共享 layer 的證據
（前幾個 sha256 是否相同？）
否  
<img width="817" height="438" alt="image" src="https://github.com/user-attachments/assets/5f92e238-da21-4545-a1c1-5d7644d1655f" />

### `docker diff` 輸出範例與解讀
（貼上 A/C/D 實例並說明）
<img width="805" height="269" alt="image" src="https://github.com/user-attachments/assets/e1e20900-21cf-4037-9eb1-4290da6bdd1d" />
| 標記 | 意思      | 範例       |
| -- | ------- | -------- |
| A  | Added   | 新增檔案     |
| C  | Changed | 檔案或目錄被修改 |
| D  | Deleted | 檔案被刪除    |

## OCI 呼叫鏈

（用自己的話說明 dockerd → containerd → containerd-shim → runc 各自負責什麼，以及 OCI Runtime Spec `config.json` 裡哪些欄位對應到 namespace / cgroup 設定）  
容器呼叫鏈的分工角色  
dockerd (頂層管理者)  
.負責與使用者互動，接收你的 docker run 或 docker stop 指令。  
.管理高層級的容器概念，例如：設定網路、管理磁碟卷（Volume）、建置與下載映像檔（Image）。  
.它不直接操作容器，而是把指令整理好後，丟給下一層。  
containerd (中層調度員)  
.負責管理容器的生命週期（建立、啟動、暫停、銷毀）。  
.它是一個符合工業標準的容器執行緒，負責管理映像檔解壓、監督容器狀態。  
.雖然它管生命週期，但它依然不是「真正去執行」容器的人，它會呼叫下層的 shim。  
containerd-shim (隱形守護者)  
.關鍵作用：解耦。 它是 containerd 和底層 runc 之間的墊片。  
.當 runc 把容器跑起來後就會立刻退出，此時 shim 就會留下來，接管該容器的 stdout、stderr 日誌輸出，並收集容器的退出狀態碼。  
.有了它，即使 dockerd 或 containerd 崩潰或重啟，容器依然可以正常運行，不會跟著死掉。  
runc (底層建造者)  
.真正動手建立容器的「工具人」，完全符合 OCI Runtime Spec 標準。  
.它只負責一件事：讀取設定檔，呼叫 Linux 核心的 unshare()、clone() 和 cgroups 機制，把隔離環境（容器）蓋好。  
.容器啟動成功後，runc 的任務就結束並直接退出。  

OCI Runtime Spec config.json 欄位對應  
runc 在建立容器時，完全依賴 config.json 這個設定檔。其中關於 Linux Namespace (隔離) 與 Cgroup (限制) 的設定，主要對應到以下欄位：  

1. Namespace (命名空間/隔離)  
對應到 json 中的 linux.namespaces 欄位。這是一個陣列，決定容器要跟主機隔離哪些資源。
json{
  "linux": {
    "namespaces": [
      { "type": "pid" },     // 進程隔離 (看不到主機和其他容器的 PID)
      { "type": "network" }, // 網路隔離 (擁有獨立的網路卡、IP 和路由表)
      { "type": "mount" },   // 掛載隔離 (擁有獨立的檔案系統掛載點)
      { "type": "ipc" },     // 進程間通訊隔離
      { "type": "uts" },     // 主機名稱與網域名稱隔離
      { "type": "user" }     // 使用者與群組 ID 隔離
    ]
  }
}
2. Cgroup (資源限制)  
對應到 json 中的 linux.resources 欄位。這裡定義了容器最多能消耗多少硬體資源。
json{
  "linux": {
    "resources": {
      "memory": {
        "limit": 536870912,       // 限制記憶體最大 512MB (Memory Cgroup)
        "swap": 1073741824        // 限制 Swap 空間 (Memory Cgroup)
      },
      "cpu": {
        "shares": 1024,           // CPU 權重分配 (CPU Cgroup)
        "quota": 50000,           // 在一個週期內最多可用多少微秒的 CPU 時間
        "period": 100000          // CPU 週期，與 quota 搭配決定 CPU 使用率上限
      },
      "pids": {
        "limit": 100              // 限制容器內最多隻能建立 100 個進程 (PIDs Cgroup)
      }
    }
  }
}

## 排錯紀錄
- 症狀：container 無法連外：ping 8.8.8.8失敗。
- 診斷：檢查：docker network inspect bridgebridge interface 正常 但 host 沒有 NAT 規則 查看：iptables -t nat -L 發現 Docker MASQUERADE 規則消失。
- 修正：重開 NAT
- 驗證：重新測試：docker run alpine ping 8.8.8.8成功收到回應。

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？  
.不是。  container 有自己的 PID namespace。  
.會殺掉 container 的主 process。
 
２. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？　　

image layer 共用。只有：  writable layer  container metadata  各自獨立。  
驗證：　　
查看：docker system df　　
會看到：SHARED SIZE　　
表示 base image layer 被共享。　　
也可以：　　
docker inspect container1　　
docker inspect container2　　
觀察相同 image ID。　　

３. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？　　

隔離會被削弱。　　
VM：　　
Guest OS　　
↓　　
Guest Kernel　　
↓　　
Hypervisor　　
↓　　
Host　　
每台 VM 有自己的 kernel。　　
因此：　　
guest kernel 與 host kernel 分離　　
隔離較強　　
host kernel 漏洞不一定直接影響 guest　　
而 container：　　
Container　　
↓　　
共用 Host Kernel　　
效能較高，但隔離性弱於 VM。　　
