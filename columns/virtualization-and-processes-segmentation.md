---
title: Virtualization and Processes' Segmentation inside x86
author: Matteo De Paoli
published: 2013-05-20
updated: 2013-05-20
tag: [bios, cpl, cpu, exception, gdt, idt, int, interrupt, isr, ivt, ldt, mmu, register, rings, syscall, tlb, uefi, virtualization, x86]
headline: 
image: http://4.bp.blogspot.com/-HVpgea7zK3A/VlyBbRRc7QI/AAAAAAAALkw/507bLnmC4kA/s1600/rings.png
lead: Nowadays, our OS runs multiple programs at the same time, and even multiple (virtual) machines may run atop of it. With a so huge sharing of the physical resources, why the code of the Nginx service that runs on an Ubuntu virtual machine inside our VMware ESXi host doesn't mess up with the resources of other services? After all, what does limit a generic machine code "injected" in the CPU to control whatever it wants and run arbitrary stuff? What does "user space" actually means? It's the CPU and OS duo that, with the help of MMU, manages this paramount mechanism! This column will cover the basics around x86 technologies that permits segmentation of the machine resources used by the applications running into our boxes, from MMU up to the OS user space concepts.

---

## Abstract


### BIOS, UEFI and the early Boot Process


In order to make this mechanism effective, the segmentation must set some components up as soon as the machine boot, in order to avoid that other code has the ability to do the same.

To understand this better, we need to explore the big-bang moment of a computer, when everything start.

[BIOS (Basic Input/Output Software)](https://en.wikipedia.org/wiki/BIOS) is a firmware, a pre-OS environment, that is present in IBM compatible PCs.  
It's specifically designed for each motherboard and resides into one of its chips, and **it’s almost the first code ran when PC is powered on**.  
After performing some checks on the hardware, **it boots the OS** (other than providing some really basic APIs).

>**NOTE**: The BIOS code is trusted, and no restriction is applied when it runs immediately after machine boot. For this reason, a malicious code installed here is one of the frightening things that could happen.

**To boot the OS the BIOS read the first sector** (ie. [Cylinder 0, Head 0, Sector 0](https://en.wikipedia.org/wiki/Cylinder-head-sector)) **of the hard drive**, called “boot sector”, that is the first part (446bytes) of the Master Boot Record ([MBR](https://en.wikipedia.org/wiki/Master_boot_record)) that contains also the MBR partitions schema (for 512 bytes in total).  
If “boot sector” ends with the “magic number” 0xaa55, the preceding data are considered code and the code is ran (the “magic number” is needed because there is no way to distinguish data from code).  
The boot sector is too small to contain most of the modern boot loaders, and so it usually loads the actual [boot loader](https://en.wikipedia.org/wiki/Booting#BOOT-LOADER) from another configured location (a.k.a. [chain loading](https://www.gnu.org/software/grub/manual/html_node/Chain_002dloading.html)).  
If you want to copy the MBR you can do:
    
    :::bash
    # dd if=/dev/sd$ of=/$your_backup_path bs=512 count=1

>**NOTE**: The "**boot sector**” is where the control of the machine resources is passed from the BIOS code to the OS code, and so, it's the **lasted frightening place where a malware could be installed before the OS takes control** and set the segmentation mechanism up, along with its own security features (including anti-viruses and so on).

Nowadays, **BIOS is almost replaced by [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)** and by the [GPT partition schema](https://en.wikipedia.org/wiki/GUID_Partition_Table).

**UEFI is much more than a basic pre-OS environment**, with tons of features and without most of the BIOS limits; particularly, as far as we're concerned, it has its own UEFI executable format, along with its **"[EFI system partition](https://en.wikipedia.org/wiki/EFI_system_partition)" (ESP)** where we can store the **EFI executables** (formatted as FAT12, FAT16 or FAT32).

>**NOTE**: UEFI and [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Intel_EFI) are not the same thing, the former is the most recent open standard, the latter was an initial proprietary implementation by Intel. But nowadays, EFI is often used to indicate an executable format used by UEFI.

UEFI, by design, must have a global boot variable stored in NVRAM where you can specify, either from the UEFI itself or from inside the OS, the proper boot loader code (or even the kernel) to load. In other words, UEFI has its own configurable boot loader.  
If the boot variable is not configured, the firmware inspects the ESP to find an EFI executable to run; anyway, this is more a fallback mechanism not to be used for booting permanently installed OS.  
To show UEFI info from Linux you can use:

    :::bash
    # efibootmgr -v

**Another interesting feature of UEFI is "[secure boot](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Secure_boot_2)"**, that through the use of digital certificates, allows a strict verification of the code allowed to run.

>**NOTE**: "secure boot" feature was very criticised initially, but mostly because of the misunderstanding of its capacity to avoid the huge security issues related to the BIOS/MBR booting mechanism mentioned above, where malicious code could be installed before that the OS turns its shields up.

That said, **with BIOS and even more with UEFI, the first code that is loaded after power on is very deterministic** and can be also verified, **and this is the first foundation for a proper segmentation** of the machine resources.


## Interrupts


Now that we know how the OS is loaded in order to ensure it's the only owner and manager of the machine resources, we need to delve into the **mechanism used for the communications with the CPU**, that is the big OS partner in the resource segmentation.

**Interrupts are signals to the processor** emitted by hardware through an interrupt request (**IRQ**) indicating the need of immediate attention; **the processor responds by suspending its activities** (while saving their state) **and by calling the “[interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)” function** (a.k.a. Interrupted Service Routine - ISR) related to the event.  
When ISR function ends, processor re-start normal activities.

ISR functions are pieces of software that are mapped to the proper event in the [Interrupt Descriptor Table (IDT)](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) (ie. x86 implementation of an [Interrupt Vector Table - IVT -](https://en.wikipedia.org/wiki/Interrupt_vector_table)), that is located at a defined location of the system memory.  
Every row of the IDT is called "Vector", and is assigned an index from 0 to 255 (it can be seen as an array).

| Vector range          | Use
|-------------          |--------------
| 0-19 (0x0-0x13)       | Non-maskable interrupts and exceptions
| 20-31 (0x14-0x1f)     | Intel-reserved
| 32-127 (0x20-0x7f)    | External interrupts (IRQs)
| **128 (0x80)**        | **Programmed exception for system calls**
| 129-238 (0x81-0xee)   | External interrupts (IRQs)


Actually, the "Vectors" contain pointers (ie. "Gate Descriptors") to the memory segment containing the code that exec the actual ISR function, rather than containing the full code of the function. We will talk about the characteristics of the memory area pointed by these vectors in the "Segments" section below.

A typical IRQ is produced by keyboard when a key is pressed, in order to alert the OS ultimately. **Contrarily to the Exceptions, the number of IRQ requests are limited** by the number of IRQ lines to the processor.

Interrupts are really important in multi-tasking and especially in real-time systems, called interrupt-driven (eg. Cisco RSP processor is interrupted for non-ASIC switching).

>**NOTE**: in Linux, [smp_affinity(Symmetric MultiProcessing)](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/s-cpu-irq.html) is used to associate interrupt numbers to core (or set of cores) in order to balance the requests through “irqbalance” daemon (eg. all the interrupts from eth0 to core1, while eth1 to core2).


### Exceptions


Exceptions are **[similar to interrupts but they don't come from hardware](http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-1-manual.pdf)**; however, the term interrupt is often used to reference to the exceptions as well, and **they're mapped in the same [Interrupt Descriptor Table (IDT)](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) used by the hardware interrupts**.  
However, sometimes their exception handler functions are called also Exception Service Routines (ESR), and the IDT is referred as "Exception Vector Table" (mostly derived by non-x86 terminology/implementations).

There are 2 main types of exceptions in x86:

- **Processor detected**: similar to the hardware interrupts but rather than arriving to the CPU from the external they’re raised internally. They’re classified in **traps, faults and aborts** (eg. a request to the [Arithmetic Logic Unit (ALU)](https://en.wikipedia.org/wiki/Arithmetic_logic_unit) of a CPU of a divide by 0 raises a divide-by-zero exception on the CPU).
- Programmed: also called “**software interrupts**”, they are triggered by specific x86 instructions (eg. “[INT](https://en.wikipedia.org/wiki/INT_(x86_instruction))”).


## System Calls

Now that we know the interrupt mechanism used for messaging between CPU and OS, let's understand **how the applications communicate with the OS**, Linux in particular, **in order to request the machine resources they need**.

**System Calls are Interfaces ([API](https://en.wikipedia.org/wiki/Application_programming_interface)) between an application and the kernel**, which can be used to give to applications running in user space the access to a protected resource.

>**NOTE**: Indeed, **the segmentation we're talking about gives to user-space applications no direct access to the data structures that control the machine resources** (eg. files, network sockets, hardware, etc.), and later on we will discuss how this is enforced.

On low level, the **System Call is performed by the application by placing the proper SysCall number/code (each OS has its own SysCall table with the code mappings) in the [EAX register](https://en.wikipedia.org/w/index.php?title=X86&redirect=no#x86_registers)** of the CPU (for x86/32bit), while the parameters of the SysCall are placed in the following register in the following consecutive order:

1. E**B**X
2. E**C**X
3. E**D**X
4. ESI
5. EDI
6. EBP

If more than 6 parameters are needed, or if parameters are too big for registers, parameters can be stored in memory and EBX will point to the address of the first one.  
Then, **when registers are set, the application calls the CPU instruction “[INT 0x80](https://en.wikibooks.org/wiki/X86_Assembly/Interfacing_with_Linux#int_0x80)”** (for Linux OS), causing the CPU to pass the control to the kernel that performs the service requested (if application is allowed), and returns to the application.  
What happens practically is:

1. The user-space application triggers a "software interrupt" with "int 0x80" to the CPU (as we said just above)
2. Than, the CPU looks to the Vector (ie. row) N.128 (that is the decimal value for 0x80) of the IDT, that contains the "System Gate" segment descriptor pointing to the ISR handler function of the OS to run for the System Calls (ie. a pointer to the memory area containing the code to run)
3. After that, the CPU runs the ISR code, practically giving the control of the machine to the OS (as the ISR code is part of the Linux kernel actually)
4. The OS looks at the System Call Code and to the other parameters initialized by the user-space app, and perform the requested stuff
5. Finally, if everything is fine, the expected result of the specific System Call and the CPU control are returned by the OS to the user-space application

>**NOTE**: More recently (starting with Pentium Pro CPU actually), the **"[sysenter](https://en.wikipedia.org/wiki/Call_gate_(Intel)#Modern_use)"** instruction was added to the Instruction Set Architecture (ISA - ie. the machine instructions supported by the CPU) along with some specific registers, to **replace "int 0x80"** in order to speed up the System Calls.

SysCall are usually invoked by [wrapper functions](https://en.wikipedia.org/wiki/Wrapper_function#Library_functions_and_system_calls) that programming language libraries have (often the name of the wrapper is the same of the system call it implements).
Worth to note also that, in most systems, system calls are allowed only from user space.

## On Chip Technologies

Now that we know the interrupt mechanism used for messaging between CPU and OS, and the System Call mechanism available to the user-space applications to request access to the machine resources, let's understand **how it's possible to enforce the user-space application to run respecting these rules, eliminating any change that they overtake the OS** to access the machine resources arbitrarily.

Before Intel 286 CPU, x86 processors operated only in **“[Real Mode](https://en.wikipedia.org/wiki/Real_mode)”**, without any **"[Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory)"** support and without any other feature that could limit the resources available to a process running in the CPU (actually "Real Mode" terminology was introduced later on to distinguish old mode from the new one).  
Someone could recall when on some ancient computers you were forced to reboot the machine when an application ended its tasks; however, even on a bit more advanced OSs, a running application could mess up the status of other applications that were paused in background (for instance, by arbitrarily writing its own data in the RAM portion used by another application).

### Segments, Virtual Memory and MMU

#### Segments

[Memory of 8086 CPUs and later is segmented](https://en.wikipedia.org/wiki/Memory_segmentation#x86_architecture), mostly to extend memory usage over the 16bit size of a single register (that was able to represent only *2^16=65KB* of RAM in total).

It means that the total **memory is logically split in segments and assigned to programs. To select a particular area inside the segment an "offset" value is used** (eg. the 16th byte of the 2nd segment).  
Until Virtual Memory, [Global and Local Descriptor Tables (GDT and LDT)](https://en.wikipedia.org/wiki/Global_Descriptor_Table) were used to maintain segment to physical address mapping.

**A program uses one or multiple segments** and usually each segment can contain either data or code, **organised logically** so that each function, class, or any other part of the program has its own segment.

From 286 CPU, **each segment is bounded to the process and can also have proprieties** like executable, read only, etc. A **violation of these constraints may raise a "[Segmentation Fault](https://en.wikipedia.org/wiki/Segmentation_fault)"**.

>**NOTE**: In same cases, some particular segments can be shared across different processes.

**Segment registers** (ie. a particular type of CPU register) **contain the "segment selectors"** (ie. index values mapped in GDT/LDT) **used to access the memory** used by the segments of a program; particularly, the **[Code Segment](https://en.wikipedia.org/wiki/Code_segment) (CS/Text) register** is filled by the OS and points to the segment assigned to the code of the program (or part of it).
On the other side, the [Instruction Pointer (IP or EIP/RIP in 32 and 64bit)](https://en.wikipedia.org/wiki/Program_counter) is the offset from the segment start to the  next byte of code that CPU will fetch.

So, **"CS:IP" form is used to represent the memory location from which the code has to be loaded in the CPU, and it's read only** (set by the CPU as consequence of instructions that alter the flow of the program).

The data contained in these segments of memory are a very important part of the segmentation of the resources available to the user-space applications, and we will look at these data further on the "Protection Rings" section of this column.


#### Virtual Memory

Implemented in x86 since 286 CPU (but really improved with paging in 386 CPU), it’s a memory model where physical memory (RAM) addresses are divided in chunks called “pages”, and these pages are mapped to **“virtual addresses”**.  
**While "segments" can be seen as a way to define logical part of a program, "pages" are oriented to the management of the physical memory, and they work in tandem**.

Each process is presented with a range of virtual addresses that is not related to the geometry of the physical memory and that is strictly bounded to the process itself; because of this, the available memory seen by a process may be as big as the full system physical memory or even more (by using the swap space in the hard drive), and virtual addresses used by different processes can overlap.  
As consequence, **virtual memory addresses do not need to be mapped/allocated to anything until they are allocated by a program** (allowing lazily-loading and over-provisioning); if more space needs to be allocated, a new page is allocated for the specific segment.  
This way **a process doesn’t need to take care of managing the access to a memory shared with other processes and can't (shouldn't) interfere with other processes** running on the same system.

The mapping between virtual and physical addresses is maintained by the “page table”.

Actually, not only the processes but the processor (CPU) itself writes and reads data through “virtual memory addresses” and also because of this, the page table is usually stored in an **“hardware cache”** called **[Translation LookasideBuffer (TLB)](https://en.wikipedia.org/wiki/Translation_lookaside_buffer)**, implemented usually as a [CAM](https://en.wikipedia.org/wiki/Content-addressable_memory) inside the **MMU** (Memory Management Unit) of the CPU.  
Anyway, **it’s up to the OS to populate and maintain the page table**.

If a requested virtual address is not on the TLB, a “TLB miss”/”TLB fault” exception is raised by the CPU, and the OS needs to fix the issue.

On the other side, a **"[Page Fault](https://en.wikipedia.org/wiki/Page_fault)" is an exception raised by the MMU** when a program tries to access a memory page that is mapped in the TLB but that’s not loaded in the physical memory (RAM), usually because the page is not in RAM and the kernel need to allocate it from scratch or by paging-in from the swap (eg. when an application first start up, or when an application that uses lazily-loading requires a page that it hadn’t loaded at start).  
**This MMU exception should be managed by the kernel that tries to make the specific page available in RAM**, or terminates the program in case of an illegal memory access/access violation (**segfault**/segmentation fault).  
In other words, by itself, **a page fault is not an error contrarily to segfault** we spoke about in the "Segments" section.

>**NOTE**: Usually Linux does a context switch and put the process that triggered the page fault in background (assuming a valid page was requested) and continue to run other processes until the requested page is loaded in RAM.

##### Shared Memory

Thanks to segments and virtual memory, **a physical memory space can be mapped with different virtual pages and segments, which are than mapped to multiple processes**. This way, things like software libraries (collections of functions of code) can be shared among different applications making efficient use of the memory.  
**The read-only flag of a shared segment can be also used to implement the [Copy-on-write (COW)](https://en.wikipedia.org/wiki/Copy-on-write)**, by sharing read-only segments between processes and by triggering a copy of this segment to a new (read/write) segment dedicated to the process that has triggered the write, while maintaining the original read-only segment shared between the other processes.

## Protection Rings

All these long abstracts are useful to understand the details of the main technology used to segment the machine resources, the "Protection Rings" (a.k.a. privilege levels).

Along with the MMU, 286 CPU introduced the **"[Protected Mode](https://en.wikipedia.org/wiki/Protected_mode)"**, to overcome the issue of the so called "Real (operation) Mode" of previous processors.  
Logically, different rings were implemented in the CPU in order to segment resources available to the applications.

![Protection Rings](http://4.bp.blogspot.com/-HVpgea7zK3A/VlyBbRRc7QI/AAAAAAAALkw/507bLnmC4kA/s1600/rings.png){: style="float:left; width: 40%;"}

**RING 0 is called “kernel mode”** or “supervisor mode”, and it's used by the kernel of the OS and by its libraries (privileged access to memory, registers, disk I/O, etc.).

**RING 3 is called “user mode”** and it's used for processes in “user space”. Usually RINGs 1 and 2 are unused.

Software running in “user mode” must access protected resources through the **“System Calls”** to the kernel, as we said above.

**Along with MMU, it may be considered one of the first virtualizations on x86**, segmenting each process in its own memory space and ensuring it can’t interfere with the outside world unless System Calls (managed by the OS).

The resources that it protects are:

- Memory
- I/O
- Available machine instructions within the IAM

> **NOTE**: Actually, with MMU, every process has a virtual view of the memory in which it can access any virtual address it wants without any issue. Because of this, protection of the segments used by different programs is not the big deal here but still be important to ensure that properties of the segments are respected.

**At any time, the CPU is running at a specific privilege level**.

Basically, privilege levels are implemented in memory through the use of the **lower 2 bits of some "segment selectors"** and by **limiting the available ISA** accordingly.

For instance, the "segment selector" stored into the CS register and into the SS (Stack Segment) register contain the [Current Privilege Level (CPL) field, stored on the lower 2 bits](http://www.intel.com/Assets/en_US/PDF/manual/253668.pdf), that represents the protection level/ring of the running code.  
Just to remark what said above in the "Segments" section, the **CS field can be modified only by the CPU** and so the CPL in it.  
Data and instructions (ie. their **memory segments) required by the code of that process must be compliant with its current privilege level (CPL)** and with the requester privilege level (RPL) , otherwise a General Protection (GP) exception is raised, avoiding the process to perform a privilege level escalation.

**CPU restricts also many of its machine instructions** (ie. Instruction Set Architecture - ISA), or just restricts specific parameters of them, based on the CPL value.

>**NOTE**: For compatibility with old x86, all the CPU starts in “Real Mode” when initialised by the BIOS, then the OS switches to “Protected Mode”, by setting the PE (“Protection Enabled”) bit of the CR0 (Control Register) to 1.


### OS Role

**When the OS (ie. kernel) boots, it sets up the IVT and EVT (ie. IDT in x86) to point their ISRs/ESRs to its own functions; this way the kernel ensures that it will manage all the interrupts, and consequently the change of the CPL value**.

In normal operations, the **applications inside the OS should transfer the control to the kernel only through the “System Gates”** (eg. software interrupt with code 0x80 or "sysenter").  
A software **interrupt from user-space to another Vector code not mapped to a "Gate descriptors" destined to the user-mode ring will trap an exception** usually.

Either way, if a process tries to access a resource of a more privileged ring, an exception is raised and the kernel takes the control of the CPU and manage the situation.

Also, **when the kernel gives an application the CPU control, it sets a timer** that causes an exception after a specific time, allowing the kernel to take the control of the CPU back; because of this, “Protection Mode” is also at the roots of the **multitasking** and of the “running in background” concepts.

Also, repeating what's said above, OS takes care of the management of the paging table of the MMU as well of the initialization of the segments (and related flags) used by a program.

### Hardware Virtualizations

Historically, 386 had a "virtual mode" (a.k.a. "virtual real mode" or vm86 mode) to allow programs written for "real mode" to be ran under an OS running in "protected mode" (mostly for programs used to access machine resource directly, without system calls, and for which vm86 intercepts the requests and acts as proxy).

The more recent techniques were introduced for a different purpose, primarily to **permit virtual machines running on the top of an hypervisor to work fast without modification of the guest OS**.  
The problem arose because **with x86 some machine instructions would act differently if executed in kernel mode (ring 0) or user mode**; this means that access to protected resource from a user space process may not trigger exceptions/interrupts (that could be captured from the hypervisor), and also that the results of the operation returned to a guest OS may be different from the one received by an OS that deals with the hardware directly.

The software alternatives for a **"Full Virtualization" requires the hypervisor to scans/catch the guest code and rewrite/translate the kernel mode instructions (a.k.a. privileged binary instructions) in a process called “ring deprivileging”**. The problem with this approach is in performance mostly (other than adding further complexity to the hypervisor).

Another software approach is called **"[Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization)"**, used notably by Xen.
With Xen Paravirtualization (PV), the **OS inside the VM (guest/domU) is modified in order to produce hyper calls** (directed to the hypervisor) rather than trying to access the hardware directly.

In order to avoid both problems of rewriting the guest OS (paravirtualization) and the overhead of the software full virtualization, **Hardware-assisted virtualization was developed on x86 CPU (Intel VT / flag vmx, AMD Pacifica / flag svm)**.

A ring -1 is added and used by the hypervisor (a.k.a. ring 0P – Privileged/Root), while guest OS run on the ring 0 (a.k.a. ring 0D – Deprivileged/Non-Root).

Privileged and sensitive action to the ring 0 trap to the hypervisor at ring -1, avoiding the need for the hypervisor to scan/catch all the instructions of the OS.

To check if your CPU does support these technologies:
    
    :::bash
    # egrep '(vmx|svm)' /proc/cpuinfo

