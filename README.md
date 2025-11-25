# 🚀 DevOpsLabX

> **A hands-on learning repository for mastering Linux System Administration and Kubernetes fundamentals**

Build a rock-solid foundation for your DevOps career by mastering the essentials: Linux administration and Kubernetes Pod management. This repository provides structured, lab-focused tutorials designed for self-learners, RHCSA candidates, and aspiring DevOps engineers.

---

## 📚 What's Inside

### 🐧 **Linux** (14 Tutorials)
Master Linux system administration from the ground up — the essential foundation for every DevOps engineer.

**Topics Covered:**
- Lab setup with Proxmox & Cloud-Init automation
- Command line mastery & shell fundamentals
- File operations, permissions & security (ACLs, special bits)
- User & group management
- Process control & system monitoring
- Disk management & filesystem configuration

### ☸️ **Kubernetes** (13 Tutorials)
Learn Kubernetes Pod lifecycle, resource management, and real-world troubleshooting.

**Topics Covered:**
- Pod fundamentals & YAML structure
- Container ports & networking basics
- Init containers & secrets management
- Resource requests & limits (CPU/Memory)
- Quality of Service (QoS) classes
- Troubleshooting: Pending pods, OOMKilled errors, scheduling issues

---

## 🎯 Who Is This For?

✅ **Aspiring DevOps Engineers** — Build the Linux + Kubernetes foundation  
✅ **System Administrators** — Transition to cloud-native technologies  
✅ **RHCSA/CKA Candidates** — Hands-on practice for certifications  
✅ **Self-Learners** — Structured path with lab exercises  

---

## 🗺️ Learning Paths

### 🌱 **Complete Beginner Path**

```
Linux Basics → Shell Mastery → System Admin → Kubernetes Pods → Resource Management → Troubleshooting
```

**Recommended Order:**
1. **Linux 01-02**: Lab setup + Why Linux matters for DevOps
2. **Linux 03-09**: Command line & shell fundamentals
3. **Linux 010-014**: Users, security, processes, storage
4. **Kubernetes 1.1-1.2**: Pod basics & networking
5. **Kubernetes 3.1-3.4**: Resource management & QoS
6. **Kubernetes 4.1-4.2**: Troubleshooting common issues

### 🔄 **Linux Admin → Kubernetes Path**

Already comfortable with Linux? Jump straight to Kubernetes:

1. **Quick Review**: Linux 011-014 (security, processes, disks)
2. **K8s Fundamentals**: 1.1-1.2 (Pods, ports, namespaces)
3. **Resource Management**: 3.1-3.4 (requests/limits, QoS)
4. **Troubleshooting**: 4.1-4.2 (Pending pods, OOMKilled)

---

## 📂 Repository Structure

```
DevOpsLabX/
│
├── Linux/                          # 14 Linux tutorials
│   ├── 01 - Build_a_Proxmox_VM_Factory.md
│   ├── 02 - RHCSA__DevOps_Foundation.md
│   ├── 03 - Mastering_the_Linux_CLI.md
│   ├── 04 - Command_Line_Demystified.md
│   ├── 05 - Working_with_Files_in_Linux.md
│   ├── 06 - Linux_Shell_Fundamentals.md
│   ├── 07 - Linux_Control_Operators.md
│   ├── 08 - Shell__Variables___History.md
│   ├── 09 - Mastering_File_Globbing.md
│   ├── 010 - Linux_User___Group_Guide.md
│   ├── 011 - File_Security_in_Linux.md
│   ├── 012 - Mastering_Linux_Processes.md
│   ├── 013 - Mastering_Linux_Disks.md
│   └── 014 - Linux_Filesystem_Management.md
│
└── Kubernetes/                     # 13 Kubernetes tutorials
    ├── 1.1 Demystifying_Kubernetes_Pods.md
    ├── 1.2 K8s_Ports__Myth_vs.md
    ├── 2.1 Kubernetes_Init_Containers.md
    ├── 2.2 Kubernetes_Pod_Passwords.md
    ├── 3.1 K8s__Taming_App_Resources.md
    ├── 3.1 Requests vs. Limits.md
    ├── 3.2 K8s_QoS__BestEffort_s_Risk.md
    ├── 3.3 Kubernetes__Burstable_Pods.md
    ├── 3.4 Guaranteed_QoS_Class.md
    ├── 4.1 The_Pending_Pod.md
    ├── 4.2 The_Case_of_the_Pending_Pod.md
    ├── 4.2 Kubernetes_OOMKilled_Guide.md
    └── 5.1 The_`nodeName`_Trap.md
```

---

## 🧪 Learning Philosophy

Every tutorial follows a consistent, hands-on approach:

✅ **Lab Exercises** — Practice every concept immediately  
✅ **Real-World Scenarios** — Troubleshooting guides based on actual issues  
✅ **Best Practices** — Learn the right way from the start  
✅ **Safety Warnings** — Avoid common pitfalls and destructive mistakes  
✅ **Cheat Sheets** — Quick reference tables for commands  

> **"If Linux is the foundation, RHCSA is the blueprint — and DevOps is the skyscraper you'll build on top of it."**

---

## 🚦 Getting Started

### Prerequisites

**For Linux Tutorials:**
- A Linux system (Ubuntu, RHEL, CentOS, or Proxmox for lab setup)
- Terminal access
- Curiosity and willingness to experiment!

**For Kubernetes Tutorials:**
- Basic Linux command line knowledge
- A Kubernetes cluster (Minikube, kind, or cloud provider)
- `kubectl` installed

### Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/YourUsername/DevOpsLabX.git
   cd DevOpsLabX
   ```

2. **Start with Linux basics:**
   ```bash
   # Read the foundation
   cat Linux/02\ -\ RHCSA__DevOps_Foundation.md
   
   # Begin hands-on learning
   cat Linux/03\ -\ Mastering_the_Linux_CLI.md
   ```

3. **Follow the labs** in each file — every tutorial includes step-by-step exercises

---

## ⚠️ Important Safety Warnings

Throughout this repository, you'll find critical warnings to prevent data loss and system issues:

### **Linux**
- ❌ Never run `rm -rf` without testing with `ls` first
- ❌ Always use `su -` (not just `su`) for proper environment
- ❌ Test `/etc/fstab` with `mount -a` before rebooting
- ❌ Never run `fsck` on mounted filesystems

### **Kubernetes**
- ⚠️ Memory overuse = immediate kill (OOMKilled)
- ⚠️ BestEffort pods = first to be evicted under pressure
- ⚠️ `nodeName` bypasses scheduler = dangerous in production
- ⚠️ Always set resource requests/limits in production

---

## 🎓 Certification Alignment

This repository aligns with industry-recognized certifications:

| Certification | Relevant Content |
|--------------|------------------|
| **RHCSA** (Red Hat Certified System Administrator) | Linux files 02-014 |
| **CKA** (Certified Kubernetes Administrator) | All Kubernetes files |
| **CKAD** (Certified Kubernetes Application Developer) | Kubernetes 1.x, 2.x, 3.x |

---

## 🔧 Tools & Technologies

### **Linux Section**
- Proxmox, Cloud-Init, Bash scripting
- File systems: ext2/4, XFS
- Partitioning: fdisk, gdisk, parted
- Security: umask, ACLs, special permission bits
- Process management: ps, top, kill, systemd

### **Kubernetes Section**
- Pods, Namespaces, Labels, Selectors
- Resource management (CPU/Memory)
- QoS classes (BestEffort, Burstable, Guaranteed)
- Init containers, Secrets
- Troubleshooting: kubectl describe, logs, top

---

## 📖 Key Concepts You'll Master

### **Linux**
- **Everything is a file** — Unified system model
- **Shell expansion** — How Bash interprets commands
- **Permissions & ACLs** — Fine-grained access control
- **Process signals** — SIGTERM vs SIGKILL
- **Filesystem hierarchy** — Understanding `/etc/fstab`, `/proc`, `/sys`

### **Kubernetes**
- **Pod lifecycle** — Ephemeral nature, restart policies
- **Resource requests vs limits** — Scheduling vs runtime enforcement
- **QoS classes** — BestEffort, Burstable, Guaranteed
- **Troubleshooting patterns** — Reading events, understanding exit codes
- **Best practices** — Labels, namespaces, resource management

---

## 🌟 What Makes This Repository Different

✨ **Integrated Learning** — Linux + Kubernetes in one place  
✨ **Lab-First Approach** — Every concept has hands-on exercises  
✨ **Troubleshooting Focus** — Dedicated guides for common failures  
✨ **Production-Ready** — Best practices and real-world warnings  
✨ **Progressive Complexity** — From basics to advanced topics  
✨ **Self-Contained** — No need to jump between multiple resources  

---

## 🚧 Roadmap

This repository is actively growing! Upcoming topics:

### **Linux (Coming Soon)**
- [ ] Advanced shell scripting & automation
- [ ] systemd service management
- [ ] Networking fundamentals (routing, firewalls)
- [ ] LVM (Logical Volume Management)
- [ ] SELinux configuration
- [ ] Performance tuning & monitoring

### **Kubernetes (Coming Soon)**
- [ ] Deployments & ReplicaSets
- [ ] Services & Ingress
- [ ] ConfigMaps & Secrets (advanced)
- [ ] StatefulSets & Persistent Volumes
- [ ] RBAC & Security
- [ ] Helm & package management
- [ ] Monitoring with Prometheus & Grafana

---

## 🤝 Contributing

Contributions are welcome! If you'd like to add tutorials, fix errors, or improve content:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-tutorial`)
3. Follow the existing tutorial format:
   - Clear explanations
   - Hands-on lab exercises
   - Safety warnings where applicable
   - Cheat sheets/summary tables
4. Commit your changes (`git commit -m 'Add tutorial on X'`)
5. Push to the branch (`git push origin feature/new-tutorial`)
6. Open a Pull Request

---

## 📝 Tutorial Format Guidelines

Each tutorial should include:

1. **Title & Overview** — What you'll learn
2. **Explanation** — Concept breakdown
3. **Commands/YAML** — Practical examples
4. **Lab Steps** — Hands-on exercises
5. **Tips & Warnings** — Best practices and pitfalls
6. **Summary/Cheat Sheet** — Quick reference

---

## 📞 Support & Community

- **Issues**: Found a bug or have a question? [Open an issue](https://github.com/YourUsername/DevOpsLabX/issues)
- **Discussions**: Share your learning journey or ask questions
- **Feedback**: Suggestions for new topics are always welcome!

---

## 📜 License

This repository is open-source and available under the [MIT License](LICENSE).

---

## 🙏 Acknowledgments

Special thanks to the DevOps and open-source communities for continuous inspiration and knowledge sharing.

---

## ⭐ Star This Repository

If you find this repository helpful, please consider giving it a star! It helps others discover these learning resources.

---

<div align="center">

**Happy Learning! 🚀**

*Built with ❤️ for the DevOps community*

[⬆ Back to Top](#-devopslabx)

</div>
