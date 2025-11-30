# 🚀 DevOpsLabX

> **A hands-on learning repository for mastering Linux System Administration and Kubernetes fundamentals**

Build a rock-solid foundation for your DevOps career by mastering the essentials: Linux administration and Kubernetes Pod management. This repository provides structured, lab-focused tutorials designed for self-learners, RHCSA candidates, and aspiring DevOps engineers.

---

## 📚 What's Inside

### 🐧 **Linux** (57 Tutorials)
Master Linux system administration from the ground up — the essential foundation for every DevOps engineer.

**Topics Covered:**
- **Fundamentals**: Lab setup with Proxmox & Cloud-Init, command line mastery, shell fundamentals
- **File Management**: File operations, globbing, permissions & security (ACLs, special bits)
- **User Administration**: User & group management, authentication
- **Process Control**: Process management, system monitoring, performance tuning
- **Storage Management**: Disk management, filesystem configuration (ext2/3/4, XFS)
- **Advanced Storage**: LVM (snapshots, thin provisioning, migration, merge/split, restoration)
- **RAID Systems**: RAID-0, RAID-1, RAID-5, RAID-6, RAID-10 configuration & management
- **Advanced Disk Features**: VDO compression, disk quotas, LUKS encryption
- **System Administration**: Kernel management, boot recovery, kernel panic resolution
- **Logging & Performance**: System logging, performance monitoring & tuning
- **Package Management**: RPM, DNF, APT, source compilation
- **Networking**: nmcli configuration, NIC teaming, firewalld
- **Virtualization**: KVM setup & management, Cockpit web dashboard
- **Security**: SELinux (contexts, MLS/MCS, troubleshooting, file contexts)
- **Services**: DNS (BIND), Apache with multi-site SSL, MySQL, Squid proxy, load balancing, Nginx, NFS
- **DevOps Tools**: Git fundamentals, version control

### ☸️ **Kubernetes** (68 Tutorials)
Comprehensive Kubernetes learning from Pod basics to production-ready deployments and advanced networking.

**Topics Covered:**
- **Pod Fundamentals**: Pod lifecycle, YAML structure, container ports & networking
- **Pod Configuration**: Init containers, secrets management, environment variables
- **Resource Management**: CPU/Memory requests & limits, Quality of Service (QoS) classes
- **QoS Deep Dive**: BestEffort, Burstable, and Guaranteed classes
- **Troubleshooting**: Pending pods, OOMKilled errors, scheduling issues, debugging techniques
- **Scheduling & Placement**: Node selection, nodeSelector, nodeName, node affinity (required/preferred)
- **Advanced Scheduling**: Taints & tolerations, pod anti-affinity, NotIn operator
- **Priority & Preemption**: Pod priority classes, preemption policies
- **Deployments**: Replica management, rolling updates, production best practices
- **Update Strategies**: RollingUpdate vs Recreate, maxSurge, maxUnavailable
- **Autoscaling**: Horizontal Pod Autoscaler (HPA v1 & v2), autoscaling strategies
- **Health & Lifecycle**: Liveness probes, readiness probes, lifecycle hooks
- **Services**: ClusterIP, NodePort, service discovery, load balancing
- **Resource Governance**: ResourceQuotas, LimitRange (CPU/Memory constraints)
- **Storage**: emptyDir, hostPath, PersistentVolumeClaims (PVC), dynamic provisioning
- **Configuration Management**: ConfigMaps, Secrets, volume mounts, stringData best practices
- **Security**: ConfigMap/Secret immutability, access control
- **Network Policies**: Ingress/egress rules, default deny policies, pod isolation
- **Advanced Networking**: NetworkPolicy troubleshooting, complex rule combinations
- **Ingress Controllers**: Traefik routing (K3s), TLS/SSL configuration, path-based routing
- **Ingress Advanced**: Annotations, Gateway API preview, future of K8s networking

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
Linux Basics → Shell Mastery → System Admin → Storage → Kubernetes Pods → Resource Management → Deployments → Networking
```

**Recommended Order:**
1. **Linux/01-Basics**: Lab setup, command line & shell fundamentals
2. **Linux/02-System-Administration**: Users, security, processes, basic storage
3. **Linux/03-Advanced-Storage**: Filesystems, LVM, RAID
4. **Kubernetes/01-Pod-Fundamentals**: Pod basics, init containers, secrets
5. **Kubernetes/02-Resource-Management**: Resource management, QoS, troubleshooting
6. **Kubernetes/04-Deployments-Scaling**: Deployments, scaling, autoscaling
7. **Kubernetes/05-Health-Services**: Health probes, services

### 🔄 **Linux Admin → Kubernetes Path**

Already comfortable with Linux? Jump to Kubernetes with this focused path:

1. **Quick Review**: Linux/04-System-Management (logging, performance, packages)
2. **K8s Fundamentals**: Kubernetes/01-Pod-Fundamentals (Pods, ports, init containers)
3. **Resource Management**: Kubernetes/02-Resource-Management (requests/limits, QoS classes)
4. **Scheduling**: Kubernetes/03-Scheduling (node selection, affinity, taints)
5. **Production Ready**: Kubernetes/04-Deployments-Scaling (Deployments, HPA, rolling updates)
6. **Networking**: Kubernetes/09-Network-Policies (NetworkPolicy, Ingress)

### 🚀 **Advanced DevOps Path**

For experienced practitioners looking to master advanced topics:

**Linux Advanced:**
1. **Storage Deep Dive**: Linux/03-Advanced-Storage (LVM, RAID, VDO, quotas, LUKS)
2. **System Hardening**: Linux/06-Virtualization-Security (SELinux mastery)
3. **Networking**: Linux/05-Networking (nmcli, NIC teaming, firewalld)
4. **Services**: Linux/07-Services (DNS, web servers, load balancing, monitoring)

**Kubernetes Advanced:**
1. **Advanced Scheduling**: Kubernetes/03-Scheduling (Anti-affinity, priority, preemption)
2. **Production Deployments**: Kubernetes/04-Deployments-Scaling (Best practices, HPA v2, troubleshooting)
3. **Resource Governance**: Kubernetes/06-Resource-Governance (ResourceQuotas, LimitRange)
4. **Storage**: Kubernetes/07-Storage (PVC, dynamic provisioning, scaling)
5. **Configuration**: Kubernetes/08-Configuration (ConfigMaps, Secrets, best practices)
6. **Network Security**: Kubernetes/09-Network-Policies (NetworkPolicy, isolation, troubleshooting)

---

## 📂 Repository Structure

```
DevOpsLabX/
│
├── Linux/                          # 57 Linux tutorials
│   ├── Basics (01-09)
│   │   ├── 01 - Build_a_Proxmox_VM_Factory.md
│   │   ├── 02 - RHCSA__DevOps_Foundation.md
│   │   ├── 03 - Mastering_the_Linux_CLI.md
│   │   ├── 04 - Command_Line_Demystified.md
│   │   ├── 05 - Working_with_Files_in_Linux.md
│   │   ├── 06 - Linux_Shell_Fundamentals.md
│   │   ├── 07 - Linux_Control_Operators.md
│   │   ├── 08 - Shell__Variables___History.md
│   │   └── 09 - Mastering_File_Globbing.md
│   │
│   ├── System Administration (010-014)
│   │   ├── 010 - Linux_User___Group_Guide.md
│   │   ├── 011 - File_Security_in_Linux.md
│   │   ├── 012 - Mastering_Linux_Processes.md
│   │   ├── 013 - Mastering_Linux_Disks.md
│   │   └── 014 - Linux_Filesystem_Management.md
│   │
│   ├── Advanced Storage (015-030)
│   │   ├── 015 - Linux_FS_Migration__EXT2-EXT4.md
│   │   ├── 016 - Mastering_Linux_Storage.md
│   │   ├── 017-023 - LVM (Resizing, Snapshots, Restore, Merge/Split, Migration, Thin Provisioning)
│   │   ├── 024-028 - RAID (RAID-0, RAID-1, RAID-5, RAID-6, RAID-10)
│   │   ├── 029 - Getting_Started_with_VDO.md
│   │   ├── 030 - Disk_Quota_Management.md
│   │   └── 031 - Disk_Encryption_with_LUKS.md
│   │
│   ├── System Management (032-037)
│   │   ├── 032 - Resolving_Kernel_Panic.md
│   │   ├── 033 - CentOS_8__boot_Recovery.md
│   │   ├── 034 - RHEL_8_Kernel_Installation.md
│   │   ├── 035 - Mastering_Linux_Logs.md
│   │   ├── 036 - Linux_Perf.md
│   │   └── 037 - Linux_Package_Management.md
│   │
│   ├── Networking (038-041)
│   │   ├── 038 - RHEL_9_Networking__nmcli.md
│   │   ├── 039 - RHEL_9_nmcli_Guide.md
│   │   ├── 040 - RHEL_9_NIC_Teaming.md
│   │   └── 041 - Firewalld_Fundamentals.md
│   │
│   ├── Virtualization & Security (042-048)
│   │   ├── 042 - Cockpit__Server_Web_Dashboard.md
│   │   ├── 043 - KVM__Zero_to_Host.md
│   │   └── 044-048 - SELinux (Fundamentals, File Contexts, MLS/MCS)
│   │
│   └── Services (049-058)
│       ├── 049 - DNS_Config_in_Linux_w__BIND.md
│       ├── 050 - Apache__Multi-Site_SSL.md
│       ├── 051 - MySQL_Server_on_RHEL_8.md
│       ├── 052 - Squid_Proxy_Configuration.md
│       ├── 053 - Building_a_Web_Load_Balancer.md
│       ├── 054 - Unlocking_Nginx.md
│       ├── 056 - Setting_Up_NFS.md
│       ├── 057 - Monitoring_Linux_Systems.md
│       └── 058 - Git__The_Dev_s_Time_Machine.md
│
└── Kubernetes/                     # 68 Kubernetes tutorials
    ├── Pod Fundamentals (1.x-2.x)
    │   ├── 1.1 Demystifying_Kubernetes_Pods.md
    │   ├── 1.2 K8s_Ports__Myth_vs.md
    │   ├── 2.1 Kubernetes_Init_Containers.md
    │   └── 2.2 Kubernetes_Pod_Passwords.md
    │
    ├── Resource Management (3.x)
    │   ├── 3.1 K8s__Taming_App_Resources.md
    │   ├── 3.1 Requests vs. Limits.md
    │   ├── 3.2 K8s_QoS__BestEffort_s_Risk.md
    │   ├── 3.3 Kubernetes__Burstable_Pods.md
    │   └── 3.4 Guaranteed_QoS_Class.md
    │
    ├── Troubleshooting (4.x)
    │   ├── 4.1 The_Pending_Pod.md
    │   └── 4.2 Kubernetes_OOMKilled_Guide.md
    │
    ├── Scheduling & Placement (5.x)
    │   ├── 5.1 The_`nodeName`_Trap.md
    │   ├── 5.2 K8s_Node_Selection.md
    │   ├── 5.3-5.4 - Node Affinity (Required/Preferred)
    │   ├── 5.5 Kubernetes__The_NotIn_Operator.md
    │   ├── 5.6 K8s_Pod_Anti-Affinity.md
    │   ├── 5.7 Kubernetes_Affinity__Power_of_OR.md
    │   ├── 5.8 Taints___Tolerations.md
    │   └── 5.9 Pod_Priority_&_Preemption.md
    │
    ├── Deployments & Scaling (6.x)
    │   ├── 6.1 Kubernetes_Deployments.md
    │   ├── 6.2 Kubernetes__Orders_vs_Goals.md
    │   ├── 6.3 Production-Ready_K8s.md
    │   ├── 6.4 Kubernetes_Rolling_Updates.md
    │   ├── 6.5 K8s__Pod_Anti-Affinity.md
    │   ├── 6.6 K8s__Deployments_vs_Services.md
    │   ├── 6.7 Default_to_Defended.md
    │   ├── 6.8 Mastering_Kubernetes_HPA.md
    │   ├── 6.9 Kubernetes_HPA__V1_to_V2.md
    │   ├── 6.10 Taming_Kubernetes_Autoscaling.md
    │   └── 6.11 Kubernetes_Troubleshooting.md
    │
    ├── Health & Services (7.x)
    │   ├── 7.1 Kubernetes_Liveness_Probes.md
    │   ├── 7.2 K8s_Health_Probes.md
    │   ├── 7.3 Mastering_Lifecycle_Hooks.md
    │   ├── 7.4 Mastering_ClusterIP_Service.md
    │   └── 7.5 Unlocking_k3s_with_NodePort.md
    │
    ├── Resource Governance (8.x)
    │   ├── 8.1 Taming_K8s_ResourceQuotas.md
    │   ├── 8.2 Kubernetes_Resource_Quotas.md
    │   ├── 8.3 K8s_LimitRange_Rulebook.md
    │   ├── 8.4 K8s__The_CPU_LimitRange.md
    │   └── 8.5 K8s_Memory_LimitRange.md
    │
    ├── Storage (9.x)
    │   ├── 9.1 Kubernetes__The_emptyDir_Volume.md
    │   ├── 9.2 Kubernetes__The_Shared_Mailbox.md
    │   ├── 9.3 hostPath__A_Double-Edged_Sword.md
    │   ├── 9.4 Kubernetes_hostPath_Deception.md
    │   ├── 9.5 PVC__Storage_Matchmaker.md
    │   ├── 9.6 Dynamic_Kubernetes_Storage.md
    │   ├── 9.7 The_K8s_Scaling_Trap.md
    │   └── 9.8 The_Ultimate_NodePort_Guide.md
    │
    ├── Configuration (11.x)
    │   ├── 11.1 Kubernetes_ConfigMaps.md
    │   ├── 11.2 Kubernetes_Secrets.md
    │   ├── 11.3 K8s_Config__Volume_Mounts.md
    │   ├── 11.4 Always_Use_`stringData`.md
    │   ├── 11.5 Lock_Down_K8s_Configs.md
    │   └── 11.6 K8s_Config___Secrets.md
    │
    ├── Network Policies (12.x)
    │   ├── 12.1 Kubernetes_NetworkPolicy.md
    │   ├── 12.2 Kubernetes_NetworkPolicy.md
    │   ├── 12.3 Kubernetes_Egress_Rules.md
    │   ├── 12.4 Kubernetes_Default_Deny.md
    │   ├── 12.5 Advanced_NetworkPolicy.md
    │   └── 12.6 NetworkPolicy_Troubleshooting.md
    │
    └── Ingress (13.x)
        ├── 13.1 Ingress__Cluster_Front_Door.md
        ├── 13.2 K3s_Ingress_Routing_with_Traefik.md
        ├── 13.3 Securing_Kubernetes_Ingress.md
        ├── 13.4 Ingress_Path-Based_Routing.md
        ├── 13.5 Ingress__Power_of_Annotations.md
        └── 13.6 Future_of_K8s_Networking.md
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
