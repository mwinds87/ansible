這是一個非常詳盡且專業的 **Ansible Playbook** 集合，用於在 **Ubuntu** 系統上實現 **Kubernetes 1.34.1** 集群的**離線（Air-gapped）安裝與部署**。

我將根據您提供的 Ansible Playbook 內容，撰寫一份結構化、完整的技術文件（README）。

-----

# Kubernetes $v1.34.1$ 離線部署技術文件 (基於 Ansible 與 Calico Operator)

## 簡介

本文件詳細說明如何使用 **Ansible Playbook** 來實現 **Kubernetes (K8s) $v1.34.1$** 集群的離線部署。本流程分為兩個主要階段：

1.  **物料準備與打包**：在一個有網際網路連線的機器上，下載所有必需的 **.deb 套件（Docker, K8s 核心）**、\*\*容器鏡像（K8s 核心與 Calico）\*\*以及 **CNI 網路配置檔案**，並打包成單一資源包。
2.  **集群離線安裝與部署**：將資源包傳輸到目標離線節點（Master/Worker），並執行安裝、系統配置、鏡像載入與集群初始化。

本方案採用 **Calico Operator** 作為網絡插件的部署方式。

## 先決條件

### 軟體與環境要求

| 項目 | 描述 | 備註 |
| :--- | :--- | :--- |
| **Ansible 控制機** | 已安裝 Ansible $2.9$ 或更高版本。 | 用於執行 Playbook。 |
| **操作系統** | 目標節點需為 **Ubuntu 20.04/22.04** 或兼容版本。 | Playbook 內的 **APT** 倉庫設定基於 Ubuntu。 |
| **網路連線** | 階段一的機器需要**完整的網際網路連線**。 | 用於下載所有離線物料。 |
| **用戶權限** | 所有 Playbook 步驟均需要 **`become: yes` (root 權限)** 執行。 | Ansible 連線用戶需配置 **`sudo` 權限**。 |
| **K8s 版本** | **$v1.34.1$** (由 Playbook 變數 `kube_version` 定義)。 | |
| **Calico 版本** | **$v3.30.3$** (由 Playbook 變數 `calico_version` 定義)。 | |

### Ansible Inventory 配置範例

假設您有一個名為 `hosts.ini` 的 Inventory 文件，至少需要定義 Master 節點。

```ini
[master]
192.168.1.10 ansible_user=your_ssh_user master_ip=192.168.1.10

[worker]
192.168.1.11 ansible_user=your_ssh_user
192.168.1.12 ansible_user=your_ssh_user

[K8S-Nodes:children]
master
worker
```

## 步驟分解

### 階段一：K8s 離線物料準備與打包 (`階段一：K8s 離線物料準備與打包`)

此階段在 **Ansible 控制機 (localhost)** 執行，需要網路連線。

1.  **創建必要目錄**：
      * `/home/administrator/kubernetes/offline` (`offline_dir`)
      * `/home/administrator/kubernetes/offline/offline_packages` (`pkg_dir`)
2.  **設置 Docker 與 K8s APT 倉庫**：
      * 安裝前置依賴 (`ca-certificates`, `curl`, `gnupg`)。
      * 下載 **Docker GPG Key** 並添加 Docker APT 倉庫。
      * 下載 **K8s GPG Key** 並添加 K8s APT 倉庫 (基於 `v1.34`)。
3.  **下載所有 .deb 套件**：
      * 使用 `apt-get download` 和 `apt-get install --download-only` 下載 **`containerd.io`**, **`docker-ce`**, **`kubelet`**, **`kubeadm`**, **`kubectl`** 等核心套件及其所有運行時依賴，確保離線安裝的完整性。
4.  **準備鏡像清單與 CNI Manifest**：
      * 創建 `required_images.txt` 包含所有 K8s 核心組件和 Calico 的容器鏡像列表。
      * 下載 Calico 的 **`tigera-operator.yaml`** 和 **`custom-resources.yaml`** 配置檔。
5.  **處理容器鏡像** (通過 `image_handling.yml` - 程式碼中為獨立的 Task Block)：
      * 遍歷 `required_images.txt`。
      * **K8s 核心鏡像** (如 `registry.k8s.io/...`)：使用 **`registry.aliyuncs.com/google_containers`** 加速器拉取，然後重新 **`docker tag`** 為官方名稱，並清理加速器鏡像。
      * **Calico 鏡像** (如 `docker.io/...`)：直接拉取。
      * 最後使用 **`docker save`** 將所有已拉取的鏡像打包成 **`k8s_calico_images_v1.34.1.tar.gz`**。
6.  **製作最終離線資源包**：
      * 將 `offline_dir` 下的所有文件打包成 **`k8s_offline_bundle_v1.34.1.tar.gz`**。

-----

### 階段二：K8s 集群離線安裝與部署 (`階段二：K8s 集群離線安裝與部署 (使用 Calico Operator)`)

此階段在 **目標 K8s 節點 (`K8S-Nodes` 組)** 執行，無需網路連線。

1.  **解壓縮離線資源包**：
      * 將 `k8s_offline_bundle_v1.34.1.tar.gz` 解壓縮到目標節點的 `{{ offline_dir }}`。
2.  **系統前置配置**：
      * **關閉 Swap**：`swapoff -a` 並註釋 `/etc/fstab` 中的 Swap 條目。
      * **網路模組與 Sysctl**：載入 **`overlay`** 和 **`br_netfilter`** 模組，並配置 `sysctl` 參數以啟用 **IPv4 轉發**和 **Bridge Filter**，這是 K8s 和 CNI 網絡的必需配置。
3.  **離線安裝所有 .deb 套件**：
      * 使用 **`dpkg -i {{ pkg_dir }}/*.deb`** 離線安裝所有預先下載的套件。
4.  **配置 Containerd Cgroup Driver**：
      * 生成默認配置檔 `/etc/containerd/config.toml`。
      * 將 `SystemdCgroup` 設置為 **`true`**，以確保 **Containerd** 和 **Kubelet** 使用相同的 **`systemd` Cgroup 驅動**。
      * 重啟並啟用 **`containerd`** 服務。
5.  **啟用 Kubelet 服務**：
      * 啟動並設置 **`kubelet`** 開機自啟。
6.  **導入所有 K8s 核心與 Calico 鏡像**：
      * 使用 **`docker load -i ...`** 載入預先打包的鏡像檔案，使鏡像在離線環境中可用。
7.  **集群初始化與網絡部署** (註釋部分，需手動執行或取消註釋)：

| 步驟編號 | 節點類型 | 動作 | 說明 |
| :--- | :--- | :--- | :--- |
| **8.** | Master | `kubeadm init` | 初始化集群，指定 `$master\_ip$` 和 `$pod\_network\_cidr$`。 |
| **9.** | Master | 配置 Kubeconfig | 將 `/etc/kubernetes/admin.conf` 複製到 `$HOME/.kube/config`。 |
| **10.** | Master | 部署 Tigera Operator | `kubectl apply -f tigera-operator.yaml` (這是 CNI 部署的第一步)。 |
| **11.** | Master | 部署 Calico CRs | `kubectl apply -f custom-resources.yaml` (定義 Calico 網絡配置)。 |
| **12.** | Worker | 執行 Join 命令 | 執行 Master 節點初始化後生成的 **`kubeadm join`** 命令。 |

## 程式碼範例：Ansible 核心變數

以下是 Playbook 中使用的核心變數：

```yaml
vars:
  k8s_version: "v1.34.1" # Kubelet/Kubeadm 的版本標籤
  kube_version: "1.34.1" # APT 倉庫使用的版本號
  offline_dir: "/home/administrator/kubernetes/offline" # 資源包存放主目錄
  pkg_dir: "{{ offline_dir }}/offline_packages" # .deb 套件目錄
  k8s_accelerator: "registry.aliyuncs.com/google_containers" # K8s 核心鏡像加速器
  calico_version: "v3.30.3" # Calico 版本
  pod_network_cidr: "192.168.0.0/16" # Pod 網絡 CIDR
  master_ip: "192.168.1.10" # Master 節點 IP (用於 kubeadm init)
```

## 錯誤處理與建議

### 1\. APT 依賴缺失 ($dpkg$ 錯誤)

**問題描述**：在階段二的第 4 步 **`dpkg -i`** 執行時，可能會因為某些次級依賴缺失而報錯。
**解決方案**：

  * **檢查**：確保階段一的第 4 步 (`強制下載所有套件的運行時依賴`) 已成功執行。`apt-get install -y --reinstall --download-only` 比單純的 `apt-get download` 更能確保依賴完整性。
  * **修復**：如果遇到依賴問題，可以在目標節點上執行 `apt --fix-broken install`，雖然這會嘗試聯網下載，但在完全離線環境中，唯一的辦法是手動檢查缺失的 `.deb` 包並將其添加到離線資源包中。

### 2\. 鏡像拉取失敗 (階段一)

**問題描述**：在階段一處理鏡像時，如果網路不穩定或加速器地址不可用，**`docker pull`** 可能失敗。
**解決方案**：

  * **檢查**：確保 **`k8s_accelerator`** (`registry.aliyuncs.com/google_containers`) 在執行機器上是可訪問且有效的。
  * **手動替換**：如果加速器無法使用，必須手動尋找一個可用的鏡像倉庫，並修改 `image_handling.yml` 中的邏輯，或者將 `required_images.txt` 中的鏡像替換為可用的離義鏡像地址。

### 3\. Cgroup 驅動不匹配

**問題描述**：`kubelet` 啟動後，節點一直處於 **`NotReady`** 狀態，日誌顯示 **`cgroup driver`** 不匹配。
**解決方案**：

  * 階段二的第 5 步已處理：確保 **`containerd`** 的 `SystemdCgroup` 設置為 **`true`**。這是 $v1.22+$ 之後 K8s 的標準做法，與 **`kubelet`** 保持一致。執行後必須 **`systemctl restart containerd`**。

## 總結

這份 Ansible Playbook 為在嚴格的離線環境下部署 K8s $v1.34.1$ 提供了一個穩健且自動化的解決方案。它細緻地處理了 **APT 套件下載**、**鏡像加速器使用**、**系統前置配置**和 **CNI 網路配置**等所有複雜環節，大大簡化了離線部署的難度。

## 參考資料

  * Kubernetes 官方文檔：[https://kubernetes.io/docs/setup/](https://kubernetes.io/docs/setup/)
  * Calico 官方文檔：[https://docs.tigera.io/calico/latest/](https://docs.tigera.io/calico/latest/)
  * Ansible 官方文檔：[https://docs.ansible.com/](https://docs.ansible.com/)