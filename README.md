# Firecracker MicroVM + Kubernetes 容器虛擬化整合


## Concept Development

現代雲端運算中，容器化技術雖然提供了輕量級的隔離環境，但傳統容器共享主機 kernel，安全隔離性有限。本專題旨在整合 Firecracker MicroVM、MicroK8s 和 Kata Containers，實現：

1. MicroVM：使用 Firecracker 啟動 MicroVM。
2. Kubernetes 支援：透過 MicroK8s 統一管理 MicroVM 化的 Pod。
3. 隔離與安全：每個 Pod 運行在獨立的 MicroVM 中，而非共享宿主機 kernel。
4. Hypervisor 架構：將 Kata Containers 預設的 QEMU 虛擬化引擎，替換為專為雲原生設計的 Firecracker MicroVM。

## Implementation Resources

- 作業系統：Ubuntu 24.04 LTS 或以上版本(目前只在24.04測試過)
- CPU：必須支援 KVM 硬體虛擬化（具備 `vmx` 或 `svm` flag）
- 記憶體：至少 8GB（建議 16GB 以上）
- 磁碟空間：至少 50GB 可用空間

## Existing Library/Software

| 軟體/模組名稱 | 參考版本 | 系統角色 |
| :--- | :--- | :--- |
| **Firecracker** | v1.15.1 | **VMM（虛擬機監控器）** |
| **Kata Containers** | v3.31 | **容器虛擬化安全隔離層**：作為 Container 與 MicroVM 的中間橋樑 |
| **MicroK8s** | v1.34.5 | **輕量化 Kubernetes 叢集**：負責整體 Pod 的生命週期管理、網路調度與服務暴露（Service）。 |
| **containerd** | Snap 內建版 | **Container Runtime**：K8s 控管容器的直接接口，透過修改其 TOML 模板來切換底層儲存後端。 |
| **Docker Engine** | 20.10+ 或最新 | **基礎容器引擎**：用於前期建置對照組。 |
| **KVM** | Linux 內建 | **核心虛擬化模組**：Linux 核心的硬體虛擬化驅動，為 Firecracker 提供底層硬體加速。 |
| **device-mapper** | Linux 核心內建 | **儲存對應子系統**：負責在底層建立 thin-pool，將檔案模擬成 Firecracker 唯一看得懂的Block device。 |
| **util-linux (`dmsetup`/`losetup`)** | Ubuntu 內建 | **區塊裝置管理工具**：提供 `losetup`（掛載虛擬磁碟檔案）與 `dmsetup`（配置與啟動 thin-pool 結構）的命令支援。 |
| **Nginx** | Docker Hub image | **測試與驗證應用**：作為專案中的網頁伺服器範例，用來驗證 MicroVM Pod 內部的獨立 Kernel 版本以及外部網路連線。 |

## Implementation Process 
開發歷程分為四個階段：

1. Firecracker 手動部署：初步下載 Firecracker 二進制檔，透過 REST API 發送 JSON 格式指令，成功配置 Kernel 與 rootfs 並實現啟動，驗證其底層運作原理。
2. MicroK8s 與 Docker 基礎建置：建立標準的 Kubernetes 測試環境，透過一般 nginx Pod 確認容器與宿主機共用 Kernel (例如 6.8.0 版本)。
3. Kata Containers + QEMU 整合驗證：導入 Kata Containers 框架。起初使用預設的 QEMU 模式，成功讓 Pod 跑在獨立的 MicroVM 中（核心版本變為 6.18.28），確認架構可行。
4. Kata Containers + Firecracker 整合：
- 遭遇問題：直接將 Kata 切換為 Firecracker 時發生 ENOENT 錯誤。原因在於 containerd 預設使用 overlayfs，但 Firecracker 僅支援 Block Device。
- 解決方案：引入 devmapper snapshotter 。我們建立了 Device Mapper Thin-Pool，並修改/var/snap/microk8s/current/args/containerd-template.toml，成功讓 Kubernetes 為 Firecracker 自動生成並掛載虛擬硬碟，完成整合。

## Installation

### 1. 環境檢查與基礎軟體安裝
```
# 確認 CPU 支援虛擬化 (有輸出代表支援)
grep -E "vmx|svm" /proc/cpuinfo

# 安裝 Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# 安裝 Firecracker (v1.15.1)
cd ~
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker/v1.15.1/x86_64/firecracker
chmod +x firecracker
sudo mv firecracker /usr/local/bin/
```
### 2. 安裝 MicroK8s 與 Kata Containers
```
# 安裝 MicroK8s 並啟用必要插件
sudo snap install microk8s --classic
sudo microk8s enable dns ha-cluster helm helm3 community kata

# 下載並安裝 Kata 3.31 靜態包
cd ~
wget https://github.com/kata-containers/kata-containers/releases/download/3.31.0/kata-static-3.31.0-amd64.tar.zst
sudo tar -I zstd -xf kata-static-3.31.0-amd64.tar.zst -C /
```
### 3. 配置 Devmapper

因為 Firecracker 需要區塊裝置，我們必須配置 devmapper thin-pool。
```
# 建立存放 devmapper 的目錄
sudo mkdir -p /var/lib/containerd-devmapper
sudo mkdir -p /var/snap/microk8s/common/var/lib/containerd-devmapper

# 建立 20GB 資料檔與 2GB 元資料檔並掛載 Loop 設備
sudo truncate -s 20G /var/lib/containerd-devmapper/data
sudo truncate -s 2G /var/lib/containerd-devmapper/metadata
sudo losetup -f /var/lib/containerd-devmapper/data
sudo losetup -f /var/lib/containerd-devmapper/metadata

# 建立 thin-pool (注意：這裡的 loop4 與 loop5 請依據上方指令實際產生的編號修改)
sudo dmsetup create containerd-thinpool \
  --table "0 $(sudo blockdev --getsz /dev/loop4) thin-pool /dev/loop5 /dev/loop4 128 32768"
```
### 4. 修改 Containerd 設定並切換為 Firecracker
```
# 修改 containerd 模板，將 snapshotter 改為 devmapper
sudo sed -i 's/snapshotter = "${SNAPSHOTTER}"/snapshotter = "devmapper"/' \
  /var/snap/microk8s/current/args/containerd-template.toml

# 將 devmapper 設定寫入設定檔底部
sudo tee -a /var/snap/microk8s/current/args/containerd-template.toml << 'EOF'

[plugins."io.containerd.snapshotter.v1.devmapper"]
  pool_name = "containerd-thinpool"
  root_path = "/var/snap/microk8s/common/var/lib/containerd-devmapper"
  base_image_size = "8192MB"
  discard_blocks = true
EOF

# 切換 Kata 設定檔為 Firecracker 模式
sudo cp /opt/kata/share/defaults/kata-containers/configuration-fc.toml \
  /opt/kata/share/defaults/kata-containers/configuration.toml

# 重啟 MicroK8s 套用設定
sudo microk8s stop && sudo microk8s start
microk8s status --wait-ready
```
## Usage
環境建置完成後，來部署一個以 Firecracker MicroVM 運行的 Nginx 伺服器

### 1. 部署 Kata Pod
```
# 建立一個使用 kata runtime 的 Pod
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kata
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx
EOF

# 等待狀態變成 Running
microk8s kubectl get pods
```

### 2. 驗證隔離性與 Firecracker
```
# 查看宿主機 (Host) 的 Kernel 版本 (例如 6.8.x)
uname -r

# 查看 Pod 內的 Kernel 版本，你會發現它完全不同 (例如 6.18.28)，證明它跑在獨立虛擬機中！
microk8s kubectl exec -it nginx-kata -- uname -r

# 確認正在使用 firecracker
/opt/kata/bin/kata-runtime kata-env 2>/dev/null | grep "firecracker"
```

### 3. 對外開放服務
```
# 建立 NodePort Service 讓外部可以連線
microk8s kubectl expose pod nginx-kata --type=NodePort --port=80

# 查看被分配到的 Port 號碼 (30000~32767之間)
microk8s kubectl get svc

# 測試連線 (請將 <NodePort> 換成上一步查詢到的號碼)
curl http://localhost:<NodePort>
```

## References 
https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-kata-containers-with-firecracker.md



