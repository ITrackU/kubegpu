## 📖 Wiki & Documentation
Full setup guide: **[Wiki Home](https://github.com/ITrackU/kubegpu)**

### **Key Sections**
- **[NVIDIA Driver Guide](nvidia-driver-rhel9)**: Step-by-step `.run` installation.
- **[Test Workloads](gpu-test-workloads)**: Validate GPU access with CUDA pods.
- **[Master Node Setup](master-node-setup)**: Deploy k3s server.
- **[GPU Node Setup](gpu-node-setup)**: Install NVIDIA drivers and join the cluster.


### **Prerequisites**
- RHEL 9 / Rocky Linux 9.
- NVIDIA GPU (Pascal or newer).
- **Secure Boot disabled** (or MOK enrolled).
