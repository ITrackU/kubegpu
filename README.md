## 📖 Wiki & Documentation
Full setup guide: **[Wiki Home](https://github.com/ITrackU/kubegpu)**

### **Key Sections**
- **[NVIDIA Driver Guide](nvidia-driver-rhel9)**: Step-by-step `.run` installation.
- **[Test Workloads](#5-gpu-test-workloads)**: Validate GPU access with CUDA pods.
- **[GPU Node Setup](#3-gpu-node-setup-per-agent)**: Install NVIDIA drivers and join the cluster.
- **[Master Node Setup](#2-master-node-setup)**: Deploy k3s server.

### **Prerequisites**
- RHEL 9 / Rocky Linux 9.
- NVIDIA GPU (Pascal or newer).
- **Secure Boot disabled** (or MOK enrolled).
