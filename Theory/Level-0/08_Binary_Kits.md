# Binary kits: brute-force lying

The term "binary kit" (sometimes referred to as a 'rootkit' or a 
'dropper kit') typically refers to a pre-packaged collection of 
malicious binaries and scripts designed to automate the deployment, 
concealment, and persistence of a rootkit on a target system.

While a "rootkit" is the technology used to hide presence, the "binary 
kit" is the delivery and management vehicle.

## 1. The Core Components of a Binary Kit

A binary kit is rarely a single file, it's a modular package that 
usually includes:

* The Rootkit Core: 

The actual kernel module (LKM for Linux) or driver (SYS for Windows) 
that performs the hooking and hiding.

* Modified System Binaries: 

These are "trojanized" versions of standard system tools. For example, 
a kit might replace `ls`, `ps`, `netstat`, or `top` with versions that 
filtered out the rootkit's files, processes, and network connections.

* The Dropper/Installer:

A specialized binary (often obfuscated) that checks the kernel 
version, determines the architecture, and handles the "dirty work" of 
loading the rootkit into kernel space.

* Backdoor Binaries:

Pre-configured shells (like a setuid shell) or network listeners that 
provide the attacker with remote access once the kit is installed.

* Configured Scripts: 

Shell scripts or small C binaries that automate the cleanup of logs 
(`/var/log/messages`, etc.) and ensure the kit restarts after a 
reboot.

## 2. Interaction with the Kernel

Binary kits are designed to bridge the gap between user space and 
kernel space.

### The Deployment Process 

1. Enviornment Discovery: The kit's installer runs `uname -a` or 
checks the Windows registry to see if the kernel is compatible. Binary 
kits for kernels are highly version-specific; running a kit built for 
Kernel 5.4 on Kernel 6.1 will often cause a *Kernel Panic* (BSOD).

2. Privilege Escalation: If the attacker doesn't already have 
root/SYSTEM access, the kit may include an exploit binary specifically 
to gain those rights.

3. Loading: The kit uses system calls like `init_module()` (Linux) to 
push the binary into kernel memory.

### Kernel Object Manipulation (DKOM)

Advanced binary kits don't just "replace" files, they include binaries 
that perform Direct Kernel Object Manipulation. These tools reach into 
kernel's internal linked lists (like the process list) and unhook the 
malicious process so the kernel itself doesn't "know" it's running.

## 3. Comparison: Rootkits vs Binary Kits

It is helpful to view these as the "payload" vs the "toolbox":

### Features of a Rootkit -

* Primary Goal:   Stealth and Persistence
* Location:       Mostly Kernel Space (Ring 0)
* Function:       Hooking syscalls, hiding files
* Format:         `.ko` (Linux) or `.sys` (Windows)


### Features of a Binary Kit - 

* Primary Goal:   Deployment and Automation
* Location:       Both User Space and Kernel Space
* Function:       Installing, cleaning logs, providing backdoors
* Format:         A `.tar.gz` or `.zip` containing many binaries

## 4. Detection and Defensive Perspectives

Because binary kits often replace standard tools, defenders use 
specific techniques to catch them:

* Signature Matching: 

Since many kits are reused (like the Suckit or Adore-ng kits), EDR and 
AV can flag the specific MD5/SHA256 hashes of the binaries in the kit.

* Integrity Checking:

Tools like `AIDE` or `Tripwire` compare the hashes of system binaries 
(like `/bin/ps`) against a known-good database. If a binary kit has 
replaced them, the hash mismatch is a "dead giveaway".

* Memory Analysis:

Since the kernel-level binaries hide themselves from the OS, forensics 
experts use tools like *Volatility* to dump the RAM and look for 
"orphaned" processes that aren't in the standard process list but are 
executing code.


*A binary kit replaces trusted tools with modified ones.*

#### For example:

* `ls` hides files
* `ps` hides processes
* `netstat` hides sockets

This works because humans trust tools, but the kernel still knows the 
truth.

#### That is why defenders:

* compare outputs
* verify hashes
* cross-check with /proc

Binary kits are simple, noisy, and historically important.
