# Rootkit

*In advanced exploit development, we are gonna see that a rootkit is 
essentially a race for the "highest ground" on a system. Whoever 
controls the lowest layer of the software stack controls the truth of 
every layer above it.*

## 1. Rootkit Hierarchy (Rings of Control)

Rootkits are categorized by which 'Ring' they inhabit. In modern 
architecture, the lower the number, the more privilege the code has.

* **User-Mode Rootkits (Ring 3):** These are the most common but the 
easiest to detect. They work by **API Hooking**. When a user runs a 
command like `tasklist`, the rootkit intercepts the call to 
`NTDLL.dll` and filters the results before they reach the screen.

* **Kernel-Mode Rootkits (Ring 0):** This is your primary area of 
interest. These live in the OS kernel itself. They don't just "lie" to 
applications, they manipulate the operating system's internal data 
structures.

* **Hypervisor Rootkits (Ring -1):** These "virtualize" the entire OS. 
The rootkit runs a thin hypervisor (like a malicious version of KVM or 
Xen), and the original OS becomes a "guest" inside it. The OS thinks 
it has full hardware access, but every action is intercepted by the 
rootkit.

* **Firmware/Bootkits (Ring -2/-3):** These live in the UEFI/BIOS or 
the SMM (System Management Mode). They execute before the OS even 
starts, making them persistent even if you wipe the hard drive.

## 2. Kernel-Level Persistence and Stealth 

*To maintain control, a kernel rootkit must perform two main tasks: 
**Concealment** and **Persistence.***

### **Concealment: The Art of the Lie**

In the kernel, a rootkit can hide using several advanved techniques:

* **SSDT Hooking (System Service Descriptor Table):** The SSDT is a 
table the kernel uses to find the addresses of system functions. A 
rootkit patches this table to point to its own code. When the OS tries 
to "read a file", it inadvertently runs the rootkit's "filtered read"
function first.

* **DKOM (Direct Kernel Object Manipulation):** Instead of hooking 
functions, the rootkit modifies the kernel's memory directly.

	* Example: To hide a process, the rootkit finds the `EPROCESS` 
	linked list in Windows (or `task_struct` in Linux) and 
	"unlinks" its own entry. The process still runs, but the 
	kernel's "list of processes" no longer contains it.

* **IRP Hooking (I/O Request Packets):** For Windows drivers, rootkits 
intercept the packets sent to hardware drivers (like the disk driver). 
If a request comes in to read the sector where the rootkit is stored, 
the rootkit modifies the packet to return "zeroes" or "empty space".


### Persistence: Surviving the Reboot

* **Kernel Module Loading:** On linux, adding an entry to 
`/etc/modules` or tampering with `initramfs` ensures the Loadable 
Kernel Module (LKM) loads at boot.

* **Filter Drivers:** Attaching the rootkit as a "Lower Filter" to a 
legitimate driver (like the keyboard or disk driver). Every time that 
hardware initializes, the rootkit initializes with it.

* **eBPF (Extended Berkeley Packet Filter):** A modern, highly 
stealthy technique. In newer Linux kernels (5.5+), attackers use eBPF 
programs to hook kernel functions without ever loading a traditional 
LKM, making traditional "lsmod" checks useless.


## 3. The Detection Battle

*Modern EDRs (Endpoint Detection and Response) and Windows 
"PatchGuard" have made rootkits much harder to deploy*

* **PatchGuard (KPP):** This is Windows feature that periodically 
checks if critical kernel structure (like the SSDT) have been 
modified. If it detects a change, it triggers a Blue Screen of Death 
(BSOD) to prevent further compromise.

*KPP stands for Kernel Patch Protection and also knows as PatchGuard*

* **Driver Signing Enforcement (DSE):** Windows will not load a driver 
unless it is digitally signed by a trusted authority. Red teamers 
bypass this using "**Bring Your Own Vulnerable Driver**" (BYOVD) 
attacks, loading an old, legitimate but vulnerable driver (like an old 
Dell or Capcom driver) and then exploiting that driver to gain kernel 
execution.
