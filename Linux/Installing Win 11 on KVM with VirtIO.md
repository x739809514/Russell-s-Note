### **1. Initial Setup**

- **Environment:** Linux Host with QEMU/KVM (using Virtual Machine Manager).
- **Guest OS:** Windows 11 (ISO).
- **Virtual Hardware Configuration:** To ensure maximum performance, the Virtual Machine (VM) was configured to use the **VirtIO** bus for both the Hard Disk and the Network Adapter, rather than legacy SATA or Intel e1000 emulation.

[[How to Install and Use KVM & Virt-Manager to Run a Windows Virtual Machine]]

---

### **2. Issue #1: No Storage Drive Found**

**The Problem:**  
During the Windows installation setup, when reaching the "Where do you want to install Windows?" screen, the list was empty. Windows could not see the virtual hard drive.

**The Root Cause:**  
Windows 11 installation media does not include native drivers for the **VirtIO SCSI controller**. Even though the virtual disk existed, the installer could not communicate with it.

**The Solution:**

1. Mounted the **`virtio-win.iso`** (Red Hat VirtIO driver disk) to the VM alongside the Windows installer ISO.
2. Clicked **"Load driver"** in the Windows setup menu.
3. Browsed to the CD drive (`virtio-win`) -> `amd64` -> `w11` -> `viostor` (or simply searched the root of the drive).
4. Installed the **Red Hat VirtIO SCSI Controller** driver.
5. **Result:** The drive appeared, and installation proceeded.

---

### **3. Issue #2: Stuck at Network Setup (OOBE)**

**The Problem:**  
After the initial installation and reboot, during the "Out of Box Experience" (OOBE) setup wizard, Windows reached the "Let's connect you to a network" screen.

- No Wi-Fi or Ethernet connection was detected.
- The "Next" button was greyed out.
- Windows 11 Home/Pro forces an internet connection to proceed.

**The Root Cause:**

1. **Missing Driver:** Similar to the storage issue, Windows 11 does not have the native driver for the **VirtIO Network Adapter (NetKVM)**.
2. **Virtualization Concept:** Even though the Host machine used Wi-Fi, the VM sees a virtual wired Ethernet connection. Without the specific VirtIO driver, the VM cannot "see" the cable.

**The Solution (Part 1: Bypassing the Requirement):**

1. On the network screen, pressed **`Shift` + `F10`** to open the Command Prompt.
2. Typed the command: `OOBE\BYPASSNRO` and pressed Enter.
3. The system restarted.
4. Upon returning to the network screen, a new option appeared: **"I don't have internet."**
5. Selected **"Continue with limited setup"** to create a local account and enter the desktop.

**The Solution (Part 2: Fixing the Driver):**

1. Once on the desktop, opened **Device Manager**.
2. Located the **"Ethernet Controller"** with a yellow warning icon (⚠).
3. Right-clicked -> **Update Driver** -> **Browse my computer for drivers**.
4. Selected the **`virtio-win` ISO drive** and checked **"Include subfolders"**.
5. Windows automatically detected and installed the **Red Hat VirtIO Ethernet Adapter**.
6. **Result:** Internet connectivity was immediately established.

---

### **4. Final Optimization**

**The Problem:**  
Device Manager showed remaining devices with yellow warning icons (e.g., "PCI Device", "Unknown Device").

**The Root Cause:**  
These were other VirtIO components (specifically the **Balloon Driver** for memory management and **VirtIO Serial** for host-guest communication) lacking drivers.

**The Solution:**

- Repeated the "Update Driver" process for all remaining unknown devices, pointing the installer to the `virtio-win` ISO each time.

### **Summary Conclusion**

Installing Windows on KVM with VirtIO hardware offers the best performance but requires manual driver intervention. Since Windows cannot natively detect VirtIO hardware, the `virtio-win` ISO is essential for:

1. **Storage:** During installation (to see the disk).
2. **Network & System:** Post-installation (via Device Manager) to enable connectivity and optimize memory management.