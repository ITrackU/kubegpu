# 🚀 GPU Setup on RHEL 9 — NVIDIA Drivers, Container Toolkit & k3s Cluster

This wiki covers three major parts:

1. **Installing NVIDIA Drivers (`.run` method)**
2. **Setting up NVIDIA Container Toolkit with Podman + containerd**
3. **Deploying a GPU-accelerated k3s Kubernetes Cluster**

---

# 1️⃣ NVIDIA Driver Installation on RHEL 9 (`.run` Method)

*Disable Nouveau, install proprietary drivers, and configure GPU acceleration.*

---

## ⚠️ Prerequisites

* **Secure Boot**: Disable in BIOS or [enroll a MOK key](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/assembly_working-with-the-kernel-using-grub2_managing-monitoring-and-updating-the-kernel#doc-wrapper).
* **Root Access**: All `sudo` commands must be run as root.
* **Backup**: Snapshot or backup critical data.
* **Network**: Ensure internet access.

---

## 📋 Installation Steps

### 1. Verify GPU & Secure Boot

```bash
lspci | grep -E "VGA|3D|NVIDIA"
mokutil --sb-state
```

---

### 2. Update System & Install Dependencies

```bash
sudo dnf update -y && sudo dnf upgrade -y
sudo dnf install -y \
  kernel-devel-$(uname -r) kernel-headers \
  gcc make dkms \
  libglvnd-glx libglvnd-opengl libglvnd-devel \
  pkgconfig acpid
```

👉 If `dkms` or `epel-release` are missing, see [Troubleshooting](#-troubleshooting-nvidia-driver).

---

### 3. Blacklist Nouveau

```bash
sudo tee /etc/modprobe.d/blacklist-nouveau.conf <<EOF
blacklist nouveau
options nouveau modeset=0
EOF

echo 'omit_dracutmodules+=" nouveau "' | sudo tee /etc/dracut.conf.d/nouveau.conf
sudo dracut -f --omit-drivers nouveau /boot/initramfs-$(uname -r).img $(uname -r)
sudo grubby --update-kernel=ALL --args="rd.driver.blacklist=nouveau modprobe.blacklist=nouveau nouveau.modeset=0"
```

---

### 4. Rebuild GRUB & Set Default Kernel

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grubby --set-default /boot/vmlinuz-$(uname -r)
```

---

### 5. Reboot into Multi-User Mode

```bash
sudo systemctl set-default multi-user.target
sudo reboot
```

👉 After reboot, log in via **text-only console**.

---

### 6. Install NVIDIA Driver

```bash
wget https://us.download.nvidia.com/tesla/535.261.03/NVIDIA-Linux-x86_64-535.261.03.run
chmod +x NVIDIA-Linux-x86_64-*.run
sudo ./NVIDIA-Linux-x86_64-*.run
```

✅ Accept license, allow kernel module compilation, skip 32-bit libs (unless needed), let it update Xorg config.

---

### 7. Re-enable Graphical Boot (optional)

```bash
sudo systemctl set-default graphical.target
sudo reboot
```

---

### 8. Verify Installation

```bash
nvidia-smi
lsmod | grep nvidia
```

---

## 🔧 Troubleshooting (NVIDIA Driver)

* **Nouveau still loading** → remove module & rebuild initramfs.
* **Missing `dkms`** → install EPEL + `dkms`.
* **Driver compile fails** → ensure `kernel-devel` matches.
* **Black screen** → boot rescue, uninstall, retry.
* **Secure Boot blocks driver** → disable or sign module.

👉 Full commands are in original [troubleshooting section](#-troubleshooting-nvidia-driver).

---

## 🧹 Cleanup

```bash
sudo dnf clean all
rm -f ~/NVIDIA-Linux-x86_64-*.run
```

---

## 📌 Next Steps

* [Install NVIDIA Container Toolkit](#2%EF%B8%8F⃣-nvidia-container-toolkit--podman--containerd)
* [Verify CUDA Support](CUDA-VERIFICATION)

---

# 2️⃣ NVIDIA Container Toolkit + Podman + containerd

---

## Prerequisites

* RHEL/CentOS 8 or 9 with NVIDIA drivers installed.
* Root privileges & internet.

---

## Installation Steps

### 1. Add NVIDIA Repository

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
  | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
```

---

### 2. Install Toolkit

```bash
sudo dnf clean all && sudo dnf makecache
sudo dnf install -y nvidia-container-toolkit
```

---

### 3. Configure Runtime

* **Docker:** `sudo nvidia-ctk runtime configure --runtime=docker`
* **containerd:** `sudo nvidia-ctk runtime configure --runtime=containerd`

---

### 4. Generate CDI Specification

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo chmod -R a+rX /etc/cdi
```

---

### 5. Reload Systemd

```bash
sudo systemctl daemon-reload
```

---

### 6. Podman Installation

```bash
sudo dnf install -y podman
sudo systemctl enable --now podman
```

---

### 7. containerd Installation

```bash
sudo wget https://github.com/containerd/containerd/releases/download/v2.0.0/containerd-2.0.0-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-2.0.0-linux-amd64.tar.gz
sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
sudo systemctl enable --now containerd
```

---

### 8. (Optional) DKMS Autoinstall

```bash
sudo dkms autoinstall
```

---

### 9. GPU Test with containerd

```bash
sudo ctr images pull docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04
sudo ctr run --rm --gpus 0 docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04 test-gpu nvidia-smi
```

---

### 10. List CDI Devices

```bash
sudo nvidia-ctk cdi list
```

---

### 11. Cleanup

```bash
sudo dnf clean all
```

---

### 12. Troubleshooting

* DKMS corruption → remove broken versions in `/var/lib/dkms/nvidia/`.
* Driver mismatch → rebuild with `dkms`.

---

# 3️⃣ GPU-Accelerated k3s Cluster

---

## 0. Cleanup (all nodes)

```bash
sudo /usr/local/bin/k3s-uninstall.sh || true
sudo /usr/local/bin/k3s-agent-uninstall.sh || true
sudo rm -rf /var/lib/rancher/k3s /etc/rancher
sudo rm -f /etc/containerd/config.toml
sudo systemctl daemon-reload
```

---

## 1. OS Preparation (all nodes)

* Update, install tools, disable swap.
* Configure kernel sysctl & SELinux.

```bash
sudo dnf update -y
sudo dnf install -y curl vim git conntrack-tools socat iptables

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap /d' /etc/fstab

# Kernel settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# SELinux config
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```

---

## 2. Master Node

* Install k3s with taints.
* Extract `MASTER_IP` + `NODE_TOKEN`.

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--disable=servicelb --disable=traefik --node-taint CriticalAddonsOnly=true:NoExecute" \
  sh -

# Verify
sudo k3s kubectl get nodes
MASTER_IP=$(hostname -I | awk '{print $1}')
NODE_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)
echo "Master IP: $MASTER_IP | Node token: $NODE_TOKEN"
```

---

## 3. GPU Nodes

* Install NVIDIA stack.
* Configure containerd with NVIDIA runtime.
* Join cluster with `K3S_URL` and `K3S_TOKEN`.


### A. NVIDIA Stack Installation
```bash
sudo dnf install -y kernel-devel gcc make

distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
```

### B. Containerd Configuration
```bash
sudo mkdir -p /etc/containerd
containerd config default | sed \
  -e 's|/usr/bin/containerd|/usr/bin/containerd|' \
  -e 's|"runtime_type": "io.containerd.runtime.v1.linux"|"runtime_type": "io.containerd.runc.v2"|' | \
  sudo tee /etc/containerd/config.toml

sudo tee -a /etc/containerd/config.toml <<'EOF'

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
EOF

sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

### C. Join Cluster
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://${MASTER_IP}:6443 \
  K3S_TOKEN=${NODE_TOKEN} \
  INSTALL_K3S_EXEC="--docker" \
  sh -
```

---

## 4. Post-Install

* Label GPU nodes.
* Deploy NVIDIA Device Plugin.

### A. Node Labeling/Tainting (on master)
```bash
# For each GPU node
sudo k3s kubectl label node <node-name> node.kubernetes.io/gpu=true
sudo k3s kubectl taint node <node-name> gpu=true:NoSchedule
```

### B. NVIDIA Device Plugin
```bash
sudo k3s kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml

# Verify resources
sudo k3s kubectl get nodes -o json | jq '.items[].status.allocatable'
```

---

## 5. GPU Test Workloads

* Simple Pod (`nvidia-smi`).
* Job with CUDA runtime.

### Option 1: Quick Pod Test
```yaml
# test-gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-gpu-pod
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-test
    image: nvidia/cuda:11.0-base
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### Option 2: Job-based Test
```yaml
# gpu-test-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cuda-smi
spec:
  template:
    spec:
      containers:
      - name: nvidia-smi
        image: nvidia/cuda:12.4.0-runtime-ubuntu22.04
        command: 
          - nvidia-smi
          - --query-gpu=name,driver_version,memory.total
          - --format=csv
        resources:
          limits:
            nvidia.com/gpu: 1
      restartPolicy: Never
      tolerations:
      - key: "gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
  backoffLimit: 2
```

Apply and verify:
```bash
sudo k3s kubectl apply -f test-gpu-pod.yaml
sudo k3s kubectl logs test-gpu-pod

# OR
sudo k3s kubectl apply -f gpu-test-job.yaml
sudo k3s kubectl logs job/cuda-smi
```

---

## Troubleshooting

* **Nodes not joining** → check `k3s-agent` logs.
* **No GPU resources** → check `nvidia-smi` + device plugin logs.
* **Driver mismatch** → rebuild DKMS modules.
* **Runtime errors** → restart containerd.

| Symptom | Checkpoints |
|---------|-------------|
| Nodes not joining | `systemctl status k3s-agent` <br> `journalctl -u k3s-agent -n 100` |
| No GPU resources | `nvidia-smi` (on node) <br> `kubectl describe node | grep -A 10 Allocatable` <br> `kubectl logs -n kube-system -l app=nvidia-device-plugin-daemonset` |
| GPU permissions | `ls -l /dev/nvidia*` <br> `dmesg | grep -i nvidia` |
| Container runtime issues | `sudo journalctl -u containerd -n 200` |
| Network problems | `sudo k3s kubectl get nodes -o wide` <br> `ping <master-ip>` <br> `nc -vz <master-ip> 6443` |

### Troubleshooting Guide for NVIDIA Driver Mismatch

| Symptom | Checkpoints |
|---------|-------------|
| **Driver/library version mismatch** | <ul><li>`nvidia-smi`: `Failed to initialize NVML: Driver/library version mismatch`</li><li>Containers fail to start with GPU errors</li><li>`dmesg | grep -i nvidia` shows version conflicts</li></ul>**Solution:**<br>```bash<br>sudo dkms install nvidia/$(ls /usr/src \| grep nvidia \| head -n1 \| cut -d'-' -f2-) -k $(uname -r)<br>sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia<br>sudo modprobe nvidia<br>sudo systemctl restart containerd<br>```<br>If persists: `sudo apt install --reinstall nvidia-driver-$(dpkg -l \| grep 'nvidia-driver' \| awk '{print $3}')` |

### Preventive Maintenance Commands

To avoid version mismatches after kernel/driver updates:

1. **Before kernel updates:**
```bash
# Preserve current NVIDIA driver modules
sudo dkms install --force -m nvidia -v $(modinfo -F version nvidia) -k $(uname -r)

# Create rebuild script (run after reboot)
sudo tee /usr/local/sbin/refresh-nvidia <<'EOF'
#!/bin/bash
current_kernel=$(uname -r)
sudo dkms install -m nvidia -v $(ls /usr/src | grep nvidia | head -n1 | cut -d'-' -f2-) -k $current_kernel
sudo update-initramfs -u
sudo systemctl restart containerd
EOF
sudo chmod +x /usr/local/sbin/refresh-nvidia
```

2. **After kernel updates:**
```bash
# Verify module consistency
sudo dkms status | grep nvidia

# Rebuild modules for current kernel
sudo dkms autoinstall -k $(uname -r)
```

3. **After NVIDIA driver updates:**
```bash
# Verify loaded vs. installed versions
lsmod | grep nvidia
ls -d /usr/src/nvidia-*

# Force DKMS rebuild if needed
sudo dkms remove -m nvidia -v $(ls /usr/src | grep nvidia | head -n1 | cut -d'-' -f2-) --all
sudo dkms install -m nvidia -v $(ls /usr/src | grep nvidia | head -n1 | cut -d'-' -f2-)

# Reconfigure Containerd
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

### Automated Maintenance Script
```bash
#!/bin/bash
# nvidia-module-sync.sh
KERNEL=$(uname -r)
DRIVER=$(ls /usr/src | grep nvidia | head -n1 | cut -d'-' -f2-)

# Rebuild for current kernel
sudo dkms install -m nvidia -v $DRIVER -k $KERNEL

# Update library links
sudo rm -f /usr/lib/x86_64-linux-gnu/nvidia/current
sudo ln -s /usr/lib/x86_64-linux-gnu/nvidia/$DRIVER /usr/lib/x86_64-linux-gnu/nvidia/current

# Verify consistency
lsmod | grep nvidia | grep $DRIVER || {
    echo "Reloading modules"
    sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia
    sudo modprobe nvidia
    sudo systemctl restart containerd
}
```

---

# 📚 References

* [NVIDIA Driver Docs](https://download.nvidia.com/XFree86/Linux-x86_64/535.261.03/README/index.html)
* [RHEL 9 Kernel Management](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/index)


