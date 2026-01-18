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
