# ✍️ Virtualization — Short Notes

### 🖥️ Virtualization

> **Running multiple virtual computers (VMs) on one physical computer.**

```text
Physical Server
      ↓
  Hypervisor
  ↙   ↓   ↘
 VM1  VM2  VM3
```

### 💻 Virtual Machine (VM)

> A **virtual computer** that has its own OS, CPU, RAM & storage.

---

### ⚙️ Hypervisor

> Software that **creates + manages VMs** and shares physical resources.

### 🔹 Types of Hypervisor

**1️⃣ Type 1 — Bare Metal**

```text
VMs
 ↓
Hypervisor
 ↓
Hardware
```

* Directly on hardware
* Used in **servers & cloud**
* Example: VMware ESXi

**2️⃣ Type 2 — Hosted**

```text
VM
 ↓
Hypervisor
 ↓
Host OS
 ↓
Hardware
```

* Runs on existing OS
* Used for **learning & testing**
* Example: VirtualBox

---

### ⭐ Why Virtualization?

* 💰 **Cost saving**
* ⚡ **Better resource usage**
* 🔒 **Isolation**
* 🧪 **Easy testing**
* 📈 **Scalable**

### 🧠 Remember

> **Type 1 → Hardware**
> **Type 2 → Host OS**
> **Hypervisor → Manages VMs**
