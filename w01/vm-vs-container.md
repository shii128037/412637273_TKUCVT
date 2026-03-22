# 至少四個維度的 VM vs Container 對照
維度	| VM（Virtual Machine）|	Container
虛擬化層級 |	硬體層虛擬化（透過 Hypervisor）|	作業系統層虛擬化
啟動速度 |	較慢（需啟動完整 OS）|	很快（秒級啟動）
資源使用 |	較高（每個 VM 都有 OS）|	較低（共用 Host OS）
隔離性 |	高（完全隔離）|	中（共享 kernel）
可攜性 |	較低（映像較大）|	高（輕量化 image）
適用場景|	多 OS 測試、強隔離需求|	微服務、快速部署

# 本課選擇「VM 裡跑 Docker」的理由（用自己的話寫）
環境隔離
  VM 提供完整的系統隔離，避免影響本機（Host）環境
  即使操作錯誤（例如刪除系統檔），也只影響 VM
可回復性（Snapshot）
  VM 支援 Snapshot，可以快速回復到特定狀態
  在故障演練中（例如移除 Docker repository），能快速還原
實驗安全性
  Container 雖然輕量，但共享 Host OS kernel
  若直接在本機跑 Docker，錯誤操作可能影響整個系統
  
# Hypervisor Type 1 vs Type 2 的差異與本課的選擇
Type 1（Bare-metal Hypervisor）
項目	說明
安裝位置	直接安裝在硬體上
效能	較高
安全性	較高
常見例子	VMware ESXi、Microsoft Hyper-V、Xen

Type 2（Hosted Hypervisor）
項目	說明
安裝位置	安裝在作業系統上
效能	較低（多一層 OS）
使用難度	較低
常見例子	VMware Workstation、VirtualBox

本課的選擇
使用的是 Type 2 Hypervisor（例如 VMware Workstation 或 VirtualBox）。
