# **Ollama-OpenWebUI Corporate Wiki**
*A secure, multi-user deployment guide for enterprise environments*

---

## **Overview**
This wiki documents the deployment of **Ollama + OpenWebUI** in a **multi-user corporate Linux environment**, addressing:
✅ **Security** – Interim safeguards while IT-Security formalizes policies.
✅ **Multi-access** – Simultaneous browser access for all users.
✅ **Compliance** – Aligned with corporate Linux standards and GPU acceleration.

The deployment uses **Podman containers**, **NVIDIA GPU acceleration**, and **HTTPS/TLS encryption** for a production-ready setup.

---

## **📌 Quick Start**
For a **fully operationalized** stack with:
- **Isolated containers** (Ollama + OpenWebUI)
- **Dual-GPU support** (optimized for LLMs)
- **Encrypted traffic** (HTTPS/TLS)
- **No dependency conflicts** (native Linux support)

→ **Deploy in hours** using the guides below.

---

## **📂 Wiki Structure**

### **1. Prerequisites**
- [NVIDIA Driver Installation Guide](NVIDIA-DRIVER-INSTALLATION-GUIDE)
- [NVIDIA Container Toolkit Setup](NVIDIA-CONTAINER-TOOLKIT-INSTALLATION)
- [Podman + CDI Installation](PODMAN-CDI-INSTALLATION)

### **2. Deployment**
- [Application Build & Configuration](APPLICATION-BUILD)
- [Container Management (watchtower, ollama, openwebui, nginx)](CONTAINER-MANAGEMENT) *(link to be added)*
- [Security & HTTPS Setup](SECURITY-HTTPS) *(link to be added)*

### **3. Operations**
- [User Access & Permissions](USER-ACCESS) *(link to be added)*
- [Monitoring & Logs](MONITORING-LOGS) *(link to be added)*
- [Troubleshooting](TROUBLESHOOTING) *(link to be added)*

---

## **🖥️ Architecture**
The deployment runs under the **`ai` user** with the following containers:

| **Container**  | **Purpose**                          |
|----------------|--------------------------------------|
| `watchtower`   | Auto-updates containers              |
| `ollama`       | LLM model hosting                    |
| `openwebui`    | Web interface for users             |
| `nginx`        | Reverse proxy + HTTPS termination   |

**Key Features:**
- **GPU-accelerated** (NVIDIA CDI + Container Toolkit)
- **Isolated** (Podman rootless mode)
- **Scalable** (Supports multiple concurrent users)

---

## **🚀 Next Steps**
1. **Install prerequisites** (NVIDIA drivers, Podman, etc.).
2. **Build the application** using the [Application Build Guide](APPLICATION-BUILD).
3. **Configure access** for users (see [User Access](USER-ACCESS) when available).

---
### **📢 Need Help?**
- Report issues in **[TROUBLESHOOTING](TROUBLESHOOTING)**.
- Suggest improvements via **[CONTRIBUTING](CONTRIBUTING)** *(link to be added)*.

---

### **🔒 Security Note**
This setup includes **interim security measures** while awaiting formal IT-Security policies. Ensure:
- HTTPS is enforced.
- Container permissions are restricted.
- GPU access is limited to authorized users.
