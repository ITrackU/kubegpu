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

---

## 2. Master Node

* Install k3s with taints.
* Extract `MASTER_IP` + `NODE_TOKEN`.

---

## 3. GPU Nodes

* Install NVIDIA stack.
* Configure containerd with NVIDIA runtime.
* Join cluster with `K3S_URL` and `K3S_TOKEN`.

---

## 4. Post-Install

* Label GPU nodes.
* Deploy NVIDIA Device Plugin.

---

## 5. GPU Test Workloads

* Simple Pod (`nvidia-smi`).
* Job with CUDA runtime.

---

## Troubleshooting

* **Nodes not joining** → check `k3s-agent` logs.
* **No GPU resources** → check `nvidia-smi` + device plugin logs.
* **Driver mismatch** → rebuild DKMS modules.
* **Runtime errors** → restart containerd.

👉 Includes preventive maintenance scripts to sync NVIDIA modules.

---

# 📚 References

* [NVIDIA Driver Docs](https://download.nvidia.com/XFree86/Linux-x86_64/535.261.03/README/index.html)
* [RHEL 9 Kernel Management](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/index)
