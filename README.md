# Firecracker MicroVM + Kubernetes 容器虛擬化整合

## Concept Development

現代雲端運算中，容器化技術雖然提供了輕量級的隔離環境，但傳統容器共享主機 kernel，安全隔離性有限。本專題旨在整合 Firecracker MicroVM、MicroK8s 和 Kata Containers，實現：

1. **MicroVM**：使用 Firecracker 啟動 MicroVM。
2. **Kubernetes 支援**：透過 MicroK8s 統一管理 MicroVM 化的 Pod。
3. **隔離與安全**：每個 Pod 運行在獨立的 MicroVM 中，而非共享宿主機 kernel。
4. **Hypervisor 架構**：將 Kata Containers 預設的 QEMU 虛擬化引擎，替換為專為雲原生設計的 Firecracker MicroVM。

---

## Implementation Resources

- 作業系統：Ubuntu 24.04 LTS（目前只在 24.04 測試過）
- CPU：必須支援 KVM 硬體虛擬化（具備 `vmx` 或 `svm` flag）
- 記憶體：至少 8GB（建議 16GB 以上）
- 磁碟空間：至少 50GB 可用空間

---

## Existing Library / Software

| 軟體/模組名稱 | 參考版本 | 系統角色 |
|---|---|---|
| **Firecracker** | v1.10.0 | **VMM（虛擬機監控器）** |
| **Kata Containers** | v3.31.0 | **容器虛擬化安全隔離層**：作為 Container 與 MicroVM 的中間橋樑 |
| **MicroK8s** | v1.28 | **輕量化 Kubernetes 叢集**：負責整體 Pod 的生命週期管理、網路調度與服務暴露 |
| **containerd** | Snap 內建版 | **Container Runtime**：K8s 控管容器的直接接口，透過修改其 TOML 模板來切換底層儲存後端 |
| **KVM** | Linux 內建 | **核心虛擬化模組**：為 Firecracker 提供底層硬體加速 |
| **device-mapper** | Linux 核心內建 | **儲存對應子系統**：建立 thin-pool，將檔案模擬成 Firecracker 唯一支援的 Block Device |
| **util-linux (`dmsetup`/`losetup`)** | Ubuntu 內建 | **區塊裝置管理工具**：掛載虛擬磁碟並配置 thin-pool 結構 |
| **Nginx** | Docker Hub image | **測試與驗證應用**：驗證 MicroVM Pod 內的獨立 Kernel 與外部網路連線 |

---

## Implementation Process

開發歷程分為四個階段：

1. **Firecracker 手動部署**：初步下載 Firecracker 二進制檔，透過 REST API 發送 JSON 格式指令，成功配置 Kernel 與 rootfs 並實現啟動，驗證其底層運作原理。
2. **MicroK8s 與 Docker 基礎建置**：建立標準的 Kubernetes 測試環境，透過一般 nginx Pod 確認容器與宿主機共用 Kernel（例如 6.8.0 版本）。
3. **Kata Containers + QEMU 整合驗證**：導入 Kata Containers 框架，使用預設的 QEMU 模式，成功讓 Pod 跑在獨立的 MicroVM 中（核心版本變為 6.18.28），確認架構可行。
4. **Kata Containers + Firecracker 整合**：遭遇 containerd 使用 overlayfs 但 Firecracker 僅支援 Block Device 的衝突，最終引入 devmapper snapshotter 解決，建立 Device Mapper Thin-Pool 並修改 `containerd-template.toml` 完成整合。

---

## Installation

### 1. 環境檢查

```bash
# 確認 CPU 支援虛擬化（有輸出代表支援）
grep -E "vmx|svm" /proc/cpuinfo
```

---

### 2. 安裝 Firecracker

```bash
# 下載並解壓縮
mkdir -p ~/firecracker-install && cd ~/firecracker-install
wget https://github.com/firecracker-microvm/firecracker/releases/download/v1.10.0/firecracker-v1.10.0-x86_64.tgz
tar -xvf firecracker-v1.10.0-x86_64.tgz

# 移至專屬資料夾並賦予執行權限
mkdir -p ~/firecracker-bin
mv release-v1.10.0-x86_64/firecracker-v1.10.0-x86_64 ~/firecracker-bin/firecracker
mv release-v1.10.0-x86_64/jailer-v1.10.0-x86_64 ~/firecracker-bin/jailer
chmod +x ~/firecracker-bin/firecracker ~/firecracker-bin/jailer
cd ~ && rm -rf ~/firecracker-install

# 建立軟連結至系統路徑
sudo ln -sf ~/firecracker-bin/firecracker /usr/local/bin/firecracker
sudo ln -sf ~/firecracker-bin/jailer /usr/local/bin/jailer

# 驗證
firecracker --version
```

---

### 3. 安裝 MicroK8s

```bash
sudo snap install microk8s --classic --channel=1.28/stable
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status --wait-ready
```

> **Ubuntu 24.04 注意**：在啟用 community 插件庫之前，需先設定 Git 信任清單，否則 Snap 沙盒機制會讓 MicroK8s 內部的 git clone 被擋下。

```bash
git config --global --add safe.directory '*'
sudo git config --global --add safe.directory '*'
microk8s enable community
```

---

### 4. 安裝 Kata Containers

```bash
cd ~
wget https://github.com/kata-containers/kata-containers/releases/download/3.31.0/kata-static-3.31.0-amd64.tar.zst
sudo apt-get update && sudo apt-get install zstd -y
sudo tar -I zstd -xf kata-static-3.31.0-amd64.tar.zst -C /
rm kata-static-3.31.0-amd64.tar.zst
```

---

### 5. 配置 Devmapper

Firecracker 只支援 Block Device，無法使用 Kubernetes 預設的 overlayfs，需建立 Device Mapper thin-pool 作為儲存後端。

```bash
# 建立目錄與虛擬磁碟檔
sudo mkdir -p /var/lib/containerd-devmapper
sudo mkdir -p /var/snap/microk8s/common/var/lib/containerd-devmapper
sudo truncate -s 20G /var/lib/containerd-devmapper/data
sudo truncate -s 2G /var/lib/containerd-devmapper/metadata

# 掛載為 Loop 設備
sudo losetup -f /var/lib/containerd-devmapper/data
sudo losetup -f /var/lib/containerd-devmapper/metadata
```

> **重要**：每台機器分配到的 loop 編號不同，執行以下指令確認後再繼續：

```bash
losetup -a
# 範例輸出：
# /dev/loop3: [xxx] (/var/lib/containerd-devmapper/data)
# /dev/loop4: [xxx] (/var/lib/containerd-devmapper/metadata)
```

將下方指令中的 `loop3`（data）與 `loop4`（metadata）替換為你實際看到的編號：

```bash
sudo dmsetup create containerd-thinpool \
  --table "0 $(sudo blockdev --getsz /dev/loop3) thin-pool /dev/loop4 /dev/loop3 128 32768"

# 驗證
sudo dmsetup ls
# 應看到：containerd-thinpool  (253:0)
```

---

### 6. 修改 Containerd 設定並切換為 Firecracker

```bash
# 將 snapshotter 改為 devmapper
sudo sed -i 's/snapshotter = "${SNAPSHOTTER}"/snapshotter = "devmapper"/' \
  /var/snap/microk8s/current/args/containerd-template.toml

# 追加 devmapper 設定區塊
sudo tee -a /var/snap/microk8s/current/args/containerd-template.toml << 'EOF'

[plugins."io.containerd.snapshotter.v1.devmapper"]
  pool_name = "containerd-thinpool"
  root_path = "/var/snap/microk8s/common/var/lib/containerd-devmapper"
  base_image_size = "8192MB"
  discard_blocks = true
EOF

# 切換 Kata 設定為 Firecracker 模式
sudo cp /opt/kata/share/defaults/kata-containers/configuration-fc.toml \
  /opt/kata/share/defaults/kata-containers/configuration.toml

# 重啟 MicroK8s
sudo microk8s stop && sudo microk8s start
microk8s status --wait-ready
```

---

### 7. 建立 Kata 相關軟連結

MicroK8s 的 containerd 會在系統 `$PATH` 中尋找 `containerd-shim-kata-v2`，但 Kata 靜態包安裝在 `/opt/kata/bin/`，需手動建立軟連結橋接。

> **若跳過此步驟**，Pod 會永遠卡在 `ContainerCreating`，並出現 `"containerd-shim-kata-v2": file does not exist` 錯誤。

```bash
sudo ln -sf /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/containerd-shim-kata-v2
sudo ln -sf /opt/kata/bin/kata-runtime /usr/local/bin/kata-runtime
sudo ln -sf /opt/kata/bin/kata-monitor /usr/local/bin/kata-monitor
```

---

### 8. 啟用 Kata 插件與宣告 RuntimeClass

```bash
microk8s enable kata --runtime-path=/opt/kata/bin

cat <<EOF | microk8s kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
EOF
```

---

## Usage

### 1. 部署 Kata Pod

```bash
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kata
  labels:
    app: nginx
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx
EOF

microk8s kubectl get pods -w
```

### 2. 驗證隔離性

```bash
# 宿主機 Kernel（例如 6.8.x）
uname -r

# Pod 內 Kernel（應與宿主機不同，代表跑在獨立 MicroVM 中）
microk8s kubectl exec -it nginx-kata -- uname -r

# 確認使用 Firecracker
/opt/kata/bin/kata-runtime kata-env 2>/dev/null | grep "firecracker"
```

### 3. 對外開放服務

```bash
microk8s kubectl expose pod nginx-kata \
  --type=NodePort --port=80 --target-port=80 \
  --name=nginx-kata-service

# 查看分配到的 Port
microk8s kubectl get svc nginx-kata-service

# 測試連線（將 <NodePort> 換成上方查到的號碼）
curl http://localhost:<NodePort>
```

---

## Troubleshooting

| 錯誤訊息 | 原因 | 解法 |
|---|---|---|
| `404 Not Found` on wget | AWS S3 下載連結已失效 | 使用步驟 2 的 GitHub 連結 |
| `fatal: detected dubious ownership` | Ubuntu 24.04 Snap + Git 安全機制衝突 | 步驟 3 執行 `git config safe.directory '*'` |
| `device-mapper: reload ioctl failed` | loop 設備編號填錯 | `losetup -a` 確認實際編號後再建立 thinpool |
| Pod 卡在 `ContainerCreating` | 系統找不到 `containerd-shim-kata-v2` | 步驟 7 執行軟連結指令 |
| `error: selector is required` | Pod 缺少 Label | 確認 YAML 中有 `labels: app: nginx` |

---

## References

- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-kata-containers-with-firecracker.md
- https://github.com/firecracker-microvm/firecracker
- https://microk8s.io/docs
