## 📖 Wiki & Documentation
Full setup guide: **[Wiki Home](https://github.com/ITrackU/kubegpu)**

### **Key Sections**
- **[NVIDIA Driver Guide](nvidia-driver-rhel9)**: Step-by-step `.run` installation.
- **[Test Workloads](gpu-test-workloads)**: Validate GPU access with CUDA pods.
- **[GPU Node Setup](gpu-node-setup)**: Install NVIDIA drivers and join the cluster.
- **[Master & Slaves Nodes Setup](master-slave-nodes-setup)**: Deploy k3s server and slaves

### **Prerequisites**
- RHEL 9 / Rocky Linux 9.
- NVIDIA GPU (Pascal or newer).
- **Secure Boot disabled** (or MOK enrolled).
