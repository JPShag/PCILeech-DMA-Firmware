# **Custom Firmware Development Guide for PCIe Endpoint Modeling**

---

**Shout out to the legend that donated, I will be in contact soon. I'm gonna put your name here if you want just DM me.**

Trying to organize this into a [wiki](https://github.com/JPShag/PCILeech-DMA-Firmware/wiki/Introduction) but honestly I can barely organize my own life right now so the wiki's coming along like everything else -- slowly and in between crisis moments. Help is welcome.

---

> **📌 DMA Threat Landscape 2024-2026:**
>
> Simple VID/DID spoofing on stock PCILeech firmware is no longer sufficient. Detection has evolved significantly. Swapping two hex values in Verilog and hoping for the best is not a strategy.
>
> **Fact tier -- confirmed:** The pre-boot DMA CVEs are real: CVE-2025-11901 (ASUS), CVE-2025-14302 (Gigabyte), CVE-2025-14303 (MSI), CVE-2025-14304 (ASRock), tracked as VU#382314 by CERT/CC. Affected UEFI firmware misreported IOMMU as active while hardware was NOT initialized during early boot. Riot's security team discovered this. Vanguard reportedly blocks game launch on unpatched boards (VAN:Restriction).
>
> **Fact tier -- observed/inferred:** IOMMU is required for FACEIT (confirmed 2025) and EAC competitive Fortnite (since Feb 2026). For Vanguard, Riot's June 2026 docs state IOMMU is required only for the *optional* Pre-Check/on-demand mode, not for the mandatory traditional driver. Community reports indicate anti-cheat kernels inspect PCIe config space for known signatures; specific offsets and signature content are not publicly documented. Honeypot memory regions are technically possible as a detection strategy but the observation mechanism varies by platform.
>
> **Fact tier -- speculative/unconfirmed:** In May 2026 Riot claimed Vanguard can interfere with unauthorized DMA hardware. The mechanism is not publicly documented; host-side software cannot generically reconfigure arbitrary FPGA endpoints without an endpoint-specific control surface. Treat as unverified.
>
> Build and test on DEDICATED ISOLATED HARDWARE. See [§19](#19-dma-advances-state-of-the-art-20242026).

----

**A Note from the Author:**

This is probably the last major version. I might do a stripped-down one for people who already understand hardware -- if you know what an FTDI chip does, I don't need to explain UART to you. This information needed to exist and nobody else was going to write it, so here it is. If it helped you and you're in a position to help back, that would mean something. If not, I get it.

This guide exists because real firmware development knowledge was scattered across forum threads, Discord servers that disappeared, pastebins that 404'd overnight, and gatekeepers who treated basic Verilog like state secrets. I've tried to put all of this together in one place -- the PCIe architecture, the Vivado workflow, the config space model, the TLP handling, the detection landscape, the shadow config technique, the whole thing from silicon to software. Knowledge doesn't expire. Understanding how the silicon works doesn't get patched out in an update.

---

## In Memoriam

![Ross](https://github.com/user-attachments/assets/de7f12fe-8992-4738-a6af-712dc48217ee)

This guide is dedicated to **Ross Freeman (1947-1989)**. Co-founder of Xilinx. Father of the FPGA. Dead at 44. This man invented a category of computing hardware that now runs AI accelerators, high-frequency trading systems, military radar, 5G infrastructure, and the Artix-7 chip in your FPGA card. All of it traces back to Ross Freeman in 1984 asking "what if the hardware could be reconfigured after manufacturing?" while the entire semiconductor industry told him he was crazy. At the time the whole race was toward smaller, faster, fixed-function ASICs, and this guy said: what if it could CHANGE? What if you could REWIRE it? They laughed at him. He was right. He got patent #4,870,302 and then died at 44 before he saw any of it. Posthumous National Inventors Hall of Fame induction in 2009. Ross Freeman gave the world programmable silicon.

*"The ultimate goal of the FPGA was to make programmable logic devices that could replace standard digital chips."* -- Ross Freeman

---

Also for **Terrence Andrew Davis (1969-2018)**. TempleOS. Wrote his own kernel, his own compiler, his own editor, his own graphics stack, his own programming language called HolyC -- one man, 100,000+ lines of code, while battling severe mental illness that made his hands shake so badly he could barely type.

Terry understood something the entire software industry forgot: COMPLEXITY IS A COPE. It's what mediocre people do to make you think they're smart -- they build things so complicated and abstract and overengineered that you can't understand them, so you assume genius. Academic journals do this. Xilinx does this. Vivado is FIFTY GIGABYTES of software for an IDE that programs an FPGA chip. FIFTY. Terry's ENTIRE operating system -- kernel, compiler, editor, games, graphics, networking stack -- was measured in MEGABYTES. He compiled his whole OS faster than Vivado can synthesize a basic PCIe core on a 16-core workstation. His computer from 2005 built an operating system in less time than your Ryzen 9 takes to generate a bitstream for a 35T Artix-7. You tell me who the madman is.

Terry passed away in August 2018 at age 48. He was homeless, living in a van. Fans brought him food and supplies, but he refused housing because Terry didn't want your house -- he wanted you to understand TempleOS. And nobody did. Not while he was alive. That's the pattern -- Ross Freeman gets the plaque after he's dead, Terry Davis gets the "actually he was a genius" articles after he's dead, and the living people building things right now are struggling and nobody notices until you're gone. I'm not Ross Freeman and I'm not Terry Davis, but I understand them. I understand building something while everything collapses. I understand doing the work anyway.

*"An idiot admires complexity, a genius admires simplicity."* -- Terry A. Davis

*"What people are going to read is, 'It's about a pathetic schizophrenic who made a crappy operating system.' My perspective is, 'God said I made His temple.'"* -- Terry A. Davis

RIP King Terry. They never figured it out.

---

> **Before you begin.** You're about to spend 40-60 hours breathing SystemVerilog, fighting Vivado's 50 GB install, and potentially bricking your FPGA at least once. If you think you're going to copy-paste hex values from a repo and be done by Tuesday -- this isn't that guide. Ross Freeman invented the FPGA in 1984. He's been dead nearly 40 years. You're standing on a dead genius's shoulders. The minimum respect is learning how the silicon actually works.

---

## **Table of Contents**

### **Part 1: The Silicon Doesn't Care About Your Feelings**

1.  [Introduction](#1-introduction)
2.  [Key Definitions](#2-key-definitions)
3.  [Device Compatibility](#3-device-compatibility)
4.  [Requirements](#4-requirements)
5.  [Gathering Donor Device Information](#5-gathering-donor-device-information)
6.  [Initial Firmware Customization](#6-initial-firmware-customization)
7.  [Vivado Project Setup and Customization](#7-vivado-project-setup-and-customization)

### **Part 2: Your Device's Passport, and Why a Forged One Is Not a Personality**

8.  [Advanced Firmware Customization](#8-advanced-firmware-customization)
    *   [8.1 Configuring PCIe Parameters](#81-configuring-pcie-parameters-for-emulation)
    *   [8.2 BARs: The Host Rents You an Address, Not a Warehouse](#82-adjusting-bars-and-memory-mapping)
    *   [8.3 Interrupts: Make Noise Like Real Hardware or Don't Pretend You're Alive](#83-emulating-device-power-management-and-interrupts)
9.  [Emulating Device-Specific Capabilities](#9-emulating-device-specific-capabilities)
10. [Completions: Where Bad Tags Go to Die](#10-transaction-layer-packet-tlp-emulation)

### **Part 3: Prove It Works Before You Call It Firmware**

11. [Building, Flashing, and Testing](#11-building-flashing-and-testing)
12. [The ILA: Vivado's Tiny Black Box Recorder](#12-advanced-debugging-techniques)
13. [Troubleshooting: When the Host Calls Your Device an Unknown Piece of Garbage](#13-troubleshooting)
14. [Timing, CDC, and Other Ways to Lose Your Weekend](#14-emulation-accuracy-and-optimizations)
15. [Best Practices for Firmware Development](#15-best-practices-for-firmware-development)
16. [Additional Resources](#16-additional-resources)
17. [Contact Information](#17-contact-information)
18. [Support and Contributions](#18-support-and-contributions)
19. [DMA Advances & State of the Art (2024-2026)](#19-dma-advances-state-of-the-art-20242026)

---



## **Part 1: The Silicon Doesn't Care About Your Feelings**

---

## **1. Introduction**

### **1.1 Purpose of the Guide**

The overarching goal of this guide is to empower you with the knowledge and practical skills to develop custom Direct Memory Access (DMA) firmware for Field-Programmable Gate Array (FPGA)-based devices. This specialized firmware allows your FPGA to accurately emulate the identity and behavior of other PCIe (Peripheral Component Interconnect Express) hardware devices. Such emulation is a powerful technique with profound implications across several advanced domains:

**Hardware Security Research**:
*   **Vulnerability Discovery**: By emulating a device, you can create a controlled environment to send malformed or unexpected data to host drivers, systematically fuzzing for vulnerabilities (e.g., buffer overflows, race conditions) that might be exploitable from a hardware peripheral.
*   **Driver Analysis**: Observe how operating systems and specific drivers interact with hardware. You can emulate devices with non-standard configurations or undocumented features to understand driver behavior, identify security assumptions, or reverse-engineer proprietary protocols.
*   **Side-Channel Analysis**: While more complex, an emulated device could potentially be programmed to assist in experiments related to information leakage through timing or power analysis, by precisely controlling peripheral operations.

**Red Teaming & Penetration Testing**:
*   **Bypassing Security Measures**: Emulate a seemingly benign or whitelisted hardware device (e.g., a common NIC or storage controller) to gain DMA privileges. Once achieved, this allows direct interaction with system memory, potentially bypassing endpoint detection and response (EDR) systems or anti-malware solutions that operate at higher software layers.
*   **Stealthy Persistence**: An emulated malicious device could offer a covert way to maintain access to a compromised system, as it might be harder to detect than software-based implants.
*   **Exploiting Trust Relationships**: Systems often have implicit trust in connected hardware. Custom firmware can exploit this by mimicking devices that are granted specific permissions or access.

**System Debugging & Diagnostics**:
*   **Reproducible Testbeds**: Create highly specific hardware scenarios to reliably reproduce elusive bugs that may only occur with particular device states or data patterns.
*   **Fault Injection**: Intentionally emulate faulty device behavior (e.g., incorrect TLP formation, delayed responses) to test the robustness and error handling capabilities of the host system and its drivers.

**Hardware Testing & Validation**:
*   **Driver Development**: Test new or modified drivers against an emulated hardware profile before physical prototypes are available, or to simulate a wider range of hardware variants than physically accessible.
*   **Compliance Testing**: While not a substitute for official compliance tests, an emulated device can help pre-verify certain aspects of PCIe protocol adherence.

**Legacy System Support & Interoperability**:
*   Emulate older, discontinued, or hard-to-source PCIe devices to keep legacy systems operational or to bridge compatibility gaps between different hardware generations.

By progressing through this guide, you will gain proficiency in:
*   Meticulously extracting identifying attributes and configuration details from a physical "donor" PCIe device.
*   Modifying and extending existing open-source FPGA firmware frameworks to model a subset of a PCIe endpoint's configuration space and MMIO behavior.
*   Configuring and utilizing the Xilinx Vivado FPGA development toolchain alongside Visual Studio Code.
*   Developing a solid understanding of the PCIe architecture's layered model, the mechanics of DMA data transfers, and the constraints of modeling hardware behavior in an FPGA.

> **What this guide does NOT deliver:** This guide teaches PCIe configuration-space modeling and generic BAR-backed MMIO behavior. It does not produce a functional clone of a NIC, NVMe controller, USB host controller, Wi-Fi adapter, or any other complex device. Real devices have proprietary register semantics, DMA descriptor engines, firmware protocols, radio/PHY/media behavior, and driver contracts that static config-space replication cannot satisfy. A Wi-Fi identity without a radio is an identity card, not a radio. An NVMe class code without an NVMe controller behind it gets you through enumeration but not through `nvme identify`. Know what you're building and what you're not.

### **1.2 Target Audience**

This guide is tailored for individuals who already possess a foundational to intermediate knowledge of computer systems, hardware principles, and software development. The content is technically demanding and assumes a capacity for detailed, low-level work. Specifically, it caters to:

*   **Firmware Developers**: Engineers aiming to design or adapt firmware for FPGAs, especially for applications involving high-speed data transfer (DMA) and direct hardware interface manipulation over PCIe. A background in Verilog/VHDL and experience with FPGA development tools are highly recommended.
*   **Hardware Engineers**: Professionals involved in the design, testing, or validation of PCIe-based hardware. This guide can help in creating sophisticated test harnesses or emulating components within a larger system design. Familiarity with PCIe protocol and digital design is expected.
*   **Cybersecurity Professionals & Researchers**:
    *   **Vulnerability Researchers & Exploit Developers**: Those looking to explore hardware-level attack surfaces or develop proof-of-concept exploits leveraging DMA. Understanding of OS internals, memory management, and driver architecture is crucial.
    *   **Red Team Members**: Operators seeking advanced techniques for system access, persistence, and data exfiltration by leveraging direct hardware manipulation.
    *   **Digital Forensics & Incident Responders**: While this guide is offensively focused, understanding these techniques can aid in recognizing and analyzing sophisticated hardware-based attacks.
*   **FPGA Enthusiasts & Advanced Hobbyists**: Individuals with prior FPGA project experience who are eager to tackle complex challenges like PCIe communication and hardware emulation. A willingness to dig into datasheets and technical specifications is key.

**Prerequisite Knowledge (Recommended)**:
*   Solid understanding of digital logic and computer architecture.
*   Familiarity with at least one Hardware Description Language (HDL), such as Verilog or VHDL.
*   Basic proficiency in a Linux environment (many tools and scripts are Linux-friendly) and familiarity with command-line interfaces.
*   Experience with a C/C++ or a scripting language (like Python) for host-side interaction or test scripting.
*   A conceptual understanding of operating system principles (memory management, drivers, interrupts).
*   Patience and a methodical approach to debugging, as firmware development can be intricate.

The learning curve can be steep, especially if PCIe or advanced FPGA concepts are new. However, the guide aims to break down complex topics into manageable steps.

### **1.3 How to Use This Guide**

This guide is segmented into three logically progressing parts designed to build your knowledge incrementally:

*   **Part 1: Foundational Concepts**: This initial part is crucial. It introduces the core terminology, the underlying principles of PCIe and DMA, the necessary hardware and software stack (including setup instructions for tools like Xilinx Vivado and the PCILeech-FPGA framework), and the initial procedures for acquiring vital information from your target "donor" device and making basic firmware modifications. You should work through this part sequentially and thoroughly.
*   **Part 2: Intermediate Concepts and Implementation**: (Forthcoming sections) Building on the foundation, this part will guide you through more advanced firmware customizations. Topics will include fine-tuning PCIe operational parameters, emulating device-specific registers and capabilities (such as power management states and Message Signaled Interrupts - MSI/MSI-X), and gaining an initial understanding of constructing and interpreting Transaction Layer Packets (TLPs).
*   **Part 3: Advanced Techniques and Optimization**: (Forthcoming sections) The final part will explore sophisticated debugging methodologies (including the use of Integrated Logic Analyzers - ILAs and external PCIe protocol analyzers), techniques for optimizing firmware performance and emulation accuracy, comprehensive troubleshooting for common and complex issues, and a critical discussion on best practices, particularly focusing on the security implications of developing and deploying emulated PCIe devices.

**Working Through the Guide**:
*   **Sequential Progression**: Especially for Parts 1 and 2, follow the sections in order, as later concepts build upon earlier ones.
*   **Hands-On Practice**: This is a practical guide. Actively perform the setup steps, code modifications, and experiments on your own hardware.
*   **Adapt to Your Environment**: File paths, specific device IDs, and software versions may vary. Understand the concepts behind the instructions to adapt them to your particular setup.
*   **Consult External Resources**: The PCIe specification and FPGA documentation are your ultimate references. This guide simplifies and directs, but deep dives often require consulting primary sources.
*   **Iterative Development**: Firmware development is rarely linear. Expect to iterate, debug, and refine your designs. Use the troubleshooting sections and debugging techniques extensively.

You will be working with HDLs (SystemVerilog in PCILeech-FPGA), FPGA synthesis and implementation tools (Vivado), and potentially host-side programming tools and PCIe analysis utilities.

---

## **2. Key Definitions**

A solid grasp of the following terminology is essential for navigating the complexities of PCIe device emulation and custom firmware development. These terms will be used extensively throughout the guide.

*   **DMA (Direct Memory Access)**:
    *   **Definition**: A fundamental feature of modern computer architectures that allows hardware peripherals (like network cards, GPUs, or your FPGA-based emulated device) to read from and write to the main system memory (RAM) directly, without involving the Central Processing Unit (CPU) for every byte transferred.
    *   **Significance**: DMA is crucial for high-performance I/O operations. By offloading data transfer tasks from the CPU, it frees up the CPU to perform other computations, significantly improving overall system throughput and efficiency. For this guide, your FPGA will leverage DMA to interact with the host system's memory, which is a powerful capability.

*   **PCIe (Peripheral Component Interconnect Express)**:
    *   **Definition**: A high-speed serial computer expansion bus standard designed to replace older bus standards like PCI, PCI-X, and AGP. It uses a point-to-point topology, with separate serial links connecting each device to the root complex (typically part of the chipset or CPU). Communication occurs via packets.
    *   **Significance**: PCIe is the dominant standard for connecting high-performance peripherals to motherboards. Understanding its protocol, layered architecture (Physical Layer, Data Link Layer, Transaction Layer), and configuration mechanisms is paramount for emulating any modern hardware device.

*   **TLP (Transaction Layer Packet)**:
    *   **Definition**: The fundamental unit of data exchange at the Transaction Layer of the PCIe protocol. TLPs are responsible for conveying requests (e.g., memory read/write, I/O read/write, configuration read/write) and completions (responses to requests) between PCIe devices. Each TLP consists of a header, an optional data payload, and an optional End-to-End CRC (ECRC).
    *   **Significance**: To emulate a device accurately, your FPGA firmware must be capable of correctly forming, transmitting, receiving, and interpreting TLPs that match the behavior of the donor device. Understanding TLP types, formats, and flow control is critical for advanced emulation.

*   **BAR (Base Address Register)**:
    *   **Definition**: Located within a PCIe device's Configuration Space, BARs are special registers used by the device to request address space resources from the host system. A device can have up to six 32-bit BARs (or fewer, or pairs of 32-bit BARs can form 64-bit BARs). These registers define the starting addresses and sizes of memory-mapped I/O (MMIO) regions or I/O port regions that the device uses to expose its registers and internal memory to the host CPU.
    *   **Significance**: When the host system enumerates a PCIe device, it reads the BARs to determine the device's memory and I/O requirements, then allocates and programs these BARs with the actual base addresses in the system's physical address map. Your emulated device must accurately define its BARs to match the donor device so that the host OS and drivers can interact with it correctly.

*   **FPGA (Field-Programmable Gate Array)**:
    *   **Definition**: An integrated circuit (IC) that can be configured by a designer or customer after manufacturing -- hence "field-programmable." FPGAs contain an array of programmable logic blocks and a hierarchy of reconfigurable interconnects that allow the blocks to be "wired together" to implement custom digital logic circuits.
    *   **Significance**: FPGAs are the core hardware used in this guide. Their reconfigurable nature makes them ideal for emulating other hardware devices, as you can define the precise logic and interfaces required to mimic the donor device's PCIe presence and behavior.

*   **MSI/MSI-X (Message Signaled Interrupts / Message Signaled Interrupts Extended)**:
    *   **Definition**: Mechanisms that allow a PCIe device to deliver interrupts to the CPU by writing a special message (a TLP, specifically a Memory Write TLP) to a system-defined memory address, rather than using dedicated physical interrupt lines (as in legacy PCI). MSI-X is an enhancement of MSI, offering more interrupt vectors and greater flexibility.
    *   **Significance**: Most modern PCIe devices use MSI or MSI-X for more efficient and flexible interrupt handling. Accurate emulation often requires implementing the chosen interrupt mechanism of the donor device, including configuring the MSI/MSI-X capability structures and generating interrupt messages correctly.

*   **DSN (Device Serial Number)**:
    *   **Definition**: A 64-bit globally unique identifier that can be optionally implemented by a PCIe device. If present, it's typically located in an extended capability structure within the device's Configuration Space.
    *   **Significance**: While not all devices have a DSN, some drivers or management software might use it for unique identification, licensing, or tracking purposes. Emulating it correctly can be important for full transparency and avoiding detection of the emulated device.

*   **PCIe Configuration Space**:
    *   **Definition**: A standardized 256-byte (for Type 0, endpoint devices) or 4KB address region associated with each PCIe function (a device can have multiple functions). This space contains vital information about the device, including its Vendor ID, Device ID, Class Code, Revision ID, BARs, capability pointers, and various status and control registers. It is accessed by the host system using special Configuration Read and Configuration Write TLPs.
    *   **Significance**: The Configuration Space is the "identity card" of a PCIe device. The very first step in device emulation is to meticulously replicate the relevant parts of the donor device's Configuration Space in your FPGA firmware. The host system uses this information to identify, configure, and allocate resources to the device.

*   **Donor Device**:
    *   **Definition**: The physical PCIe hardware device whose identity and behavior you aim to emulate on your FPGA. This device serves as the source for extracting configuration details (Vendor ID, Device ID, BAR settings, capabilities, etc.) and behavioral patterns.
    *   **Significance**: The fidelity of your emulation directly depends on how accurately and completely you can gather and replicate the characteristics of the donor device.

*   **Root Complex (RC)**:
    *   **Definition**: The entity in a PCIe hierarchy that connects the CPU and memory subsystem to the PCIe fabric. It generates PCIe transactions on behalf of the CPU and processes transactions initiated by downstream PCIe devices. It also performs the initial bus enumeration and configuration.
    *   **Significance**: Your emulated device will primarily interact with the Root Complex (or switches connected to it) when communicating with the host system.

*   **Endpoint (EP)**:
    *   **Definition**: A type of PCIe device that resides at the periphery of the PCIe fabric, consuming or producing data. Examples include network cards, graphics cards, storage controllers, and the FPGA device you will be programming. Endpoints request resources and initiate transactions to the Root Complex.
    *   **Significance**: In this guide, your FPGA will be programmed to act as an Endpoint device, emulating a specific donor Endpoint.

*   **HDL (Hardware Description Language)**:
    *   **Definition**: A specialized computer language used to describe the structure, design, and operation of electronic circuits, particularly digital logic circuits. Common HDLs include Verilog and VHDL.
    *   **Significance**: You will be working with Verilog (specifically SystemVerilog, an extension of Verilog) within the PCILeech-FPGA project to define the custom logic for your emulated device.

*   **Bitstream**:
    *   **Definition**: The final configuration file that is loaded onto an FPGA to program its logic blocks and interconnects, thereby implementing your custom hardware design. It's the compiled output from the FPGA development tools (like Xilinx Vivado).
    *   **Significance**: Generating and flashing the correct bitstream is the ultimate step in deploying your custom firmware onto the FPGA.

*   **IOMMU (Input-Output Memory Management Unit)** -- Intel VT-d / AMD-Vi:
    *   **Definition**: A hardware unit that provides address translation and access control for DMA-capable devices, analogous to how a CPU's MMU protects process memory. Each PCIe device can be assigned a set of page tables (a domain) limiting which physical addresses it can read or write.
    *   **Significance**: When IOMMU is enabled and properly configured, a DMA device cannot access arbitrary system memory -- it can only reach addresses mapped in its domain. A device hitting an unmapped address generates a translation fault. This is the primary hardware defense against unauthorized DMA. FACEIT and EAC competitive modes require IOMMU to be enabled. Vanguard's optional Pre-Check mode requires it.

*   **ACS (Access Control Services)**:
    *   **Definition**: A PCIe capability that controls peer-to-peer routing between devices. ACS can block downstream devices from sending TLPs directly to each other, forcing all traffic through the root complex.
    *   **Significance**: DMA cards connected via Thunderbolt or behind a PCIe switch need ACS enforcement to prevent peer-to-peer DMA bypass of IOMMU. FACEIT's IOMMU requirement typically also requires ACS to be enabled and enforced.

*   **VFIO (Virtual Function I/O)**:
    *   **Definition**: A Linux kernel framework that exposes direct device access to userspace through IOMMU-protected memory mappings. Used by QEMU/KVM for device passthrough to virtual machines.
    *   **Significance**: VFIO is the Linux mechanism for attaching an IOMMU domain to a device. When you see "IOMMU groups" in `lspci -vvv`, what you're seeing is the VFIO grouping. Automated firmware generation tools use VFIO to dump donor device config space from userspace without a kernel driver.

*   **TOCTOU (Time-of-Check, Time-of-Use)**:
    *   **Definition**: A class of race condition where a value is validated ("check") and then used ("use") with a gap between the two operations allowing the value to change. In DMA contexts, TOCTOU attacks exploit the window between when a hypervisor or IOMMU validates a DMA buffer and when the device actually reads/writes it.
    *   **Significance**: Modern DMA defense research (DMAArmor, Deferred DMA Attack) focuses on eliminating TOCTOU windows in DMA buffer lifetime management. A device that passes validation for a read to address X, then has its mapping revoked before the actual TLP arrives, creates a TOCTOU vulnerability.

*   **VMD (Volume Management Device)**:
    *   **Definition**: An Intel chipset feature that presents NVMe SSDs behind a PCIe bridge called a VMD controller. The OS sees the VMD controller rather than the individual NVMe devices behind it.
    *   **Significance**: A DMA firmware trick circa 2024-2025 was to present as an NVMe device hidden behind Intel VMD, hoping anti-cheat would only enumerate the VMD controller and not drill down. Vanguard now enumerates through VMD (confirmed May 2025). Do not rely on VMD as a hiding mechanism.

---

## **3. Device Compatibility**

Achieving successful and accurate PCIe device emulation hinges on ensuring your chosen FPGA-based hardware and host system configuration are fully compatible. This section details the supported FPGA platforms, critical PCIe hardware considerations, and the necessary system requirements to set up your development environment.

### **3.1 Supported FPGA-Based Hardware**

While this guide provides a generic methodology adaptable to various FPGA-based DMA hardware, our primary examples and specific instructions will focus on **Xilinx 7-series FPGAs**, commonly found in open-source DMA boards due to their balance of performance and accessibility.

The core principles and techniques for customizing the PCIe IP core and developing HDL code are broadly applicable to the following FPGA families:

*   **Squirrel / Screamer (Artix-7 35T)** -- entry-level, USB-C, sufficient logic for standard memory acquisition and basic emulation. Most common starting point.
*   **Enigma-X1 (Artix-7 75T)** -- mid-tier, more logic cells and BRAM for complex emulation.
*   **ZDMA (Artix-7 100T)** -- higher-performance, Thunderbolt, 1000+ MB/s, most on-chip resources in the Artix-7 line.
*   **Kintex-7 (K325T, K410T, etc.)** -- advanced tier, higher PCIe lane counts and speeds (Gen3 x8/x16), significantly more logic and DSP slices.

> **Newer 7-series boards that ship PCILeech-compatible firmware (2024-2026):**
> *   **CaptainDMA** (35T / 75T / 100T / 4.1th / **M.2**)  --  flashes via WCH CH347 / CH347_OpenOCD over an integrated update port, no separate JTAG dongle required. The M.2 100T variant pushes ~220 MB/s over USB-C.
> *   **Screamer / Squirrel** (35T)  --  the original workhorse boards. USB-C, ~190 MB/s.
> *   **IMMORTAL DMA** (XC7A35T, USB-C)  --  compact Artix-7 card, ~275 MB/s.
> *   **M.2 form factor**  --  sits on an M.2/NGFF slot or via an M.2-to-PCIe cable. Smaller footprint than a full-height card. Available from multiple PCB vendors.
> *   **MITM passthrough**  --  not endpoint emulators but man-in-the-middle PCIe interposers: a real donor device is passed through and its traffic is observed/modified, giving genuine device behavior by construction because the config space bytes come from actual silicon. Build your own or don't bother  --  anyone selling pre-built interposers on Discord is a scammer.
>
> **Beyond 7-series:**
> *   **Xilinx UltraScale+ (Kintex/Virtex UltraScale+)** with the **Gen3/Gen4** integrated block (PCIe4C/PCIE4CE, PG213) -- moves you from Gen2 (7-series ceiling) to **Gen3 x4/x8 and Gen4 x4/x8**.
> *   **Xilinx Versal ACAP** (Prime/Premium) with the **Gen4/Gen5** DMA & Bridge Subsystem (PG344) -- **Gen5** capability for maximum throughput.
> *   **Wupper** (Nikhef/CERN FELIX) -- a proven open-source **PCIe Gen3/Gen4/Gen5 DMA engine** ported to Virtex-7, Kintex UltraScale, Virtex UltraScale+, and Versal; a strong base if you outgrow the PCILeech 7-series pipeline.

**Important Note on FPGA Families**: While the principles are similar, specific IP core configurations and clocking structures may vary slightly between different Xilinx 7-series FPGAs (Artix-7, Kintex-7, Zynq-7000 PS/PL). Always refer to the specific board's documentation and the Xilinx PCIe IP Core user guides for your chosen FPGA family. The PCILeech-FPGA project often provides board-specific Tcl scripts and source files to simplify this process.

**Supported scope for this guide:**

- **Target hardware:** Xilinx Artix-7 (35T, 75T, 100T), Kintex-7 (K325T, K410T) FPGAs using the 7-series Integrated Block for PCI Express (PG054)
- **PCIe generation:** Gen2 x1/x4. The 7-series hard block does not support Gen3 or higher
- **Toolchain:** Xilinx Vivado 2023.x or later, WebPACK edition (free)
- **Config space:** Conventional 256-byte configuration space (offsets 0x00-0xFF). Extended capabilities (offsets >= 0x100) are not covered by the baseline `.coe` workflow
- **Interface:** AXI4-Stream user interface to the PCIe hard block, as generated by Vivado Tcl scripts
- **Verified host platforms:** 64-bit Windows 10/11, 64-bit Ubuntu 20.04/22.04 LTS
- **What's not covered:** Gen3/Gen4/Gen5 (requires UltraScale+ or Versal), extended configuration space emulation, full device-class behavioral modeling (NIC DMA rings, NVMe command queues, USB transfer descriptors), radio/PHY/media-layer emulation, bitstream encryption, formal verification

> **⚠️ Cheap M.2-to-PCIe adapters:** Those $8 adapter boards from AliExpress will destroy your hardware. They print CE, FCC, and RoHS logos on the silkscreen like it means something  --  it doesn't. Nobody verified those marks. Nobody tested that board. That CE mark was applied by the same person who misspelled "PCI Express" in the listing title. The factory that assembled it doesn't have a compliance department, doesn't own a spectrum analyzer, and has never filed a RoHS declaration with any regulatory body. That CE-FCC-RoHS trinity printed on the solder mask is just a sticker. A prayer. A lie printed in white silkscreen on a PCB that will arc your 12V rail to ground because the trace clearance is whatever the autorouter decided looked fine on a Friday afternoon. There is no polyfuse. There is no ESD protection. There is nobody to sue when your motherboard catches fire. You saved $12 and lost a $200 board. Buy from a vendor that publishes actual schematics and lists real protection components with DigiKey part numbers. If the listing photo is an AI-upscaled render of a green rectangle with no visible component markings, close the tab. Electrical fires are not a debugging tool.

### **3.2 PCIe Hardware Considerations**

To ensure smooth and unrestricted operation of your FPGA-based DMA device for emulation, several PCIe-specific and host system features require careful consideration and, in some cases, modification.

*   **IOMMU / VT-d / AMD-Vi Settings**
    *   **Recommendation**: For initial setup and testing, it is **highly recommended to disable IOMMU (Intel's Virtualization Technology for Directed I/O - VT-d) or AMD's equivalent (AMD-Vi)** in your system's BIOS/UEFI settings.
    *   **Rationale**: IOMMUs are hardware components that provide memory management units for DMA-capable devices. They perform address translation, similar to a CPU's MMU, and can enforce memory access permissions. While crucial for security and virtualization (preventing a rogue device from accessing unauthorized memory regions), they *will* restrict your DMA device's access to system memory, potentially interfering with memory acquisition and device emulation. Disabling the IOMMU allows your DMA device unrestricted access, which is often necessary for advanced emulation and security research purposes.
    *   **Location**: Typically found under "CPU Configuration," "Virtualization," "Advanced Settings," or "I/O Virtualization" in your BIOS/UEFI.
*   **Kernel DMA Protection (Windows) / Thunderbolt Security Level (Linux)**
    *   **Recommendation (Windows)**: Disable **Kernel DMA Protection** features in modern Windows systems. This includes settings like **Virtualization-Based Security (VBS)** and **Memory Integrity (HVCI)**. These features leverage the IOMMU to prevent unauthorized DMA attacks from external peripherals connected via Thunderbolt or PCIe.
    *   **Steps (Windows)**:
        *   Access Windows Security settings: **Start > Settings > Privacy & security > Windows Security > Device security**.
        *   Under "Core isolation," click "Core isolation details."
        *   Turn off "Memory integrity."
        *   You may also need to disable Secure Boot in BIOS/UEFI, as VBS often depends on it.
        *   **Caution**: Disabling these features significantly **reduces your system's security posture**, making it vulnerable to various attacks, including those involving malicious DMA devices. This should only be done on a dedicated test system, not on your primary machine, and in a secure, isolated environment where you understand the risks.
    *   **Recommendation (Linux/Thunderbolt)**: If using a system with Thunderbolt ports, understand and potentially adjust the **Thunderbolt Security Level** in your BIOS/UEFI. Lower security levels (e.g., "No Security," "User Authorization") are generally required for arbitrary Thunderbolt/PCIe devices to perform DMA without explicit host approval.
*   **PCIe Slot Requirements**
    *   **Recommendation**: Use a compatible PCIe slot that physically matches the FPGA device's requirements. Most Artix-7-based DMA cards operate at PCIe Gen2 x1 or x4.
    *   **Rationale**:
        *   **Physical Fit**: An x1 card can fit into x1, x4, x8, or x16 slots, but an x4 card requires at least an x4 slot.
        *   **Performance**: While an x4 card *might* work in an x1 slot (if the physical connector is open-ended or modified), it will operate at the x1 speed, severely limiting data transfer rates. For optimal performance and accurate emulation of a donor device's capabilities, ensure the FPGA board is installed in a slot that provides at least the *emulated* link width and speed (e.g., if you're emulating a Gen2 x4 device, use a Gen2 x4 slot on the host).
    *   **Motherboard BIOS Settings**: Some motherboards allow configuration of PCIe slot speeds (e.g., forcing Gen1 or Gen2). Ensure these settings do not conflict with your desired emulation speed.

### **3.3 System Requirements**

Setting up a robust development environment is key for efficient firmware development, synthesis, and debugging.

*   **Host System**
    *   **Processor**: A modern multi-core CPU is essential for running FPGA development tools like Vivado, which are computationally intensive during synthesis and implementation. (e.g., Intel Core i5/i7/i9 or AMD Ryzen 5/7/9 equivalent, 8th generation or newer recommended).
    *   **Memory (RAM)**: Minimum 16 GB RAM is strongly recommended; **32 GB or more is ideal** for complex FPGA designs, as Vivado can consume significant memory, especially during implementation.
    *   **Storage**: A Solid-State Drive (SSD) with at least **200 GB of free space** is crucial. FPGA tool installations (Vivado alone can be 50+ GB), project files, and synthesis/implementation outputs can quickly consume disk space. The speed of an SSD dramatically reduces build times.
    *   **Operating System**:
        *   **Windows 10/11 (64-bit Professional or Enterprise edition)**: Widely supported by Xilinx Vivado and many hardware debugging tools. Remember the Kernel DMA Protection considerations.
        *   **Compatible Linux Distribution (64-bit)**: Ubuntu LTS (Long Term Support) releases (e.g., 20.04, 22.04) are commonly used and well-supported for Vivado. Linux often provides a more flexible environment for scripting and low-level PCIe interaction tools.
*   **Peripheral Devices**
    *   **JTAG Programmer**: Absolutely necessary for flashing the compiled bitstream onto your FPGA-based DMA card. Examples include Xilinx Platform Cable USB II, Digilent JTAG-HS3, or integrated JTAG programmers found on some development boards. Ensure it's compatible with your FPGA board and Vivado.
    *   **PCIe Slot**: As discussed in Section 3.2, ensure your host system has an available and compatible PCIe slot for your DMA card.
    *   **USB Port**: For connecting the JTAG programmer and potentially for a UART/serial console to your FPGA board for debug output.

---

## **4. Requirements**

This section outlines the essential hardware and software components, along with the recommended environment setup, necessary to embark on custom firmware development for PCIe device emulation. Having these prerequisites in place before you begin will streamline your development process.

### **4.1 Hardware**

*   **Donor PCIe Device**
    *   **Purpose**: This is the physical hardware device whose configuration and behavior you intend to emulate on your FPGA. It serves as the authoritative source for critical identification details, register values, and operational characteristics.
    *   **Examples**: Common examples include a standard Network Interface Card (NIC), a SATA or NVMe storage controller, a USB controller, or any other generic PCIe expansion card that you can safely remove from a system for analysis. It is highly recommended to use a device that is *not* essential for the system's operation, as you will be inspecting its low-level configuration.
*   **DMA FPGA Card**
    *   **Description**: An FPGA-based development board specifically designed or adapted to perform Direct Memory Access (DMA) operations over a PCIe interface. This is the platform onto which your custom firmware will be loaded.
    *   **Examples**: As detailed in Section 3.1, compatible cards include the **Squirrel (Artix-7 35T)**, **Enigma-X1 (Artix-7 75T)**, **ZDMA (Artix-7 100T)**, or various **Kintex-7** based solutions. Ensure your chosen card has a PCIe edge connector.
*   **JTAG Programmer**
    *   **Purpose**: This crucial tool facilitates the communication between your development PC and the FPGA on your DMA card. It's used to program (flash) the compiled bitstream onto the FPGA and, importantly, for interactive debugging using tools like Vivado's Hardware Manager and Integrated Logic Analyzer (ILA).
    *   **Examples**:
        *   **Xilinx Platform Cable USB II**: A traditional, widely compatible programmer for Xilinx FPGAs. Ensure you have the necessary drivers installed.
        *   **Digilent JTAG-HS3 / JTAG-HS2**: Popular and reliable programmers known for good Vivado integration and support. The HS3 offers faster programming speeds.
        *   **Integrated JTAG**: Some FPGA boards may have an onboard USB-to-JTAG bridge (e.g., a FTDI chip) which eliminates the need for a separate programmer. Consult your board's documentation.

### **4.2 Software**

*   **Xilinx Vivado Design Suite**
    *   **Description**: The official, comprehensive FPGA development environment from Xilinx (now AMD). Vivado is essential for synthesizing your HDL code, implementing the design onto the target FPGA, generating the final bitstream, and performing hardware debugging. It includes the necessary IP cores, compilers, and utilities.
    *   **Download**: Visit the official Xilinx (AMD) downloads page: [https://www.xilinx.com/support/download.html](https://www.xilinx.com/support/download.html).
    *   **Version Note**: While older versions like Vivado 2020.1 might be referenced in some legacy guides, it is strongly recommended to download a **recent stable version** (e.g., Vivado 2023.x or later) compatible with your target FPGA family (Artix-7, Kintex-7). The PCILeech-FPGA project generally supports newer Vivado versions.
*   **Visual Studio Code**
    *   **Description**: A highly customizable and feature-rich code editor from Microsoft. It's an excellent choice for writing and editing your Verilog/SystemVerilog HDL code due to its extensive extension ecosystem, offering features like syntax highlighting, linting, auto-completion, and version control integration.
    *   **Download**: [https://code.visualstudio.com/](https://code.visualstudio.com/)
*   **PCILeech-FPGA**
    *   **Description**: An open-source framework and base code repository for FPGA-based DMA development. It provides ready-to-use PCIe IP core instantiations and a well-structured project that serves as an excellent starting point for custom firmware. This guide heavily leverages its architecture.
    *   **Repository**: [https://github.com/ufrisk/pcileech-fpga](https://github.com/ufrisk/pcileech-fpga)
*   **Arbor (MindShare)**
    *   **Description**: A powerful and user-friendly software tool specifically designed for in-depth scanning and analysis of PCIe devices. It provides detailed insights into the configuration space, capabilities, and registers of connected PCIe hardware, making it invaluable for gathering donor device information.
    *   **Download**: Available from the MindShare website: [https://www.mindshare.com/](https://www.mindshare.com/) (You will likely need to navigate to their software section).
    *   **Note**: Typically requires account creation and may offer a time-limited trial.
*   **Alternative PCIe Device Analysis Tools**
    *   **Telescan PE (Teledyne LeCroy)**:
        *   **Description**: A no-cost utility from Teledyne LeCroy for PCIe traffic analysis and device enumeration. While it's primarily a software tool that interfaces with their hardware protocol analyzers, it can also provide some basic configuration space views without dedicated hardware.
        *   **Download**: [https://www.teledynelecroy.com/protocolanalyzer/pci-express/telescan-pe-software/resources/analysis-software](https://www.teledynelecroy.com/protocolanalyzer/pci-express/telescan-pe-software/resources/analysis-software)
        *   **Note**: Requires manual registration and approval for download.
    *   **OS-Native Tools (For basic checks)**:
        *   **Windows Device Manager**: Provides basic Vendor ID, Device ID, Subsystem ID, and Class Code information under the "Details" tab of a device's properties.
        *   **Linux `lspci` utility**: A powerful command-line tool for inspecting PCIe devices. Use `lspci -nn` for Vendor/Device IDs, `lspci -vvv` for verbose details including BARs and capabilities, and `lspci -s <BUS:DEV.FUN> -xxxx` for raw configuration space dumps.

### **4.3 Environment Setup**

#### **4.3.1 Install Vivado Without Losing Your Mind**

Download the latest Vivado that supports 7-series from [xilinx.com](https://www.xilinx.com/support/download.html). During install, **only select "7 Series" devices** unless you enjoy downloading 50 GB of Kintex UltraScale+ support you will never use. Select "Design Tools" and "Programming & Debugging" components. That is it. The installer will take an hour. Go touch grass.

#### **4.3.2 VS Code + Verilog Extension**

Install [VS Code](https://code.visualstudio.com/), then add the "Verilog-HDL/SystemVerilog" extension (mshr-h). Syntax highlighting. Linting. Done.

#### **4.3.3 Clone the Source**

```bash
git clone https://github.com/ufrisk/pcileech-fpga.git
cd pcileech-fpga
```

Navigate into the board-specific subdirectory for your hardware (e.g., `pcileech-squirrel-35t`). The project structure from here is: `src/` holds the SystemVerilog, `ip/` holds `.xci` Xilinx IP cores, and the Vivado `.tcl` scripts in the board root do project generation and building.

#### **4.3.4 Use Dedicated Hardware**

Do not develop DMA firmware on your gaming PC. Do not use a VM with PCIe passthrough for your first build. Use a separate physical machine where you can disable IOMMU, Secure Boot, and any kernel DMA protection without worrying about bricking something you care about. This is not a suggestion.

---

## **5. Gathering Donor Device Information**

Accurate device emulation hinges on meticulously extracting and replicating critical information from the donor device. This comprehensive data collection enables your FPGA to faithfully mimic the target hardware's PCIe configuration and behavior, ensuring compatibility and functionality when interfacing with the host system.

### **5.1 Using Arbor for PCIe Device Scanning**

Arbor (MindShare) is the tool for extracting donor device configuration space. It shows you every register, every capability, every BAR in decoded form. Alternatives: `lspci -vvv` on Linux, Telescan PE (Teledyne LeCroy, free with registration), or Windows Device Manager for basic VID/DID checks.

#### **5.1.1 Install and Scan**

Download Arbor from [mindshare.com](https://www.mindshare.com/) (requires account). Install it. Launch it. Click Local System > Scan. It enumerates every PCIe device on your bus. Find your donor by matching vendor name or VID/DID. Click it to view its full configuration space.

#### **5.1.2 The Data You Actually Need**

Extracting detailed information from the donor device is essential for accurate emulation.

**Information to Extract:**

*   **Device ID (0xXXXX):** A 16-bit identifier unique to the device model.
*   **Vendor ID (0xYYYY):** A 16-bit identifier assigned to the manufacturer.
*   **Subsystem ID (0xZZZZ):** Identifies the specific subsystem or variant (e.g., a specific model within a product line).
*   **Subsystem Vendor ID (0xWWWW):** Identifies the vendor of the subsystem (often the same as the main Vendor ID, but can differ for OEM versions).
*   **Revision ID (0xRR):** Indicates the hardware revision level of the device.
*   **Class Code (0xCCCCCC):** A 24-bit code that defines the primary function/type of device (e.g., `0x020000` for Ethernet Controller, `0x010802` for NVMe controller). This helps the OS load generic drivers.
*   **Base Address Registers (BARs):**
    *   Registers defining the memory or I/O address regions the device uses.
    *   Includes BAR0 through BAR5, each potentially 32 or 64 bits. For each BAR, note its **Type (Memory or I/O)**, **Bit Width (32-bit or 64-bit)**, **Size (e.g., 256 MB, 4KB)**, and **Prefetchable Status (Yes/No)**. This is crucial for memory mapping.
*   **Capabilities:** Lists supported features and their configurations, often found in a linked list structure within the configuration space. Examples include:
    *   **PCIe Capability Structure**: PCIe Link Speed (e.g., Gen2, Gen3), Link Width (e.g., x1, x4), Max Payload Size, Max Read Request Size.
    *   **MSI/MSI-X Capability Structure**: Information on Message Signaled Interrupts, including the number of vectors supported.
    *   **Power Management Capability Structure**: Supported power states (D0, D1, D2, D3hot, D3cold).
*   **Device Serial Number (DSN):** A 64-bit unique identifier, if the device supports it (found in the "Device Serial Number" Extended Capability). Not all devices implement this.

**Steps:**

1.  **Navigate to the PCI Config Tab:**
    *   Within the device's detailed view, find and select the **PCI Config** or **Configuration Space** tab. This typically presents a decoded view of the raw configuration space registers.
2.  **Record Relevant Details:**
    *   Carefully document each of the required fields listed above.
    *   Use screenshots or copy the values into a text file, a dedicated spreadsheet, or a structured documentation format for accuracy.
    *   Ensure hexadecimal values are noted correctly, including the `0x` prefix if used.
3.  **Expand Capability Lists:**
    *   Look for sections labeled **Capabilities** or **Advanced Features**. These are often clickable or expandable to reveal sub-sections.
    *   Document each capability present and its relevant parameters (e.g., MSI message control, power state flags, current/max PCIe link settings).
4.  **Examine BARs in Detail:**
    *   Within the Configuration Space, locate the entries for BAR0 through BAR5.
    *   For each active BAR, note its allocated size, whether it's Memory-mapped or I/O, its bit width (32-bit or 64-bit), and if it's prefetchable. This information is often presented clearly in Arbor's GUI.
5.  **Save the Data for Reference:**
    *   Compile all the extracted information into a well-organized document (e.g., a Markdown file, a `.txt` file, or an Excel spreadsheet).
    *   Label each section clearly for easy reference during firmware customization.



https://github.com/user-attachments/assets/0f192c29-d1ad-4e68-aa21-607976037218



### **5.2 Extracting and Recording Device Attributes**

After capturing the data, it's crucial to understand the significance of each attribute and ensure they've been accurately documented for successful emulation.

**Ensure You Have Accurately Recorded the Following:**

1.  **Device ID:**
    *   **Purpose:** Uniquely identifies the specific model of the PCIe device.
    *   **Usage in Emulation:** Essential for the host Operating System (OS) to correctly identify the emulated device and, crucially, to attempt to load the appropriate device driver.
2.  **Vendor ID:**
    *   **Purpose:** Identifies the manufacturer of the PCIe device.
    *   **Usage in Emulation:** Used in conjunction with the Device ID to form a unique identifier (`VendorID:DeviceID`) that the OS uses to match the device to its corresponding driver.
3.  **Subsystem ID and Subsystem Vendor ID:**
    *   **Purpose:** These optional IDs allow for differentiation between variants of a device from the same vendor, or for OEM-specific versions where the main Vendor/Device ID might be generic.
    *   **Usage in Emulation:** Important for emulating devices with multiple configurations or those supplied by an OEM, as the driver might specifically look for these values.
4.  **Revision ID:**
    *   **Purpose:** Indicates the hardware revision level of the device.
    *   **Usage in Emulation:** Helps in identifying specific hardware versions that might require different drivers, firmware, or possess subtle behavioral differences.
5.  **Class Code:**
    *   **Purpose:** A 24-bit code that categorizes the device's general function (e.g., `0x020000` for an Ethernet controller, `0x010802` for an NVMe controller, `0x0C0300` for a USB host controller). It's composed of a Base Class, Sub-Class, and Programming Interface.
    *   **Usage in Emulation:** Allows the OS to understand the device's general function and load a generic class driver if a specific vendor driver isn't found. This is crucial for initial device recognition.
6.  **Base Address Registers (BARs):**
    *   **Purpose:** Define the memory-mapped or I/O port address regions that the device uses for registers, internal buffers, or configuration space extensions. The host OS allocates physical addresses to these BARs during enumeration.
    *   **Usage in Emulation:** Critical for mapping the emulated device's internal memory and registers into the host system's address space. The size, type (Memory/I/O, 32/64-bit), and prefetchability status of each BAR must be precisely matched to the donor device.
7.  **Capabilities:**
    *   **Purpose:** Lists the advanced features the device supports, such as advanced error reporting, power management, MSI/MSI-X, PCIe Advanced Capabilities (e.g., AER, VC/PF), etc. Each capability is defined by a structure with its own registers.
    *   **Usage in Emulation:** Essential for accurately replicating how the donor device advertises its features and how the host system interacts with these features (e.g., interrupt delivery mechanisms, power state transitions, error reporting).
8.  **Device Serial Number (DSN):**
    *   **Purpose:** A unique 64-bit identifier for the device, typically an optional extended capability.
    *   **Usage in Emulation:** While optional, some drivers or management applications might specifically query and rely on the DSN for identification, licensing, or security checks. Emulating this accurately can prevent detection of your device as a generic or modified peripheral.

**Best Practices for Data Collection:**

*   **Organize the Data:** Create a structured document or spreadsheet. Use clear headings and subheadings for each attribute. A template can be beneficial.
*   **Include Units and Formats:** Always indicate units for sizes (e.g., MB, KB) and use consistent formatting for hexadecimal values (e.g., `0x1234`, `16'h1234`).
*   **Cross-Reference with Specifications (If Possible):** If available, consult the donor device's datasheet or publicly available specifications to verify values. This can help identify any discrepancies or unusual configurations not immediately obvious from a raw scan.
*   **Secure the Data:** Store the collected information securely. Be mindful that this data might contain proprietary or sensitive information.
*   **Understand "What's Missing":** Specialized tools like Arbor are excellent, but they may not capture every single nuance of complex, highly proprietary devices (e.g., specific vendor-defined registers outside standard configuration space). For advanced emulation, you might need to combine this information with reverse engineering of the donor device's drivers.

---

## **6. Initial Firmware Customization**

With the donor device's information meticulously documented, the next critical phase involves customizing your FPGA's firmware to emulate the donor device accurately. This process begins by modifying key identification registers within the PCIe configuration space and ensuring that specific identifiers, like the Device Serial Number, are correctly integrated.

### **6.1 Modifying Configuration Space**

The PCIe Configuration Space is a fundamental component that defines how the device is recognized and interacts with the host system during enumeration. Customizing this space to precisely match the donor device's profile is absolutely essential for successful emulation, allowing the host OS to load the correct driver and interact as expected.

#### **6.1.1 Navigate to the Configuration File**

The PCIe configuration space parameters are typically defined within a specific SystemVerilog (`.sv`) file in the PCILeech-FPGA project. This file synthesizes into the logic that configures the PCIe IP core and exposes the device's identity to the host.

**Common Path for PCILeech-FPGA (Artix-7 based boards like Squirrel):**
Locate the file responsible for configuring the PCIe parameters for your specific board. For many Artix-7 PCILeech variants, this will be:
```
pcileech-fpga/<your_board_variant>/src/pcileech_pcie_cfg_a7.sv
```
*   **Example (for Squirrel 35T)**:
    ```
    pcileech-fpga/pcileech-squirrel-35t/src/pcileech_pcie_cfg_a7.sv
    ```
    *Note: The actual folder name like `pcileech-squirrel-35t` might vary slightly based on the specific PCILeech-FPGA version or fork you cloned. Always navigate to the relevant board-specific subdirectory after cloning the main repository.*

#### **6.1.2 Open the File in Visual Studio Code**

Editing the configuration file requires a suitable code editor that supports syntax highlighting for SystemVerilog (or Verilog), making the code easier to read and modify.

**Steps:**

1.  **Launch Visual Studio Code:**
    *   Click on the VS Code icon or find it via the Start Menu.
2.  **Open the File:**
    *   Use **File > Open File** or press `Ctrl + O` (or `Cmd + O` on macOS).
    *   Navigate to the configuration file path identified in Section 6.1.1 (e.g., `pcileech-fpga/pcileech-squirrel-35t/src/pcileech_pcie_cfg_a7.sv`).
    *   Select the file and click **Open**.
3.  **Verify Syntax Highlighting:**
    *   Ensure that the editor recognizes the `.sv` file extension and applies proper SystemVerilog syntax highlighting. If it doesn't, revisit Section 4.3.2 to ensure you've installed the recommended Verilog/SystemVerilog extensions for VS Code.
4.  **Familiarize Yourself with the File Structure:**
    *   Scroll through the file. You will typically find parameters defined using `localparam` or `reg` assignments, often with comments explaining their purpose. Look for sections where standard PCIe configuration registers (Vendor ID, Device ID, etc.) are defined and assigned.

#### **6.1.3 Modify Device ID and Vendor ID**

Updating these fundamental identifiers is the most crucial step for the host system to correctly recognize the emulated device as your donor. The Operating System relies heavily on the `Vendor ID` and `Device ID` pair to identify connected hardware and load the appropriate device driver.

**Steps:**

1.  **Search for `cfg_deviceid`:**
    *   Use the search functionality (`Ctrl + F` or `Cmd + F`) within VS Code.
    *   Locate the line defining `cfg_deviceid`. It will typically look something like:
        ```verilog
        reg [15:0] cfg_deviceid = 16'hAAAA; // Default or placeholder Device ID
        ```
2.  **Update Device ID:**
    *   Replace `AAAA` with the 16-bit hexadecimal Device ID you extracted from the donor device using Arbor (e.g., `0x1234`).
    *   **Example:**
        If the donor's Device ID is `0x1234`, update the line as:
        ```verilog
        reg [15:0] cfg_deviceid = 16'h1234; // Updated with donor's Device ID (e.g., from network card)
        ```
3.  **Search for `cfg_vendorid`:**
    *   Locate the line defining `cfg_vendorid`. It will be similar in format to `cfg_deviceid`:
        ```verilog
        reg [15:0] cfg_vendorid = 16'hBBBB; // Default or placeholder Vendor ID
        ```
4.  **Update Vendor ID:**
    *   Replace `BBBB` with the 16-bit hexadecimal Vendor ID extracted from the donor device (e.g., `0xABCD`).
    *   **Example:**
        If the donor's Vendor ID is `0xABCD`, update the line as:
        ```verilog
        reg [15:0] cfg_vendorid = 16'hABCD; // Updated with donor's Vendor ID (e.g., Intel Corporation)
        ```
5.  **Ensure Correct Formatting:**
    *   Verify that hexadecimal values are correctly prefixed with `16'h` (indicating a 16-bit hexadecimal number).
    *   Maintain consistent indentation and commenting style for readability.

#### **6.1.4 Modify Subsystem ID and Revision ID**

These identifiers provide additional details about the device variant, specific product models, or hardware revisions. While often optional, matching them enhances the authenticity of the emulation and can be critical for drivers that perform granular checks.

**Steps:**

1.  **Search for `cfg_subsysid`:**
    *   Locate the line defining `cfg_subsysid`.
    ```verilog
    reg [15:0] cfg_subsysid = 16'hCCCC; // Placeholder Subsystem ID
    ```
2.  **Update Subsystem ID:**
    *   Replace `CCCC` with the 16-bit hexadecimal Subsystem ID from your donor device (e.g., `0x5678`).
    *   **Example:**
        ```verilog
        reg [15:0] cfg_subsysid = 16'h5678; // Set to donor's Subsystem ID
        ```
3.  **Search for `cfg_subsysvendorid`:**
    *   Locate the line defining `cfg_subsysvendorid`.
    ```verilog
    reg [15:0] cfg_subsysvendorid = 16'hDDDD; // Placeholder Subsystem Vendor ID
    ```
4.  **Update Subsystem Vendor ID (if applicable):**
    *   Replace `DDDD` with the 16-bit hexadecimal Subsystem Vendor ID from your donor device (e.g., `0x9ABC`). If your donor device does not have a unique Subsystem Vendor ID (i.e., it's the same as the main Vendor ID), you should still set it to that value.
    *   **Example:**
        ```verilog
        reg [15:0] cfg_subsysvendorid = 16'h9ABC; // Set to donor's Subsystem Vendor ID
        ```
5.  **Search for `cfg_revisionid`:**
    *   Locate the line defining `cfg_revisionid`.
    ```verilog
    reg [7:0] cfg_revisionid = 8'hEE; // Placeholder Revision ID
    ```
6.  **Update Revision ID:**
    *   Replace `EE` with the 8-bit hexadecimal Revision ID from your donor device (e.g., `0x01`).
    *   **Example:**
        ```verilog
        reg [7:0] cfg_revisionid = 8'h01; // Set to donor's Revision ID
        ```

#### **6.1.5 Update Class Code**

The Class Code informs the host Operating System about the device's general type and function (e.g., network controller, storage device). This is vital for the OS to load generic class drivers, even if a specific vendor driver isn't installed.

**Steps:**

1.  **Search for `cfg_classcode`:**
    *   Locate the line defining `cfg_classcode`.
    ```verilog
    reg [23:0] cfg_classcode = 24'hFFFFFF; // Default or placeholder Class Code
    ```
2.  **Update Class Code:**
    *   Replace `FFFFFF` with the 24-bit hexadecimal Class Code you extracted from the donor device (e.g., `0x020000` for an Ethernet Controller). Remember the format: Base Class, Sub-Class, Programming Interface.
    *   **Example:**
        If the donor's Class Code is `0x020000` (meaning Base Class: 0x02 - Network Controller, Sub-Class: 0x00 - Ethernet Controller, Prog IF: 0x00), update as:
        ```verilog
        reg [23:0] cfg_classcode = 24'h020000; // Set to donor's Class Code (e.g., Ethernet Controller)
        ```
3.  **Verify Correct Bit Width:**
    *   Ensure that the Class Code is correctly represented as a 24-bit value using the `24'h` prefix for hexadecimal.

#### **6.1.6 Save Changes**

After making all modifications to the configuration parameters, it's critically important to save and review the changes.

**Steps:**

1.  **Save the File:**
    *   Click **File > Save** in VS Code, or press `Ctrl + S` (or `Cmd + S`).
2.  **Review Changes:**
    *   Before closing, quickly re-read the modified lines to confirm their accuracy against your documented donor device information.
    *   Check for any obvious syntax errors or typos (VS Code's extensions may highlight these).
3.  **Optional - Use Version Control:**
    *   If you are using Git (highly recommended for any code project, especially firmware), commit your changes with a clear and meaningful message. This creates a historical record of your modifications.
    *   **Example Git Commands:**
        ```bash
        git add pcileech_pcie_cfg_a7.sv
        git commit -m "Updated PCIe configuration registers (VID, DID, SubIDs, Revision, Class Code) to match donor device: [Donor Device Name]"
        ```

### **6.2 Inserting the Device Serial Number (DSN)**

The Device Serial Number (DSN) is a unique 64-bit identifier that some PCIe devices (especially those with advanced features or specific drivers) may utilize. Including it enhances the authenticity of your emulation and can help bypass checks in drivers that explicitly query for this value.

#### **6.2.1 Locate the DSN Field**

The DSN, if implemented by the donor device, is part of the PCIe Extended Capabilities. In the PCILeech-FPGA framework, the DSN field is often exposed as a configurable parameter within the same configuration file you've been editing.

**Steps:**

1.  **Search for `cfg_dsn`:**
    *   In `pcileech_pcie_cfg_a7.sv` (or your board's equivalent configuration file), use the search function (`Ctrl + F` or `Cmd + F`) to find `cfg_dsn`.
2.  **Understand the Existing Assignment:**
    *   The DSN may be set to a default value (often all zeros) or commented out. It will typically look like:
        ```verilog
        reg [63:0] cfg_dsn = 64'h0000000000000000; // Default DSN (usually 0 if not used)
        ```

#### **6.2.2 Insert the DSN**

Updating the DSN involves setting it to the exact 64-bit hexadecimal value captured from your donor device.

**Steps:**

1.  **Update `cfg_dsn`:**
    *   Replace the existing hexadecimal value with the 64-bit DSN you extracted from the donor device using Arbor.
    *   **Example:**
        If the donor's DSN is `0x0011223344556677`, update as:
        ```verilog
        reg [63:0] cfg_dsn = 64'h0011223344556677; // Donor Device Serial Number
        ```
2.  **Handle DSN Unavailability or Irrelevance:**
    *   If your donor device does *not* have a DSN, or if you've determined it's not a required parameter for the driver you're targeting, you can simply leave it as zeros:
        ```verilog
        reg [63:0] cfg_dsn = 64'h0000000000000000; // No specific DSN from donor, leaving as default 0
        ```
    *   **Caution**: For critical emulations, if the donor device has a DSN, it's best to emulate it accurately.
3.  **Ensure Correct Formatting:**
    *   The DSN is a 64-bit value; ensure it's properly formatted with the `64'h` prefix for hexadecimal values.

#### **6.2.3 Save Changes**

Finalize the DSN modification by saving and reviewing the file.

**Steps:**

1.  **Save the File:**
    *   Click **File > Save** in VS Code, or press `Ctrl + S`.
2.  **Verify the Syntax:**
    *   Check for any red underlines or error indications from VS Code's syntax checker. Correct any issues immediately.
3.  **Document the Changes:**
    *   If using version control, commit the updates with an appropriate message.
    *   **Example Git Commands:**
        ```bash
        git commit -am "Inserted donor Device Serial Number (DSN) into PCIe configuration"
        ```

---

## **7. Vivado Project Setup and Customization**

With the firmware files updated to reflect the donor device's critical identification and configuration data, the next crucial step is to integrate these changes into the Vivado project. This involves generating the project files for your specific FPGA board, customizing the embedded PCIe IP core, and preparing the entire design for the synthesis and implementation phases.

### **7.1 Generating Vivado Project Files**

Vivado uses Tcl scripts to automate project creation. The PCILeech-FPGA repo ships one per board. Run it. Don't click around the GUI trying to add source files manually. You will miss an IP core dependency and spend three hours debugging a black-box netlist that never got generated.

Open Vivado. Open the Tcl Console (`Window > Tcl Console`). If you need directions to find the Tcl Console, stop here and spend twenty minutes becoming friends with Vivado before it becomes your landlord.

```tcl
cd C:/path/to/pcileech-fpga/pcileech-squirrel-35t
source vivado_generate_project_squirrel.tcl -notrace
```

Replace `squirrel` with your board variant. The `-notrace` flag suppresses the firehose of Tcl debug output. The script creates the `.xpr` project file, adds all SystemVerilog sources, adds Xilinx IP core configs (`.xci`), adds XDC constraints, and configures project settings. If it fails with "file not found," your checkout is wrong  --  do not begin randomly editing HDL to compensate for a missing generated IP core.

After the script finishes, open the generated `.xpr` in Vivado (`File > Open Project`). Expand the Sources pane. Verify all expected files are present. Check the Messages pane for critical warnings. You should see `pcie_7x_0.xci` under the IP section  --  that is the PCIe hard block you will customize next.

### **7.2 Modifying IP Blocks**

The PCIe IP core is the heart of your device's PCIe interface. It's a pre-verified, configurable block from Xilinx that handles the complex PCIe protocol layers. While some configuration space values are handled in the SystemVerilog file (Section 6.1), other core PCIe parameters, especially those related to link capabilities and BAR structures, are configured directly within the PCIe IP core's customization settings in Vivado. Customizing the IP core ensures that your FPGA behaves identically to the donor hardware at the PCIe protocol level.

#### **7.2.1 Access the PCIe IP Core**

The PCIe IP core is instantiated as an IP block within your Vivado project. You'll need to open its customization GUI to modify its parameters.

**Steps:**

1.  **Locate the PCIe IP Core:**
    *   In the **Sources** pane (within the **Project Manager** window), ensure the **Hierarchy** tab is selected.
    *   Expand the design hierarchy until you find the instance of the PCIe IP core.
    *   It is commonly named `pcie_7x_0.xci` (for 7-Series FPGAs) or similar, typically found within the `ip` subdirectory of your project sources.
2.  **Open the IP Customization Window:**
    *   **Right-click** on the `pcie_7x_0.xci` file.
    *   From the context menu, select **Customize IP**.
    *   The **IP Configuration** window (or similar name, like "Customize IP" or "Re-customize IP") will open, presenting a graphical interface with various tabs and options for configuring the PCIe core.
3.  **Wait for IP Settings to Load:**
    *   The IP customization interface may take a few moments to initialize and populate all its settings. Ensure that all options and tabs are fully loaded and responsive before you begin making changes.

#### **7.2.2 Config-space ownership: who owns what**

Not every config-space field is set by editing a SystemVerilog register. The Xilinx 7-series PCIe hard block owns some fields at IP generation time, some are status outputs, and some are gated through the user-logic config-management interface. Guessing where VID/DID comes from is how you spend three hours changing a `.sv` file that the core ignores.

| Field group | Owner | Where to set it | Verification |
|---|---|---|---|
| VID/DID, subsystem IDs, revision, class code | PCIe IP core `.xci` or config-management HDL (depends on fork) | IP GUI (IDs tab) OR `pcileech_pcie_cfg_a7.sv`  --  find which path your fork uses | Host reads offsets `0x00`-`0x0B` after enumeration |
| BAR type, size, prefetchable | PCIe IP core `.xci` | IP GUI (BARs tab) | All-ones BAR sizing probe after enumeration |
| Command/Status register | IP core + user logic status inputs | IP GUI + config-management path | Write patterns to validate RW/RO/RW1C behavior |
| Capability pointer (offset `0x34`) | Depends on capability chain implementation | IP GUI for core-generated capabilities; user logic for custom | Capability chain traversal from offset `0x34` |
| Conventional capabilities (PM, MSI, PCIe) | Core-generated or user-defined per capability | IP GUI enables the structure; user logic handles register behavior | Read/write/functional tests per capability |
| Extended capabilities (DSN, AER, etc.) | Explicit user implementation required | Custom HDL + full 4 KB config-space BRAM | Accesses at offsets >= `0x100` |
| MMIO BAR behavior | User logic (`bar_controller`) | `pcileech_tlps128_bar_controller.sv` | Read/write/byte-enable regression at BAR addresses |

Do not hand-edit both the IP GUI and the `.sv` file for the same field and hope they agree. Find the actual source of truth for your fork. Verify with host-side reads. A value that appears correct in simulation but wrong on hardware means your config path is lying to you.

#### **7.2.3 Customize Device IDs and BARs within the IP Core**

While some device identifiers are set in `pcileech_pcie_cfg_a7.sv`, the PCIe IP core itself also contains internal parameters for Device ID, Vendor ID, and crucially, the definitions for the Base Address Registers (BARs). You must ensure these align. Some values set in the `.sv` file might override or be fed into the IP core, but it's good practice to ensure consistency here too. The BAR settings in the IP core are particularly important as they dictate the hardware implementation of memory mapping.

**Steps:**

1.  **Navigate to Basic/Identification Parameters:**
    *   In the IP customization window, look for a tab or section related to **Basic**, **Device and Vendor Identifiers**, **General**, or **PCIe Capabilities**. This is where fundamental IDs and initial link settings are defined.
2.  **Verify/Enter the Device ID, Vendor ID, Subsystem IDs, Revision ID, Class Code:**
    *   **Crucially, confirm these match the values you put in `pcileech_pcie_cfg_a7.sv` and from your donor device.**
    *   Find fields like:
        *   **Device ID**: Enter `0xXXXX` (e.g., `0x1234`).
        *   **Vendor ID**: Enter `0xYYYY` (e.g., `0xABCD`).
        *   **Subsystem ID**: Enter `0xZZZZ` (e.g., `0x5678`).
        *   **Subsystem Vendor ID**: Enter `0xWWWW` (e.g., `0x9ABC`).
        *   **Revision ID**: Enter `0xRR` (e.g., `0x01`).
        *   **Class Code**: Enter `0xCCCCCC` (e.g., `0x020000`).
    *   The Xilinx PCIe IP core owns VID/DID/SSID/SSVID/Revision/Class Code in the configuration space it generates. Identify which specific parameters in the `.xci` or its generated wrapper feed those fields. Some forks also route these through the user-logic config-management path (`pcileech_pcie_cfg_a7.sv`). Find the actual source of truth in your fork and set it there -- do not guess. Verify by reading config offsets `00h` through `0Bh` from the host after enumeration.
3.  **Navigate to Base Address Registers (BARs) Tab:**
    *   Within the IP customization window, locate and select the **BARs** tab or section. This is where you define the memory regions the PCIe device exposes.
4.  **Configure Each BAR:**
    *   For each BAR (BAR0 through BAR5) that your donor device uses, meticulously configure the following parameters based on the information extracted using Arbor:
        *   **Enable BAR**: Check this box only if the donor device uses this specific BAR. Disable (uncheck) any BARs the donor device does not use.
        *   **BAR Size**: Select the exact size from the dropdown list (e.g., **256 MB**, **64 KB**, **4 KB**). This is critical for the host OS to allocate the correct amount of memory.
        *   **BAR Type**: Choose the appropriate type:
            *   **Memory (32-bit Addressing)**: For memory-mapped regions accessible with 32-bit addresses.
            *   **Memory (64-bit Addressing)**: For memory-mapped regions that can reside anywhere in the 64-bit address space (required for large memory regions or if the donor device uses it).
            *   **I/O**: For legacy I/O port regions (less common in modern PCIe but still possible).
        *   **Prefetchable**: Check this box if the donor's BAR is marked as prefetchable. This property allows the host system to cache or prefetch data from this region for performance.
    *   **Example Configuration (based on your donor device):**
        *   **BAR0**:
            *   Enabled: Yes
            *   Size: **256 MB**
            *   Type: **Memory (64-bit Addressing)**
            *   Prefetchable: Yes
        *   **BAR1**:
            *   Enabled: No (if the donor device does not use BAR1)
        *   *Continue for BAR2-BAR5, mirroring the donor device's configuration.*
5.  **Ensure Alignment and Non-Overlapping Spaces**:
    *   The Vivado IP core typically handles the alignment automatically based on the size you select. However, be aware that PCIe specification requires BAR sizes to be a power of two, and BARs must be aligned to their size.
    *   Ensure that every implemented BAR range has intentional access behavior. A BAR's advertised aperture does not need to be fully backed by BRAM -- sparse decode and defined fallback behavior for unimplemented regions are valid. Confirm that any region requiring real backing storage fits your selected BRAM, external-memory, or windowing design.

#### **7.2.3 Finalize IP Customization**

After configuring all necessary settings within the IP core's customization window, you must apply these changes for them to take effect in your Vivado project.

**Steps:**

1.  **Review All Settings:**
    *   Before applying, take a moment to quickly review each tab in the IP customization window one last time.
    *   Confirm that all entries precisely match the documented specifications of your donor device. A small error here can lead to device detection or functionality issues.
2.  **Apply Changes:**
    *   Click the **OK** or **Generate** button (the label may vary) at the bottom of the IP customization window.
    *   If Vivado prompts you to confirm that you wish to proceed with the changes and regenerate the IP output products, confirm by clicking **Yes**.
3.  **Regenerate IP Core:**
    *   Vivado will now regenerate the IP core's output products (e.g., netlists, simulation models, new `.xci` configuration files) to reflect your new configurations.
    *   Monitor the **Messages** pane (bottom of Vivado window) for any errors, warnings, or critical warnings that may arise during this regeneration process. Address any critical warnings immediately.
4.  **Update IP in Project:**
    *   Vivado might automatically update or prompt you to update any IP dependencies in your project after a core is regenerated. Allow it to do so to ensure the latest configuration is used throughout your design.

#### **7.2.4 Generated IP Is Source, Not Scenery**

Commit the `.xci` file, board Tcl, XDC, and a manifest containing the Vivado version, FPGA part, and generated-IP checksum. Regenerate output products from Tcl in a clean build directory. If an IP setting changed, the `.xci` diff and regeneration log should prove it -- not a GUI checkbox you clicked three hours ago and forgot about.

Do not hand-edit generated HDL. Do not rely on a GUI-only change that cannot be reproduced from a clean checkout. The build must be deterministic from your version-controlled sources.

---

## **Part 2: Your Device's Passport, and Why a Forged One Is Not a Personality**

---

## **8. Advanced Firmware Customization**

To achieve a precise and convincing emulation of the donor device, further in-depth customization of the FPGA firmware is necessary beyond just basic identification. This involves aligning the PCIe parameters (such as link speed and transaction sizes), meticulously adjusting Base Address Registers (BARs) and their associated memory mappings, and accurately emulating power management and interrupt mechanisms. These steps ensure that the emulated device not only looks like the original hardware to the host system but also behaves identically at the protocol and functional levels.

### **8.1 Configuring PCIe Parameters for Emulation**

Accurate emulation requires that the PCIe operational parameters of your FPGA device are meticulously configured to match those of the donor device. This includes settings such as the PCIe link speed, link width, capability pointers, and maximum payload sizes. Proper configuration ensures compatibility with the host system, the correct operation of drivers and applications that interact with the device, and optimal performance for data transfer.

#### **8.1.1 Matching PCIe Link Speed and Width**

The PCIe link speed (e.g., Gen1, Gen2, Gen3) and link width (e.g., x1, x4, x8) are critical parameters that determine the maximum theoretical data throughput and performance of the device. Matching these settings with the donor device is essential for accurate emulation, as drivers or system components might expect specific link capabilities.

**Steps:**

1.  **Access PCIe IP Core Settings:**
    *   **Open Your Vivado Project:** Launch Vivado and open the project you previously created or modified (e.g., `pcileech_squirrel_top.xpr`). Ensure that all source files are correctly added to the project.
    *   **Locate the PCIe IP Core:** In the **Sources** pane (typically on the left), expand the design hierarchy to find the PCIe IP core instance. For Xilinx 7-series designs (like Artix-7 used in Squirrel), this is usually named `pcie_7x_0.xci`.
    *   **Customize the IP Core:** Right-click on `pcie_7x_0.xci` and select **Customize IP**. The IP customization window will open, displaying various configuration options across multiple tabs.

2.  **Set Maximum Link Speed:**
    *   **Navigate to Link Parameters:** In the IP customization window, click on the **PCIe Capabilities** tab (or sometimes "PCIe Configuration" or "General"). Within this, look for a section related to **Link Parameters** or **Link Capability Register**.
    *   **Configure Maximum Link Speed:** Find the option labeled **Maximum Link Speed** (or "Target Link Speed").
    *   Set it to match the maximum link speed supported and advertised by your donor device.
        *   **Example:**
            *   If the donor device operates at **PCIe Gen2 (5.0 GT/s)**, select **5.0 GT/s**.
            *   If it operates at **PCIe Gen1 (2.5 GT/s)** or **PCIe Gen3 (8.0 GT/s)**, select the corresponding option.
    *   **Note**: Ensure that your FPGA's transceivers and the physical hardware (motherboard PCIe slot) support the selected link speed. The FPGA will only negotiate up to its configured maximum.

3.  **Set Link Width:**
    *   **Configure Link Width:** In the same **Link Parameters** section, locate the **Link Width** setting (or "PCIe Link Width," "Target Link Width").
    *   Set it to match the maximum link width advertised by your donor device.
        *   **Example:**
            *   If the donor device uses a **x4** link, set the **Link Width** to **4**.
            *   Options typically include **1**, **2**, **4**, **8**, **16** lanes.
    *   **Note**: The physical PCIe slot and the FPGA's package must support the selected link width. Attempting to configure a width greater than the physical connection will result in link negotiation issues.

4.  **Save and Regenerate:**
    *   **Apply Changes:** After configuring the link speed and width, click **OK** to apply the changes within the IP customization window.
    *   **Regenerate IP Output Products:** Vivado will likely prompt you to regenerate the IP core's output products due to the changes made. Confirm and allow the regeneration process to complete. This can take some time.
    *   **Verify Settings:** Once regeneration is complete, you can optionally revisit the IP core settings to ensure the configurations are correctly applied. Check for any warnings or errors in the **Messages** window in Vivado.

#### **8.1.2 Setting Capability Pointers**

Capability pointers within the PCIe configuration space are 8-bit registers that form a linked list, pointing to various capability structures (e.g., Power Management, MSI/MSI-X, PCIe Express Capability). Correctly setting these pointers ensures that the host system can traverse the capability list and locate and utilize the device's advertised features.

**Steps:**

1.  **Locate Capability Pointer in Firmware:**
    *   **Open Configuration File:** In Visual Studio Code, open the primary configuration file for your board, typically `pcileech_pcie_cfg_a7.sv`, located at `pcileech-fpga/<your_board_variant>/src/pcileech_pcie_cfg_a7.sv`.
    *   **Understand the Capability Pointer:** The capability pointer (`cfg_cap_pointer`) in this file points to the *first* capability structure in the PCIe configuration space, usually starting after the standard 64-byte configuration header. Subsequent capabilities are chained by their "Next Capability Pointer" fields.

2.  **Set Capability Pointer Value:**
    *   **Find the Assignment for `cfg_cap_pointer`:** Search for the line in the code where `cfg_cap_pointer` is assigned.
        ```verilog
        reg [7:0] cfg_cap_pointer = 8'hXX; // Current value (e.g., 8'h40 for default)
        ```
    *   **Update the Capability Pointer:** Replace `XX` with the 8-bit hexadecimal capability pointer value observed from your donor device using Arbor. This value typically points to the offset of the first capability structure after the Device-Specific Configuration Space (which usually ends at offset `0x3F`). A common starting point for capabilities is `0x40` or `0x60`.
        *   **Example:**
            *   If the donor device's first capability pointer is `0x60` (meaning its first capability structure starts at offset `0x60` in the configuration space), update the line to:
                ```verilog
                reg [7:0] cfg_cap_pointer = 8'h60; // Updated to match donor device's first capability offset
                ```
    *   **Ensure Correct Alignment:** Capability structures must be aligned on a 4-byte boundary. The capability pointer should always point to a valid 4-byte aligned offset within the configuration space.

3.  **Save Changes:**
    *   **Save the Configuration File:** After making the changes, save the file by clicking **File > Save** or pressing `Ctrl + S`.
    *   **Verify Syntax:** Ensure there are no syntax errors introduced by the changes (VS Code will usually highlight these).
    *   **Comment for Clarity:** Add a comment explaining the change for future reference and maintainability.
        ```verilog
        reg [7:0] cfg_cap_pointer = 8'h60; // Set to donor's capability pointer (e.g., PCIe Cap at 0x60)
        ```

#### **8.1.3 Adjusting Maximum Payload and Read Request Sizes**

These parameters define the maximum amount of data that can be transferred in a single PCIe Transaction Layer Packet (TLP) and the maximum size of a non-posted Memory Read Request TLP. Matching these settings with the donor device ensures compatibility and optimal performance for data transfer operations. Mismatches can lead to reduced throughput or communication errors.

**Steps:**

1.  **Set Max Payload Size Supported (Device Capabilities):**
    *   In the PCIe IP core customization window (`pcie_7x_0.xci`), navigate to **PCIe Capabilities**.
    *   **Max Payload Size Supported** is a read-only capability field in the PCI Express Device Capabilities register (bits 2:0). Set it to the value your endpoint can handle (128, 256, 512, 1024, 2048, or 4096 bytes).
    *   The actual active **Max Payload Size** is in Device Control (bits 7:5) and is negotiated by software. Your datapath must respect the active value, not just the capability field.
    *   **Max Read Request Size** (MRRS) lives in Device Control (bits 14:12) and is set by software. There is no separate "Max Read Request Size Supported" capability field in Device Capabilities  --  MRRS is a control field, not a capability advertisement. Your request path must obey whatever value software programs into this field.
                ```verilog
                reg [2:0] max_read_request_size_supported = 3'b010; // Supports up to 512 bytes (0x200)
                ```
    *   **Reasoning**: These firmware parameters often dictate the behavior of your user logic that interfaces with the PCIe core, ensuring that your logic respects the configured maximums.

4.  **Save Changes:**
    *   **Save the File:** After updating the values in `pcileech_pcie_cfg_a7.sv`, save the file.
    *   **Verify Consistency:** It's crucial that the values configured in the Vivado PCIe IP core's GUI *match* the values set in your HDL configuration file. Any mismatch can lead to unexpected behavior or link training issues.
    *   **Add Comments:** Document these changes clearly in your code for future reference.

### **8.2 BARs: The Host Rents You an Address, Not a Warehouse**

A 256 MiB BAR does not mean you need 256 MiB of BRAM. If it did, every Artix-7 35T owner would be outside Xilinx headquarters with a brick. A 35T has about 225 KiB of BRAM total. A 256 MiB BAR would require over 1000 times that. Don't do it.

A BAR is the aperture you advertise to the host. It says: "map this much address space for me." It does NOT say: "I have physically built a RAM chip behind every byte of it." Your logic can implement a register file, a sparse decoder, a window into external RAM, or a deliberately defined unsupported region that returns 0xFFFFFFFF on reads and ignores writes. All of these are valid.

The host will probe the BAR. The PCIe core must answer that probe correctly: returning the BAR size when the host writes all-ones to the register and reads it back is a hardware-level negotiation, not something you implement in user logic -- the Xilinx PCIe IP core handles this for you when you configure the BAR size in the IP customization GUI. After enumeration, every MMIO address within that advertised aperture needs intentional behavior from your firmware. "Nothing happens" is behavior, but make it deterministic. A giant BAR with a 4 KiB register block hiding in the corner is fine. A giant BAR with random garbage, broken completions, and no plan for address decoding is how you become an Unknown Device with Code 43.

**Do this:**
- Match aperture size, 32/64-bit form, and prefetchable state to the donor profile in the IP core GUI
- A 64-bit BAR consumes two BAR slots (BAR0+BAR1, BAR2+BAR3, etc.)
- Decode only the address ranges you actually implement in your `bar_controller`
- Define behavior outside implemented ranges: return completions with UR (Unsupported Request) or CA (Completer Abort), return zeros, or ignore writes. Be consistent.
- Do not instantiate a BRAM IP core sized to match a 256 MiB BAR on a 35T. You will run out of BRAM, Vivado will fail implementation, and you will be sad.

Base Address Registers (BARs) are how a PCIe device exposes its internal memory and registers to the host system. Correctly configuring the BARs and defining their memory mapping is crucial for the proper operation of host-side device drivers.

#### **8.2.1 Setting BAR Sizes and Types (IP Core & BRAMs)**

Configuring the BAR sizes and types ensures that your emulated device requests the correct amount of address space from the host during enumeration and that the host allocates and maps these regions appropriately. This also requires defining deterministic endpoint behavior for accesses within each advertised aperture. That behavior may be register-backed, sparse-decoded, windowed into external memory, or deliberately unsupported. The requirement is not that every address be backed by local memory, but that access behavior be intentional and compatible with the selected device profile.

**Steps:**

1.  **Access BAR Configuration (PCIe IP Core):**
    *   **Customize PCIe IP Core:** In Vivado, right-click on `pcie_7x_0.xci` and select **Customize IP** to open its configuration GUI.
    *   **Navigate to BARs Tab:** In the IP customization window, click on the **Base Address Registers (BARs)** tab.

2.  **Configure Each BAR (IP Core):**
    *   **Match Donor Device's BARs:** For each BAR (BAR0 to BAR5), set the size, type, and prefetchable status to precisely match what you extracted from the donor device using Arbor.
    *   **Enable/Disable BARs:** Ensure that only the BARs actually used by the donor device are enabled. Disable (uncheck) any unused BARs.
    *   **Set BAR Sizes:** Select the appropriate size from the dropdown for each *enabled* BAR. This will be a power-of-two size (e.g., 4KB, 8KB, 64KB, 1MB, 256MB, 1GB).
        *   **Example:**
            *   If **BAR0** is **64 KB**, set **BAR0 Size** to **64 KB**.
            *   If **BAR1** is **128 MB**, set **BAR1 Size** to **128 MB**.
    *   **Set BAR Types:**
        *   Choose between **Memory (32-bit Addressing)** or **Memory (64-bit Addressing)** if the BAR is memory-mapped. Select **64-bit Addressing** if the donor device's BAR is 64-bit or if you require access to addresses above 4GB.
        *   Choose **I/O** if the BAR is for I/O port space (less common for modern PCIe devices).
    *   **Set Prefetchable Status**: Check the "Prefetchable" box if the donor's BAR was identified as prefetchable. This bit allows the host to prefetch data from that region, potentially improving performance.

3.  **Update BRAM Configurations (if applicable):**
    *   Many PCILeech-FPGA projects use Xilinx Block RAM (BRAM) IP cores to represent the memory regions exposed by the BARs. These BRAMs provide the physical storage for your emulated device's memory.
    *   **Locate BRAM IP Cores:** In your Vivado project's **Sources** pane, within the `ip` subdirectory (or similar), you might find `.xci` files for BRAMs, potentially named like:
        ```
        pcileech-fpga/<your_board_variant>/ip/bram_bar_zero4k.xci
        pcileech-fpga/<your_board_variant>/ip/bram_pcie_cfgspace.xci
        # And potentially others for BAR1, BAR2 etc.
        ```
    *   **BRAM sizing reality check:** The BRAM only needs to be large enough to back the portions of the BAR aperture you actually implement -- NOT the full advertised BAR size. If your donor advertises a 256 MB BAR but only the first 4 KB is actually used for MMIO registers, your BRAM only needs to be 4 KB. Use sparse decoding in your `bar_controller` to handle accesses outside the implemented range gracefully. A 35T has ~225 KiB of total BRAM. A 100T has ~607 KiB. You cannot physically back a 256 MB BAR with on-chip BRAM on any Artix-7 device. If you need large buffer storage, use external DDR memory (if your board has it) or implement a windowing scheme. A small register map inside a larger advertised aperture is correct and common  --  the requirement is that every accessed address has intentional behavior, not that every address be backed by physical memory.

4.  **Save and Regenerate:**
    *   **Apply Changes (IP Core):** After configuring the BARs in the PCIe IP core, click **OK** in the IP customization window.
    *   **Regenerate IP Cores:** Vivado will prompt you to regenerate the PCIe IP core and any associated BRAM IP cores due to the size changes. Allow the regeneration to complete. This ensures the hardware netlist reflects your new BAR definitions.
    *   **Check for Errors:** Review the **Messages** window for any warnings or errors related to BAR configurations or BRAM instantiation.

#### **8.2.2 Defining BAR Address Spaces in Firmware**

While the PCIe IP core configures the *hardware* aspects of the BARs, your custom firmware (SystemVerilog code) needs to define the *logic* for how your emulated device responds when the host CPU performs read or write operations to addresses within these BAR regions. This involves address decoding and implementing register/memory access logic.

**Steps:**

1.  **Open the BAR Controller File:**
    *   In Visual Studio Code, open the SystemVerilog file responsible for handling BAR accesses. For PCILeech-FPGA, this is often:
        ```
        pcileech-fpga/<your_board_variant>/src/pcileech_tlps128_bar_controller.sv
        ```
        This module typically receives PCIe Memory Read/Write TLPs and decodes the address to determine which BAR (and which offset within that BAR) is being accessed.

2.  **Implement Address Decoding Logic:**
    *   Within the `pcileech_tlps128_bar_controller.sv` module, you'll find logic that determines which BAR an incoming transaction targets. This often involves checking the address bits against the configured BAR sizes.
    *   You'll need to define how the incoming address `req_addr` (from the TLP) maps to an offset within your specific BAR.
    *   **Conceptual Example:**
        ```verilog
        // Example: Logic for BAR0 (assuming it's a 256MB 64-bit Memory BAR, for registers/data)
        // 'bar_hit[0]' would be an input signal indicating a hit on BAR0, typically from the PCIe core.
        // 'req_addr' is the incoming PCIe address.
        // 'req_be' is the byte enable from the TLP.
        // 'req_data' is the incoming write data.
        // 'rsp_data' is the outgoing read data.

        // Assuming BAR0 is 256MB (2^28 bytes), address bits [27:0] are within the BAR.
        localparam BAR0_SIZE_BITS = 28; // 2^28 = 256MB

        reg [31:0] internal_register_0; // Example register within BAR0
        reg [31:0] internal_register_1; // Another example register

        assign bar0_offset = req_addr[BAR0_SIZE_BITS-1:0]; // Extract offset within BAR0

        always_comb begin
            // Default response
            rsp_data = 32'hFFFFFFFF; // Default to all F's or similar for unmapped regions

            if (bar_hit[0]) begin // If the transaction targets BAR0
                if (req_write) begin // This is a write operation
                    case (bar0_offset)
                        // Example: Map offset 0x0 to internal_register_0
                        32'h0000_0000: begin
                            if (req_be[3]) internal_register_0[31:24] = req_data[31:24];
                            if (req_be[2]) internal_register_0[23:16] = req_data[23:16];
                            if (req_be[1]) internal_register_0[15:8]  = req_data[15:8];
                            if (req_be[0]) internal_register_0[7:0]   = req_data[7:0];
                        end
                        // Example: Map offset 0x4 to internal_register_1
                        32'h0000_0004: begin
                            if (req_be[3]) internal_register_1[31:24] = req_data[31:24];
                            if (req_be[2]) internal_register_1[23:16] = req_data[23:16];
                            if (req_be[1]) internal_register_1[15:8]  = req_data[15:8];
                            if (req_be[0]) internal_register_1[7:0]   = req_data[7:0];
                        end
                        // Add more register mappings or memory accesses (e.g., BRAM access)
                        default: begin
                            // Handle unmapped writes within BAR0, e.g., ignore or log
                        end
                    endcase
                end else if (req_read) begin // This is a read operation
                    case (bar0_offset)
                        // Example: Read from internal_register_0
                        32'h0000_0000: rsp_data = internal_register_0;
                        // Example: Read from internal_register_1
                        32'h0000_0004: rsp_data = internal_register_1;
                        // Add more register mappings or memory accesses (e.g., BRAM access)
                        default: begin
                            rsp_data = 32'h0; // Return 0 for unmapped reads or specific error value
                        end
                    endcase
                end
            end
        end
        ```
    *   **Handle Data Transfers:** The `always_comb` block (or `always_ff` for sequential logic) should define how `rsp_data` is generated for reads and how internal registers/memories are updated for writes, based on the `bar0_offset` and byte enables (`req_be`).

3.  **Implement BRAM Accesses (if BAR maps to BRAM):**
    *   If a BAR maps to a large block of memory (e.g., 256MB), you'll typically instantiate a BRAM IP core (as discussed in 8.2.1) and interface your `bar_controller` logic to it. The `bar_controller` would provide the address and control signals to the BRAM.
    *   **Conceptual BRAM Integration (simplified):**
        ```verilog
        // Inside pcileech_tlps128_bar_controller.sv or a sub-module
        // Interface to BRAM
        wire [BAR0_SIZE_BITS-1:0] bram_addr;
        wire [31:0] bram_wr_data;
        wire [3:0] bram_wr_en; // Byte enables for BRAM
        wire bram_wr_ce;
        wire bram_rd_ce;
        wire [31:0] bram_rd_data;

        // Map TLP signals to BRAM interface
        assign bram_addr = bar0_offset;
        assign bram_wr_data = req_data;
        assign bram_wr_en = req_be;
        assign bram_wr_ce = bar_hit[0] && req_write;
        assign bram_rd_ce = bar_hit[0] && req_read;

        // Instantiate the BRAM IP core
        bram_bar_zero bram_inst ( // Assuming 'bram_bar_zero' is your BRAM IP module
            .clka(clk),
            .ena(1'b1),
            .wea(bram_wr_en),
            .addra(bram_addr),
            .dina(bram_wr_data),
            .douta(bram_rd_data)
        );

        // For reads from BAR0, output data from BRAM
        if (bar_hit[0] && req_read) begin
            rsp_data = bram_rd_data;
        end
        ```

4.  **Save Changes:**
    *   After implementing the logic for each BAR, save the `pcileech_tlps128_bar_controller.sv` file.
    *   **Verify Functionality:** This logic is complex. Thorough simulation (using a test bench) and later hardware testing will be crucial to ensure correct behavior.

#### **8.2.3 Handling Multiple BARs**

Properly managing multiple BARs is essential for devices that expose several distinct memory or I/O regions. The `bar_controller` module typically handles all BARs.

**Steps:**

1.  **Implement Logic for Each BAR:**
    *   Within `pcileech_tlps128_bar_controller.sv`, extend the logic to handle all enabled BARs (BAR0, BAR1, BAR2, etc.) that your donor device uses.
    *   **Separate Logic Blocks:** For clarity and maintainability, create distinct `if/else if` blocks or `case` statements that activate based on which `bar_hit` signal is asserted.
        ```verilog
        // BAR0 Handling
        if (bar_hit[0]) begin
            // BAR0 specific read/write logic for its registers/memory
        end else if (bar_hit[1]) begin
            // BAR1 specific read/write logic for its registers/memory
        end else if (bar_hit[2]) begin
            // BAR2 specific logic
        end
        // ... continue for other BARs
        ```
    *   **Define Registers and Memories:** Allocate separate sets of registers or connect distinct BRAM instances for each BAR as needed.

2.  **Ensure Non-Overlapping Address Spaces:**
    *   While the PCIe IP core handles the negotiation of distinct address spaces for each BAR with the host OS, your internal firmware logic *must* assume that these spaces are separate and non-overlapping.
    *   **Validate Address Ranges**: Double-check your BAR size configurations in the PCIe IP core to ensure they are distinct and correctly aligned to power-of-two boundaries as per PCIe specifications.
    *   **Update Address Decoding**: Your `bar_controller` logic relies on the `bar_hit` signals generated by the PCIe IP core. Ensure these are correctly interpreted and lead to the unique handling logic for each BAR.

3.  **Test BAR Accesses:**
    *   **Simulation Testing:** Before hardware deployment, use simulation tools (e.g., Vivado Simulator) with a comprehensive test bench to verify all read and write operations to each BAR.
        *   Send Memory Write TLPs to specific offsets within each BAR.
        *   Send Memory Read TLPs to specific offsets within each BAR and verify the returned data.
    *   **Hardware Testing:** After programming the FPGA, use host-side software tools (like PCILeech client software, or custom C/Python scripts) to access and verify each BAR.
        *   **Linux**: Use `lspci -vvv` to inspect BAR mappings (`Memory at XXXX (64-bit, prefetchable) [size=YYYY]`). You can then use `devmem2` or a custom kernel module to read/write to these mapped addresses.
        *   **Windows**: Use tools like "RW-Everything" or custom user-mode applications to inspect and interact with the mapped memory regions.
        *   Perform various read/write patterns to ensure data integrity and correct addressing across all BARs.

---

### **8.3 Interrupts: Make Noise Like Real Hardware or Don't Pretend You're Alive**

Emulating power management features and implementing interrupts are critical for devices that need to interact closely and efficiently with the host operating system's power management and interrupt handling mechanisms. Without these, the emulated device may not appear fully functional, or performance could be suboptimal.

#### **8.3.1 Power Management Configuration**

Implementing power management allows the emulated device to support various power states (e.g., D0, D3hot), contributing to system-wide power efficiency and compliance with operating system expectations. The host OS will query the device's capabilities and send commands to transition between these states.

**Steps:**

1.  **Enable Power Management in PCIe IP Core:**
    *   **Access Capabilities:** In the PCIe IP core customization window (`pcie_7x_0.xci`), navigate to the **PCIe Capabilities** tab.
    *   **Enable Power Management:** Look for a section or option related to **Power Management Capability**. Ensure this is checked or enabled to include the Power Management (PM) capability structure in the device's configuration space.

2.  **Set Power States Supported:**
    *   **Configure Supported States:** Within the Power Management Capability section of the IP core, specify which power states the device supports. These are typically checkboxes or dropdowns. Match these settings to the donor device's capabilities as observed by Arbor.
        *   **D0 (Fully On/Operational)**: Always supported.
        *   **D1, D2 (Intermediate States)**: Optional, for low-power idle states.
         *   **D3hot (Power Down, Auxiliary Power Present)**: Configuration space remains reachable. The device responds to configuration accesses and participates in PM state transitions. Your logic should model software-visible D-state behavior: quiesce activity, retain/reset defined state, accept/reject requests, and resume deterministically.
        *   **D3cold (Power Off)**: No power to the device.
    *   **Example**: If the donor device only supports D0 and D3hot, enable only those.

3.  **Implement Power State Logic in Firmware:**
    *   **Open `pcileech_pcie_cfg_a7.sv` (or relevant control module):** You'll typically need to modify the firmware to reflect and potentially react to power state transitions commanded by the host. The PCIe core itself handles much of the protocol, but your user logic needs to know the state.
    *   **Handle Power Management Control and Status Register (PMCSR) Writes:** The host OS changes the device's power state by writing to specific bits in the PMCSR, which is part of the PM Capability structure. Your firmware should ideally have logic to read these bits and adjust device behavior (e.g., pausing/resuming operations, enabling/disabling clocks).
        ```verilog
        // Example: Part of pcileech_pcie_cfg_a7.sv or a dedicated PM module
        // Assume 'cfg_write' is asserted for config writes, 'cfg_address' is the offset, 'cfg_writedata' is the data.
        // The D-state bits are at offset 0x04 within the PM capability structure, bits [1:0].

        // PMCSR Register (Internal representation)
        reg [15:0] pmcsr_reg = 16'h0000; // Initialize to D0

        // User logic signal indicating current power state
        reg [1:0] current_d_state = 2'b00; // 00 = D0, 01 = D1, 10 = D2, 11 = D3hot

        always @(posedge clk) begin
            if (reset) begin
                pmcsr_reg <= 16'h0000;
                current_d_state <= 2'b00; // Reset to D0
            end else begin
                // Example: Capture writes to PMCSR (if directly handled in user logic)
                // Note: The PCIe IP core manages much of this, but your user logic might need to read values from it.
                // Assuming the PCIe core provides an output reflecting the current D-state:
                // assign current_d_state = pcie_core_d_state_output;

                // If user logic *needs* to write to PMCSR (less common, usually read-only status)
                // Or if it needs to process the command
                // if (cfg_write && (cfg_address == PM_CAP_OFFSET + 2'h04)) begin // PMCSR is at +0x04 from PM Cap base
                //     pmcsr_reg[1:0] <= cfg_writedata[1:0]; // Capture New D-state
                //     // current_d_state <= cfg_writedata[1:0]; // Update internal state
                // end

                // In PCILeech, the PCIe core manages the PMCSR. You would likely read a signal from the core.
                // For demonstration, let's assume 'pcie_d_state' is an input from the IP core.
                current_d_state <= pcie_d_state; // Update based on PCIe core's status
            end
        end

        // Example: Logic reacting to D-state changes
        always @(*) begin
            if (current_d_state == 2'b11) begin // D3hot state
                // Disable power to non-essential blocks, pause operations,
                // assert a signal to main DMA logic to stop activity.
                // For instance: dma_engine_enable = 1'b0;
            end else if (current_d_state == 2'b00) begin // D0 state
                // Enable full functionality
                // For instance: dma_engine_enable = 1'b1;
            end
        end
        ```
    *   **Manage Power State Effects:** Implement logic to alter the device's internal behavior (e.g., enable/disable clocks, put sub-modules into low-power modes) based on the `current_d_state`. This is crucial for accurate power consumption emulation and to ensure the device responds correctly to OS commands.

4.  **Save Changes:**
    *   Save any modified firmware files.
    *   Thoroughly test power management features through simulation or hardware testing (e.g., Windows "Sleep" or "Hibernate" functions, or Linux `poweroff` commands to see if the device transitions correctly).

#### **8.3.2 MSI/MSI-X Configuration**

Implementing Message Signaled Interrupts (MSI) or its extended version (MSI-X) allows the emulated device to use message-based interrupts. These are significantly more efficient and scalable than traditional pin-based interrupts (INTx) and are the preferred method for modern PCIe devices. MSI/MSI-X allow the device to notify the CPU by writing a special TLP to a specific memory address.

**Steps:**

1.  **Enable MSI/MSI-X in PCIe IP Core:**
    *   **Access Interrupts Configuration:** In the PCIe IP core customization window (`pcie_7x_0.xci`), navigate to the **Interrupts** tab or a section specifically labeled **MSI/MSI-X Capabilities**.
    *   **Select Interrupt Type:** Choose between **MSI** or **MSI-X** based on the donor device's capabilities. MSI-X is generally preferred for its flexibility (more vectors, per-vector masking).
    *   **Configure Number of Supported Vectors:** Set the number of interrupt vectors (messages) that the device will support. Match this to the donor device.
        *   **MSI** supports up to 32 vectors (typically 1, 2, 4, 8, 16, or 32).
        *   **MSI-X** supports up to 2048 vectors, allowing for more granular interrupt sources.
    *   **Enable Capabilities:** Ensure that the MSI or MSI-X capability structure is explicitly enabled to be included in the device's configuration space. This is how the host OS discovers the device's interrupt capabilities.

2.  **Implement Interrupt Logic in Firmware:**
    *   **Open `pcileech_pcie_tlp_a7.sv` (or user logic module):** This file is generally responsible for user-defined TLP generation and might be the appropriate place to initiate MSI/MSI-X messages. However, the *trigger* for the interrupt will come from your custom logic.
    *   **Define Interrupt Signals:** Declare internal signals that will indicate when an interrupt needs to be generated.
        ```verilog
        // In a custom module (e.g., 'my_device_logic.sv') that interfaces with TLP generation logic
        reg msi_trigger_signal; // Assert this when an interrupt condition occurs
        ```
    *   **Implement Interrupt Generation Logic:** Define the conditions under which an interrupt should be triggered. This typically involves event detection within your emulated device's logic.
        ```verilog
        // Inside 'my_device_logic.sv'
        input wire clk;
        input wire reset;
        input wire event_data_ready; // Example: Input from your logic when data is ready

        always @(posedge clk or posedge reset) begin
            if (reset) begin
                msi_trigger_signal <= 1'b0;
            end else if (event_data_ready) begin // When a specific event happens
                msi_trigger_signal <= 1'b1; // Trigger an MSI
            end else begin
                msi_trigger_signal <= 1'b0; // Clear after one cycle or when acknowledged
            end
        end
        ```
    *   **Connect to PCIe Core's MSI Interface:** The `msi_trigger_signal` (or similar output from your custom logic) needs to be connected to the appropriate input of the PCIe IP core (e.g., `s_axis_tdata_tready`, `s_axis_tdata_tvalid`, `s_axis_tdata_tlast` if using an AXI-Stream interface for MSI TLPs, or dedicated MSI request ports provided by the IP core). The PCIe core then forms and sends the actual MSI/MSI-X TLP. Consult the Xilinx PCIe IP core documentation for precise interface details.

3.  **Save Changes:**
    *   Save any modified firmware files after implementing the interrupt logic.
    *   **Check for Timing Constraints:** New logic, especially interrupt paths, can be timing-critical. Ensure that the synthesis and implementation tools do not report any timing violations related to your interrupt generation logic.

#### **8.3.3 Implementing Interrupt Handling Logic (Device Side)**

Beyond just enabling the capability, defining *when* and *how* interrupts are generated by your emulated device is crucial for its proper interaction with the host's interrupt handling mechanisms and driver behavior. This involves creating the internal logic that asserts the interrupt request.

**Steps:**

1.  **Define Interrupt Conditions:**
    *   **Identify Trigger Events:** Based on your donor device's behavior, determine the specific internal events that should cause your emulated device to generate an interrupt.
        *   **Examples**: Data transfer completion, new data ready in a receive buffer, an internal error condition, completion of a specific command, link status change.
    *   **Implement Condition Logic:** Use combinational or sequential logic within your custom SystemVerilog modules to precisely detect these events and generate a short-duration pulse or level-based signal that indicates an interrupt request.

2.  **Create Interrupt Generation Module (Modular Design):**
    *   It's a good practice to encapsulate your interrupt generation logic into a separate, dedicated module for clarity, reusability, and easier debugging. This module would take internal events as inputs and produce an `msi_req` (or similar) output that connects to the PCIe core.
        ```verilog
        // File: interrupt_generator.sv
        module interrupt_generator (
            input wire clk,
            input wire reset,
            input wire event_trigger,        // Input signal from your custom logic (e.g., data_ready, error_flag)
            output reg msi_req_o            // Output: Assert to request an MSI/MSI-X
        );

        // Simple pulse generator for MSI (one-shot interrupt)
        reg event_trigger_d1;

        always @(posedge clk or posedge reset) begin
            if (reset) begin
                msi_req_o <= 1'b0;
                event_trigger_d1 <= 1'b0;
            end else begin
                event_trigger_d1 <= event_trigger;
                // Generate a single-cycle pulse when event_trigger goes high
                if (event_trigger && !event_trigger_d1) begin
                    msi_req_o <= 1'b1; // Assert MSI request
                end else begin
                    msi_req_o <= 1'b0; // De-assert after one cycle
                end
            end
        end

        endmodule // interrupt_generator
        ```
    *   **Integrate with Main Firmware:** Instantiate this `interrupt_generator` module in your top-level user logic (e.g., within `pcileech_squirrel_top.sv` or a module it instantiates) and connect its `msi_req_o` output to the PCIe IP core's MSI input.

3.  **Ensure Proper Timing and Sequencing:**
    *   **Adhere to PCIe Specifications:** MSI/MSI-X messages are TLPs. Ensure that the generation of these messages adheres to PCIe TLP formatting, flow control, and timing requirements. The PCIe IP core handles much of this, but your input signals to it must be stable and correctly timed.
    *   **Manage Interrupt Latency:** Optimize your logic to minimize any unnecessary delay between the internal event occurrence and the assertion of the `msi_req_o` signal.

4.  **Test Interrupt Delivery:**
    *   **Simulation:** Use a comprehensive test bench to simulate scenarios where interrupts should be generated. Verify that your `msi_req_o` signal behaves as expected and that the PCIe core generates the correct MSI/MSI-X TLPs.
    *   **Hardware Testing:**
        *   Program the FPGA with the updated firmware.
        *   Use host-side software to trigger the events that should cause an interrupt (e.g., initiating a DMA transfer that completes).
        *   Confirm that the interrupts are received by the host operating system. On Linux, `dmesg` can show interrupt messages. On Windows, you might use specific driver debug tools or Event Viewer.
        *   **Debugging Tools:** Utilize Vivado's Integrated Logic Analyzer (ILA) cores (as discussed later in Section 12) to monitor `event_trigger`, `msi_req_o`, and the PCIe core's TLP output signals in real time to verify correct interrupt generation.

5.  **Save Changes:**
    *   Finalize all code modifications and save the relevant firmware files.
    *   Review and refine your interrupt logic based on testing results to ensure reliability.

---

## **9. Emulating Device-Specific Capabilities**

Beyond the standard PCIe configuration space and generic DMA functionality, many donor devices possess unique capabilities, custom registers, or vendor-specific features that are critical for their full functionality or for interacting with their proprietary drivers. Accurate emulation requires understanding and replicating these nuances. This section delves into implementing such advanced capabilities, enabling a more faithful and fully functional emulation.

### **9.1 Implementing Advanced PCIe Capabilities**

The PCIe specification includes various *extended capabilities* beyond the basic configuration space. These capabilities provide features like advanced error reporting, power management, virtualization, and more. Implementing these helps your emulated device appear more legitimate and interact correctly with modern host systems.

**Steps:**

1.  **Identify Required Extended Capabilities:**
    *   When gathering donor device information with tools like Arbor, carefully look for and document any Extended Capabilities present in the donor's configuration space. These are usually found beyond the initial 256 bytes of standard configuration space.
    *   **Common Examples:**
        *   **Advanced Error Reporting (AER)**: Provides robust error detection, logging, and reporting mechanisms for PCIe links.
        *   **Device Serial Number (DSN)**: (Already covered in Section 6.2).
        *   **Power Management (PM)**: (Already covered in Section 8.3.1).
        *   **PCI Express (PCIe) Capability Structure**: (Already covered in Section 8.1 for Link Speed/Width, Max Payload/Read Request, but also includes other fields like Device Control/Status, Link Control/Status).
        *   **Virtual Channel (VC)/Multi-Function Virtual Channel (MFVC)**: For Quality of Service (QoS) and traffic management.
        *   **Precision Time Measurement (PTM)**: For synchronizing time across devices.
        *   **Latency Tolerance Reporting (LTR)**: For power management based on latency requirements.
        *   **Resettable FPC (Function-Level Reset)**: For more granular resets.

2.  **Enable Capabilities in Vivado PCIe IP Core:**
    *   Access the PCIe IP core customization window (`pcie_7x_0.xci`) in Vivado.
    *   Navigate through the various tabs (e.g., "PCIe Capabilities," "Extended Capabilities," "Advanced Options").
    *   Look for checkboxes or dropdowns to enable and configure the specific extended capabilities identified from your donor device.
    *   **Example (AER):** You'll find a section for "Advanced Error Reporting" where you can enable it and configure its registers (e.g., severity masks).
    *   **Note:** The Xilinx PCIe IP core provides a high degree of configurability for many standard and extended capabilities. It's often just a matter of enabling the correct options in the GUI.

3.  **Implement Firmware Logic for Capability Registers (if necessary):**
    *   While the PCIe IP core handles the *presence* and much of the *protocol* for these capabilities, some capabilities expose registers that your custom firmware might need to read or write, or whose values your firmware needs to react to.
    *   **Example (AER):** If your emulated device detects an internal error that should be reported via AER, your firmware needs to write to specific AER error status registers (which might be exposed as part of the BARs or handled internally by the PCIe core and then reflected to user logic). Your user logic would then assert an error input to the PCIe core.
    *   **Example (Power Management):** As discussed in 8.3.1, your firmware needs to react to the D-state changes signaled by the PCIe core.
    *   **Process:**
        *   Identify the specific registers within each enabled capability structure that your donor device's driver interacts with.
        *   Locate the corresponding signals or logic within the PCILeech-FPGA framework that interface with these registers (often within `pcileech_pcie_cfg_a7.sv` or the `bar_controller`).
        *   Implement read/write logic for these registers, ensuring your emulated device's internal state accurately reflects the values the driver expects.

### **9.2 Emulating Vendor-Specific Features**

This is where true "full device emulation" becomes highly specialized. Many real-world devices have unique registers, undocumented commands, custom data formats, or proprietary control flows that differentiate them. Replicating these requires deeper analysis and custom HDL development.

**Steps:**

1.  **Reverse Engineer Vendor-Specific Behaviors:**
    *   This is often the most challenging part.
    *   **Static Analysis (Driver/Firmware):** Disassemble the donor device's official driver (Windows `.sys`, Linux `.ko`) or the device's original firmware (if available). Look for unique I/O or MMIO access patterns, magic values, or sequences of register writes. Tools like Ghidra, IDA Pro, or objdump can be invaluable.
    *   **Dynamic Analysis (Driver Execution):** Run the donor device with its driver and monitor PCIe traffic using a **PCIe Protocol Analyzer** (e.g., Teledyne LeCroy, Keysight, as discussed in Section 12.2). This is the gold standard for understanding actual TLP exchanges, including vendor-defined messages and sequences of register accesses. Pay attention to:
        *   Specific memory addresses accessed within BARs.
        *   Read/write patterns to these addresses.
        *   The values written to or read from specific registers.
        *   Timing relationships between commands and responses.
    *   **System Calls/API Monitoring**: On the host, use tools like Procmon (Windows) or `strace` (Linux) to see how the driver interacts with the OS and what specific device I/O control (IOCTL) codes it uses, which might correspond to specific hardware operations.
    *   **Hardware Sniffing**: If possible, use a hardware sniffer (like a Saleae Logic Analyzer) to capture signals on the device's internal buses (e.g., SPI, I2C) if it has external flash or components.

2.  **Implement Custom Registers and Logic within BARs:**
    *   Once you've identified vendor-specific registers or command protocols, you'll need to define these within your FPGA firmware, typically as memory-mapped registers accessible via one of your BARs.
    *   **Create Internal Registers:** Declare `reg` variables in your SystemVerilog code to represent these custom registers.
        ```verilog
        // In your pcileech_tlps128_bar_controller.sv or a sub-module
        reg [31:0] custom_control_reg;
        reg [31:0] custom_status_reg;
        reg [31:0] custom_data_reg;

        // Example: Map these to specific offsets within BAR0 (assuming BAR0 is large enough)
        // Adjust the 'bar0_offset' case statement (from Section 8.2.2)
        // ...
        if (bar_hit[0]) begin
            if (req_write) begin
                case (bar0_offset)
                    32'h0000_1000: custom_control_reg <= req_data; // Custom control register
                    32'h0000_1004: custom_data_reg <= req_data;    // Custom data write register
                    // ... other mappings
                endcase
            end else if (req_read) begin
                case (bar0_offset)
                    32'h0000_1000: rsp_data = custom_control_reg; // Read control register
                    32'h0000_1008: rsp_data = custom_status_reg;  // Custom status register
                    // ... other mappings
                endcase
            end
        end
        // ...
        ```
    *   **Implement Behavioral Logic:** Create SystemVerilog logic (state machines, combinational logic) that:
        *   Responds to writes to your `custom_control_reg`. For example, a specific bit in this register might trigger a DMA transfer, clear a status flag, or initiate an internal operation.
        *   Updates your `custom_status_reg` based on the internal state of your emulated device (e.g., "operation complete," "error occurred," "data available").
        *   Processes data written to `custom_data_reg` or provides data from it on reads, mimicking the donor device's data paths.

3.  **Emulate Vendor-Specific Messages (if applicable):**
    *   Some complex devices might use "Vendor Defined Messages" (VDMs) over PCIe for specific control or communication. If your analysis reveals such messages, you would need to:
        *   Enable VDM support in the PCIe IP core (if available).
        *   Implement TLP generation logic (as will be discussed in Section 10) to craft and send these VDMs.
        *   Implement TLP reception and parsing logic to interpret incoming VDMs from the host.

4.  **Validate Emulated Behavior:**
    *   **Iterative Testing:** This is a highly iterative process. Make small changes, compile, flash, and test.
    *   **Driver Loading:** Does the donor device's driver load correctly without errors?
    *   **Functional Tests:** Can the driver initiate basic operations? Does it get the expected responses from your emulated registers?
    *   **Application Tests:** Can applications that rely on the donor device function correctly with your emulated version?
    *   **Debugging:** Use ILA and PCIe protocol analyzers extensively to compare your emulated device's behavior against the captured behavior of the real donor device. Look for discrepancies in TLP timing, register values, and overall protocol flow.

---

## **10. Completions: Where Bad Tags Go to Die**

Transaction Layer Packets (TLPs) are the fundamental units of communication in the PCIe architecture. Every interaction between a host system and a PCIe device, from configuration reads to data transfers, is encapsulated within one or more TLPs. Accurate TLP emulation is not just important; it is *crucial* for your emulated device to interact properly with the host system, ensuring that drivers function correctly and that data moves as expected.

### **10.1 Understanding and Capturing TLPs**

Before you can craft custom TLPs, you must deeply understand their structure and common types. Capturing real-world TLPs from your donor device provides the most accurate blueprint.

#### **10.1.1 Learning the TLP Structure**

A TLP is generally composed of a header, an optional data payload, and an optional End-to-End CRC (ECRC). The header is paramount, defining the TLP's type, transaction specifics, and routing information.

*   **Components of a TLP**:
    *   **Header**: The most important part, typically 3 or 4 Double Words (Dwords = 4 bytes). It contains critical fields that define the TLP's purpose and how it should be handled:
        *   **Fmt (Format) and Type**: Defines the TLP's format (3DW/4DW, with/without data) and its specific purpose (e.g., Memory Read Request, Memory Write, Completion, Configuration Read/Write).
        *   **Length**: Specifies the size of the data payload in Dwords.
        *   **Requester ID (Bus, Device, Function)**: Identifies the PCIe function that initiated the request. Essential for routing completions back to the correct source.
        *   **Tag**: A unique identifier assigned by the Requester to a transaction, allowing a Completer to match a Completion TLP to its original Request TLP.
        *   **Address**: For Memory/IO transactions, this is the target memory or I/O address.
        *   **First DW Byte Enable (FBE)** and **Last DW Byte Enable (LBE)**: Specify which bytes within the first and last Dwords of the payload are valid for write operations, or which bytes are being requested for read completions.
        *   **Traffic Class (TC)** and **Transaction ID (TID)**: For QoS and ordering rules.
    *   **Data Payload (Optional)**: Present in TLPs like Memory Write, Configuration Write, and Read Completions. It contains the actual data being transferred.
    *   **End-to-End CRC (ECRC) (Optional)**: A 32-bit CRC covering the entire TLP, providing optional end-to-end data integrity. ECRC generation and checking is IP-core-dependent; on Xilinx hard-IP-based designs, do not assume user RTL or host software directly builds or verifies it without consulting the specific core's ECRC support.

*   **Understanding Common TLP Types**: Your firmware will primarily deal with these:
    *   **Memory Read Request (MRd)**: A TLP sent by a Requester (e.g., host CPU, or your FPGA as a DMA master) to read data from a specific memory address.
    *   **Memory Read Completion (CplD)**: A TLP sent by a Completer (e.g., your FPGA responding to a host MRd) carrying the requested data.
    *   **Memory Write (MWr)**: A TLP sent by a Requester (e.g., host CPU, or your FPGA) to write data to a specific memory address.
    *   **Completion Without Data (Cpl)**: A TLP sent by a Completer to acknowledge a non-posted request that does not return data (e.g., a successful Configuration Write or an I/O Write). Memory Writes (MWr) are posted transactions and do NOT receive a completion.
    *   **Configuration Read Request (CfgRd)**: A TLP from the host to read registers in the device's Configuration Space.
    *   **Configuration Read Completion (CplD)**: A TLP from the device returning data for a CfgRd.
    *   **Configuration Write Request (CfgWr)**: A TLP from the host to write to registers in the device's Configuration Space.
    *   **Vendor-Defined Messages (VDM)**: Custom TLPs used by specific vendors for proprietary communication.

#### **10.1.2 Capturing TLPs from the Donor Device**

Capturing real PCIe traffic from your donor device is invaluable. It provides concrete examples of TLP structures, sequences, and timing, allowing you to replicate them precisely.

*   **Steps**:
    1.  **Set Up a PCIe Protocol Analyzer**:
        *   The most effective method involves using dedicated hardware tools, often referred to as "PCIe Protocol Analyzers." These devices sit in-line between the host and the donor PCIe card, passively capturing all traffic.
        *   **Examples**:
            *   **Teledyne LeCroy PCIe Analyzers**: Industry standard, highly capable, but significant investment.
            *   **Keysight PCIe Analyzers**: Another leading vendor.
            *   (For basic debugging, some higher-end logic analyzers with PCIe decoders might offer limited TLP viewing, but a true protocol analyzer is superior).
    2.  **Capture Transactions**:
        *   Install the donor device in a test system with the protocol analyzer connected.
        *   Run the donor device's driver and any associated applications.
        *   Monitor and record the PCIe transactions during normal operation, and especially during critical phases like:
            *   Device enumeration (when the OS first detects it).
            *   Driver loading and initialization.
            *   Typical data transfer operations (e.g., large file copies for a storage device, network traffic for a NIC).
            *   Device-specific commands or diagnostics.
    3.  **Analyze Captured TLPs**:
        *   Use the protocol analyzer's sophisticated software to dissect the captured TLPs. The software will decode fields, provide chronological views, and allow filtering and searching.
        *   Pay close attention to:
            *   The exact `Fmt` and `Type` fields.
            *   `Requester ID` and `Tag` values (especially for completions).
            *   The `Address` and `Length` for memory transactions.
            *   The contents of `Data Payload` for writes and read completions.
            *   Any vendor-specific fields or custom TLPs.

#### **10.1.3 Documenting Key TLP Transactions**

Structured documentation of captured TLPs creates a blueprint for your emulation.

*   **Steps**:
    1.  **Identify Critical Transactions**:
        *   Focus on TLPs that are essential for the device's core functionality. These include:
            *   **Initialization Sequence**: The series of configuration reads/writes the OS performs during enumeration.
            *   **Driver Initialization**: The commands and data exchanged when the driver starts up.
            *   **Primary Data Transfers**: How `MWr` and `MRd` TLPs are structured and completed for the device's main function.
            *   **Error Handling**: How the device reports errors (e.g., Completion with Completer Abort (CA), Unsupported Request (UR)).
            *   **Power Management Transitions**: TLPs related to D-state changes.
            *   **Interrupt Generation**: How MSI/MSI-X messages are sent.
    2.  **Create Detailed Documentation**:
        *   For each key TLP sequence, record:
            *   The **TLP Type** (e.g., MWr, MRd, CplD).
            *   Its **header fields** (Fmt, Type, Requester ID, Tag, Length, Address, Byte Enables).
            *   The **data payload** (if applicable).
            *   The **sequence number** or order within a transaction.
            *   The **conditions** under which it is sent (e.g., "sent by host when driver initializes," "sent by device on DMA completion").
            *   Any **expected responses** or follow-up TLPs.
        *   Screenshots from the protocol analyzer can be very helpful here.
    3.  **Understand Timing and Sequencing**:
        *   Beyond just the TLP content, the *timing* and *sequence* of TLPs are vital. PCIe has strict ordering rules and flow control mechanisms. Pay attention to:
            *   **Delay between Request and Completion**: How quickly the real device responds.
            *   **Flow Control Credits**: How the device manages its buffer space for incoming/outgoing TLPs. While the Xilinx PCIe IP core handles basic flow control, for advanced emulation, knowing the donor's typical credit usage can be helpful.
            *   **Transaction Layer Packet Ordering**: Understand how posted (writes) and non-posted (reads, completions) transactions are ordered.

### **10.2 Crafting Custom TLPs for Specific Operations**

Once you understand the blueprint, you can translate this knowledge into your FPGA firmware (SystemVerilog) to actively generate and respond to TLPs. The PCILeech-FPGA framework provides abstraction layers, but for deep emulation, you might need to interact directly with the TLP generation/parsing logic.

#### **10.2.1 Implementing TLP Handling in Firmware**

Your firmware will need logic to both send and receive TLPs. The PCIe IP core handles the physical and data link layers, exposing a Transaction Layer interface (often AXI-Stream) to your user logic.

*   **Files to Modify (Primary)**:
    *   `pcileech-fpga/<your_board_variant>/src/pcileech_pcie_tlp_a7.sv` (or similar, depending on board variant)
        *   This file often contains the core logic for translating user requests into outbound TLPs and parsing incoming TLPs into signals for your user logic.
    *   `pcileech-fpga/<your_board_variant>/src/pcileech_tlps128_bar_controller.sv`
        *   This module specifically handles parsing incoming Memory Read/Write TLPs that target your device's BARs and generating corresponding Completion TLPs.

*   **Steps**:

    1.  **Understand the PCIe IP Core Interface**:
        *   Before writing TLP logic, thoroughly read the Xilinx PCIe IP Core User Guide (specifically the sections on the User Application Interface or AXI4-Stream Interface). This defines how your SystemVerilog logic connects to the PCIe core to send and receive TLPs. You'll typically interact with `s_axis_rx_tdata` (received TLP data), `s_axis_rx_tvalid` (valid TLP received), `m_axis_tx_tdata` (outgoing TLP data), `m_axis_tx_tready` (core ready to accept TLP), etc.

    2.  **Create TLP Generation Functions (for outbound TLPs)**:
        *   In `pcileech_pcie_tlp_a7.sv` (or a module that interfaces with `m_axis_tx_*`), you will write logic to assemble TLPs with the required headers and payloads. This often involves combining various fields into a `[127:0]` (for 128-bit interface) or `[63:0]` (for 64-bit interface) bus that feeds the PCIe core.
        *   **WARNING:** The TLP field positions, AXI-stream bus width, endianness, `tkeep`/`tlast` behavior, and Fmt/Type bit slicing shown below are **specific to one Xilinx PCIe core configuration and wrapper version.** The exact bit positions of `s_axis_rx_tdata` or `m_axis_tx_tdata` depend on your generated core's interface width, the wrapper generated by Vivado, and the Tcl parameters used during project generation. Do not copy these bit slices into your design. Use the field positions documented in your core's user guide (PG054 for 7-series) and verify with ILA captures against known-good transactions. The examples below show the *concept* of TLP field layout, not the implementation-ready bit positions for your board.
        *   **Example (Conceptual, Simplified -- NOT implementation-ready):**
            ```verilog
            // This is a conceptual helper function. In reality, you'd build a state machine
            // to send TLPs over an AXI-Stream interface, possibly using a FIFO.
            function automatic [95:0] create_3dw_tlp_header; // Assuming 3 Dwords = 96 bits
                input logic [7:0] tlp_type_fmt;   // Format and Type fields
                input logic [15:0] requester_id;  // BDF
                input logic [7:0] tag;
                input logic [7:0] lower_address_bits; // Or more complex address
                input logic [7:0] byte_enables;   // First DW Byte Enable

                begin
                    create_3dw_tlp_header = {
                        tlp_type_fmt,                     // Fmt[6:4], Type[3:0]
                        8'b0,                             // Reserved
                        4'b0,                             // TC[3:0] (Traffic Class)
                        3'b0,                             // Attr[2:0]
                        1'b0,                             // TH (TLP Hint)
                        2'b0,                             // D(igest) (ECRC presence)
                        1'b0,                             // EP (Poisoned)
                        1'b0,                             // TD (Type Dependent)
                        // DW0: Fmt, Type, TC, Attr, TH, D, EP, TD, Length (not in 3DW, 4DW has it)

                        requester_id,                     // Requester ID (Bus[7:0], Device[4:0], Function[2:0])
                        tag,                              // Tag
                        lower_address_bits,               // Example: lower bits of address or a portion of data
                        byte_enables,                     // First DW Byte Enable
                        4'b0,                             // Reserved
                        4'b0                              // Last DW Byte Enable (for MWr usually)
                        // DW1, DW2... fields
                    };
                end
            endfunction

            // Example: Generating a Completion with Data (CplD) in a state machine
            // This is just a snippet, not a full implementation
            localparam  CPLD_3DW_FMT = 8'h4A; // Fmt=100 (4DW, with data), Type=1010 (Cpl)
            localparam  CPL_D_FMT_TYPE_LEN = 8'h4A; // Fmt=010 (3DW, with data), Type=01010 (CplD). A normal Completion with Data uses a 3-DW header, not 4-DW.

            // ... state machine to send TLP
            // In a state where you are ready to send CplD
            if (tx_ready_from_pcie_core) begin
                // Build the header and payload
                // For a CplD, you need Complier ID, Status, Byte Count, Requester ID, Tag, Completion ID, Lower Address
                // And then the actual read data payload
                m_axis_tx_tdata_reg = {
                    CPL_D_FMT_TYPE_LEN,         // Byte 0: Fmt/Type
                    tlp_length_dw_minus_one,    // Byte 1: TLP Length (in Dwords) - 1
                    status_completion_bits,     // Byte 2: Cpl Status, BCM, Rsvd
                    byte_count_dws_upper,       // Byte 3: Byte Count (upper bits)
                    requester_id,               // Byte 4-5: Requester ID (from original MRd)
                    tag,                        // Byte 6: Tag (from original MRd)
                    byte_count_dws_lower,       // Byte 7: Byte Count (lower bits)
                    completion_id,              // Byte 8-9: Completion ID (your BDF)
                    lower_address_from_request  // Byte 10-11: Lower Address from Request
                    // ...followed by actual data payload
                };
                m_axis_tx_tvalid_reg = 1'b1;
                m_axis_tx_tlast_reg = 1'b1; // Last TLP segment
                // ... state transition to wait for tready
            end
            ```
        *   **Note**: The actual implementation involves state machines, FIFOs, and adherence to the AXI-Stream protocol for the PCIe IP core. The PCILeech-FPGA framework already provides a good base for this, but you might extend or modify it for very specific TLP behaviors.

    3.  **Handle TLP Reception (for inbound TLPs)**:
        *   Implement logic to parse incoming TLPs from the PCIe core's receive interface (e.g., `s_axis_rx_tdata`, `s_axis_rx_tvalid`).
        *   This parsing involves:
            *   Checking `s_axis_rx_tvalid` to know a TLP is present.
            *   Reading the `Fmt` and `Type` fields from the TLP header to determine its purpose.
            *   Extracting relevant fields like `Requester ID`, `Tag`, `Address`, `Length`, and `Data Payload`.
        *   Use `case` statements or `if/else if` blocks based on the TLP type to route the information to the appropriate internal logic (e.g., `bar_controller` for Memory Writes, a configuration module for Configuration Writes).
        *   **Example (Conceptual, Simplified Parsing):**
            ```verilog
            // In pcileech_pcie_tlp_a7.sv or a TLP parser module
            input wire [127:0] s_axis_rx_tdata;
            input wire s_axis_rx_tvalid;
            output wire s_axis_rx_tready; // Needs to be asserted to accept more data

            reg [7:0] received_tlp_fmt_type;
            reg [15:0] received_requester_id;
            // ... declare other parsed fields

            assign s_axis_rx_tready = 1'b1; // **NOT PRODUCTION RTL: field-extraction toy.** A real AXI-Stream parser must manage backpressure with framing (header/data beat boundaries, partial data, transaction metadata retention, downstream capacity) before accepting more traffic. This sketch shows field offsets only.

            always @(posedge clk) begin
                if (s_axis_rx_tvalid) begin
                    received_tlp_fmt_type = s_axis_rx_tdata[127:120]; // Assuming highest bits
                    received_requester_id = s_axis_rx_tdata[111:96]; // Example offset

                    // Decode based on TLP type
                    case (received_tlp_fmt_type[3:0]) // Only TLP Type bits
                        4'h0: // Memory Write (3DW or 4DW depending on Fmt)
                            // Extract address, length, payload and pass to BAR controller
                            begin
                                // Pass to BAR controller for writing to emulated memory
                                // bar_write_enable = 1'b1;
                                // bar_write_address = s_axis_rx_tdata[...];
                                // bar_write_data = s_axis_rx_tdata[...];
                            end
                        4'h1: // Memory Read
                            // Extract address, length, and pass to BAR controller for reading
                            begin
                                // bar_read_enable = 1'b1;
                                // bar_read_address = s_axis_rx_tdata[...];
                                // (Completion will be generated by BAR controller)
                            end
                        // ... other TLP types
                        default: begin
                            // Handle unsupported or reserved TLP types (e.g., log, error)
                        end
                    endcase
                end
            end
            ```

    4.  **Ensure Compliance**:
        *   Strictly verify that both your generated and parsed TLPs conform to the PCIe specification regarding format, field definitions, and timing. Deviations will lead to communication failures.

    5.  **Implement Completion Handling**:
        *   For Memory Read Requests (MRd) and Configuration Read Requests (CfgRd) received from the host, your device is obligated to send back appropriate Completion TLPs (CplD for data, Cpl for no data) within a specific timeframe. The `bar_controller` module (Section 8.2.2) is where this logic for BAR reads resides.

    6.  **Save Changes**:
        *   Save the files (`pcileech_pcie_tlp_a7.sv`, `pcileech_tlps128_bar_controller.sv`, or any custom modules) after implementing the changes.

#### **10.2.2 Handling Different TLP Types**

Each TLP type has a specific header format and behavior. Your firmware must be adept at handling the ones relevant to your donor device.

*   **Memory Read Requests (MRd)**:
    *   **Implementation**:
        *   When an MRd TLP is received (parsed by `pcileech_pcie_tlp_a7.sv` and routed to the `bar_controller`), the `bar_controller` needs to:
            *   Parse the requested address and length.
            *   Fetch the data from the appropriate internal memory location (e.g., BRAM connected to a BAR) or internal register.
            *   Assemble a **Completion with Data (CplD)** TLP. Crucially, this TLP must include the original `Requester ID`, `Tag`, and `Completion ID` (your device's BDF) from the MRd request, along with the fetched data payload.
            *   Send the CplD TLP back to the host via the PCIe IP core's transmit interface.

*   **Memory Write Requests (MWr)**:
    *   **Implementation**:
        *   When an MWr TLP is received, the `bar_controller` needs to:
            *   Parse the target address, length, and `Byte Enables` (FBE/LBE).
            *   Extract the `data payload`.
            *   Write the data to the specified memory location within your emulated device (e.g., a BRAM or internal registers), respecting the byte enables.
        *   Memory Writes are "posted transactions," meaning they don't require a completion TLP for acknowledgment unless an error occurs.

*   **Configuration Read/Write Requests (CfgRd/CfgWr)**:
    *   **Implementation**:
        *   These TLPs target the device's Configuration Space (Vendor ID, Device ID, BARs, Capabilities, etc.). The Xilinx PCIe IP core handles the majority of standard Configuration Space accesses automatically based on its configuration.
        *   However, if you have custom registers or extended capabilities *within* the Configuration Space that are not standard, you might need specific logic to:
            *   For CfgRd: Return the requested data from your internal `cfg_` registers.
            *   For CfgWr: Update your internal `cfg_` registers or trigger actions based on the written data.
        *   Configuration Reads require a **Completion with Data (CplD)**, while Configuration Writes require a **Completion without Data (Cpl)**.

*   **Vendor-Defined Messages (VDM)**:
    *   **Implementation**:
        *   If your donor device uses VDMs, this requires specialized parsing and response logic.
        *   **Parsing Incoming VDMs**: Identify VDMs based on their `Fmt` and `Type` fields. Extract the vendor-specific data and interpret it according to your reverse-engineering findings.
        *   **Crafting Outbound VDMs**: Create logic to assemble VDMs with the precise vendor-specific header and payload formats when your emulated device needs to send them.

#### **10.2.3 Validating TLP Timing and Sequence**

Even if TLPs are perfectly formatted, incorrect timing or sequencing will lead to device malfunction or detection as non-compliant.

*   **Steps**:

    1.  **Use Simulation Tools**:
        *   **Test Benches**: Develop comprehensive SystemVerilog test benches for your TLP generation and parsing modules.
        *   Simulate various scenarios (e.g., host sending MRd, your device sending CplD; host sending MWr; host enumerating device) to validate that TLPs are correctly formed, transmitted, received, and processed.
        *   Verify the sequence of TLPs and that completions are sent within reasonable timeframes.

    2.  **Monitor with ILA (Integrated Logic Analyzer)**:
        *   As detailed in Section 12.1, insert an ILA core into your Vivado design.
        *   Connect the ILA probes to the AXI-Stream interfaces of the PCIe IP core (e.g., `s_axis_rx_tdata`, `s_axis_rx_tvalid`, `m_axis_tx_tdata`, `m_axis_tx_tready`).
        *   Set triggers to capture specific TLPs (e.g., on `m_axis_tx_tvalid` for a certain TLP type).
        *   This allows you to see the actual TLP bits on the FPGA in real time during hardware operation, verifying if your firmware is sending/receiving the correct data and control signals to/from the PCIe IP core.

    3.  **Check Timing Constraints**:
        *   The PCIe IP core has strict timing requirements for its AXI-Stream interfaces. Ensure that your user logic providing data to `m_axis_tx_tdata` and handling `s_axis_rx_tdata` meets these timing constraints.
        *   Vivado's timing analysis reports (after synthesis and implementation) will flag any violations. Address these by optimizing your logic or adjusting clocking where possible.

    4.  **Compliance Testing (Advanced)**:
        *   For high-fidelity emulation, consider using a dedicated PCIe compliance test suite (often integrated with high-end protocol analyzers). These tests systematically check adherence to the PCIe specification, uncovering subtle protocol violations.

    5.  **Save Changes**:
        *   Save all modified files after thorough testing and validation. Iteration is key in TLP-level debugging.

---

## **Part 3: Prove It Works Before You Call It Firmware**

---

## **11. Building, Flashing, and Testing**

After all your customizations are complete, the moment of truth arrives: building the firmware, programming it onto your FPGA, and rigorously testing its functionality to ensure it behaves exactly like the donor device. This phase transitions your design from code to a working hardware emulation.

### **11.1 Synthesis and Implementation**

These are the core steps in the FPGA design flow, where your high-level SystemVerilog code is transformed into a low-level hardware configuration that can be loaded onto the FPGA.

#### **11.1.1 Running Synthesis**

Synthesis is the process by which Vivado translates your HDL code into a gate-level netlist (a description of logical gates and their interconnections). It also performs preliminary timing analysis and resource estimation.

*   **Steps**:
    1.  **Start Synthesis**:
        *   In the Vivado GUI, in the **Flow Navigator** pane (typically on the left), under "Synthesis," click **Run Synthesis**.
    2.  **Monitor Progress**:
        *   Vivado will open a "Launch Runs" dialog. You can typically just click "OK."
        *   Monitor the **Messages** tab at the bottom of the Vivado window. It will show the progress of the synthesis run.
        *   **Common Warnings/Errors to Watch For**:
            *   **`[Synth 8-327]` Unconnected Ports / Unused Inputs**: These indicate that a signal or port in your design is not connected to anything. While sometimes intentional (e.g., unused pins on the FPGA), they can also point to typos in port names or forgotten connections. Investigate each to ensure it's not a functional issue.
            *   **`[Synth 8-256]` Registers/Wires Not Optimized**: This might indicate that logic is being inferred incorrectly or that you have redundant logic that could be optimized.
            *   **Syntax Errors**: If there are fatal syntax errors in your SystemVerilog code, synthesis will fail immediately. Fix these in Visual Studio Code.
    3.  **Review Synthesis Report**:
        *   Upon successful completion, Vivado will ask you what to do next. Select **Open Synthesized Design** or **Open Report**.
        *   Crucially, review the **Utilization Summary** in the synthesis report. This shows how much of the FPGA's resources (LUTs, Flip-Flops, BRAMs, DSPs) your design consumes. Ensure that the design fits within your target FPGA's capacity (e.g., for a Artix-7 35T, you should be well within its limits).

#### **11.1.2 Running Implementation**

Implementation is the most time-consuming step. It takes the synthesized netlist and physically maps it onto the FPGA's resources (placing logic blocks, routing connections) and then performs a detailed timing analysis to ensure the design can operate at the specified clock frequencies.

*   **Steps**:
    1.  **Start Implementation**:
        *   After successful synthesis, in the **Flow Navigator**, under "Implementation," click **Run Implementation**.
        *   Confirm the "Launch Runs" dialog.
    2.  **Monitor Progress**:
        *   Implementation consists of several phases: Opt Design, Power Opt Design, Place Design, Post-Placement Phys Opt Design, Route Design, Post-Route Phys Opt Design. Each phase can take a significant amount of time.
        *   Monitor the **Messages** tab for progress and potential issues.
    3.  **Analyze Timing Reports**:
        *   This is *the most critical step* after implementation. Upon completion, Vivado will again ask what to do next. Select **Open Implemented Design** or, more importantly, **Open Report** and then select **Report Timing Summary**.
        *   **Ensure that all timing constraints are met.** Look for "WNS (Worst Negative Slack)" values.
            *   **Positive WNS**: Indicates that all timing paths meet their requirements (slack means you have extra time). This is what you want.
            *   **Negative WNS**: Indicates **timing violations**, meaning your design cannot operate at the desired clock frequency or data might not be stable. **This is a critical issue that *must* be addressed.**
        *   **Address Violations**:
            *   If you have negative slack, investigate the specific paths that are failing. Vivado's timing report will show you the source, destination, and components of the failing paths.
            *   Solutions can include:
                *   Optimizing your HDL code to reduce logic depth or critical path delays.
                *   Adding pipeline stages (registers) to break up long combinational paths.
                *   Refining your XDC (constraints) file, ensuring all clocks are correctly defined and propagated.
                *   Adjusting clock frequencies (if the application allows).
                *   Using faster timing-closure strategies in Vivado.
                *   Ensuring your custom logic interfaces correctly with the PCIe core's AXI-Stream interface timing requirements.
    4.  **Verify Placement (Optional)**:
        *   In the implemented design, you can open the "Device" view to see how your logic has been placed on the FPGA. This is generally for advanced users to confirm that critical components are placed optimally (e.g., close to PCIe transceivers).

#### **11.1.3 Generating Bitstream**

The bitstream is the final, binary configuration file (`.bit` extension) that will be loaded onto your FPGA. It's the culmination of synthesis and implementation.

*   **Steps**:
    1.  **Generate Bitstream**:
        *   After successful implementation (with no critical timing violations), in the **Flow Navigator**, under "Program and Debug," click **Generate Bitstream**.
    2.  **Wait for Completion**:
        *   This process generally takes less time than implementation but can still vary based on design complexity.
    3.  **Review Bitstream Generation Log**:
        *   Upon completion, Vivado will indicate success. Review the log for any warnings, though typically if implementation passed cleanly, bitstream generation will too.
        *   The `.bit` file will be generated in your project's `pcileech_squirrel_top.runs/impl_1/` directory (or similar path for your board).

### **11.2 Flashing the Bitstream**

Programming (flashing) the bitstream loads your compiled design onto the FPGA, making your emulated device active.

#### **11.2.1 Connecting the FPGA Device**

*   **Steps**:
    1.  **Prepare Hardware**:
        *   Ensure your FPGA-based DMA board is correctly inserted into a compatible PCIe slot on your host system.
        *   Connect the JTAG programmer (e.g., Digilent HS3, Xilinx Platform Cable) to the JTAG header on your FPGA board and to a USB port on your development PC.
        *   Power on the host system.
        *   Refer to your specific FPGA board's manual for precise power, JTAG, and PCIe connection instructions.
    2.  **Open Hardware Manager**:
        *   In Vivado, navigate to **Flow Navigator > Program and Debug > Open Hardware Manager**.
        *   If Vivado is not running, you can launch Hardware Manager as a standalone application.

#### **11.2.2 Programming the FPGA**

*   **Steps**:
    1.  **Connect to the Target**:
        *   In the Hardware Manager window, click **Open Target** (often a large button or link) and select **Auto Connect**.
        *   Vivado should automatically detect your JTAG programmer and then the connected FPGA device(s) on the JTAG chain. If detection fails, check JTAG cable connections, power to the board, and JTAG drivers on your PC.
    2.  **Program Device**:
        *   Once your FPGA device is detected and displayed in the Hardware window, **right-click** on your FPGA device (e.g., `xc7a35t_0`) and select **Program Device**.
        *   A dialog will appear. Click the "..." button next to the "Bitstream file" field and navigate to your generated bitstream file (e.g., `pcileech_squirrel_top.runs/impl_1/pcileech_squirrel_top.bit`).
        *   Click **Program** to begin flashing the firmware onto the FPGA.
        *   Wait for the programming process to complete. You'll see a progress bar.

#### **11.2.3 Verifying Programming**

*   **Steps**:
    1.  **Check Status**:
        *   Ensure the programming completes without errors in Vivado's Hardware Manager. Vivado will display a "Program Device" success message upon completion.
    2.  **Observe LEDs or Indicators**:
        *   Many FPGA boards have status LEDs. A successful programming operation often causes a specific LED to illuminate or change state (e.g., a "DONE" LED). This is a quick visual confirmation.
    3.  **Host System Reboot (Sometimes Required)**:
        *   For the host operating system to correctly recognize the newly programmed PCIe device, a system reboot is often necessary, especially on Windows, to trigger the full PCIe enumeration process.

### **11.3 Testing and Validation**

After programming, the crucial step is to verify that your emulated device is detected correctly by the host and that it functions as expected, mimicking the donor device.

#### **11.3.1 Verifying Device Enumeration**

This confirms that the host OS sees your FPGA as the donor device based on the IDs you programmed.

*   **Windows**:
    *   **Steps**:
        1.  **Open Device Manager**: Press `Win + X` and select **Device Manager** from the Quick Link menu.
        2.  **Check Device Properties**:
            *   Look under the appropriate device category (e.g., **Network Adapters**, **Storage Controllers**, **System devices**).
            *   Find your emulated device. It should now appear with the *name of the donor device* (e.g., "Intel(R) Ethernet Connection...")
            *   Right-click on the device, select **Properties**, and go to the **Details** tab.
            *   In the "Property" dropdown, select "Hardware Ids." Confirm that the **Device ID (DID)** and **Vendor ID (VID)** (e.g., `PCI\VEN_ABCD&DEV_1234`) match those you programmed into your firmware.
            *   Also check "Class Code" and "Subsystem ID" for further verification.
*   **Linux**:
    *   **Steps**:
        1.  **Use `lspci`**: Open a terminal and use the `lspci` command.
            ```bash
            lspci -nn # Shows VendorID:DeviceID
            lspci -vvv # Shows verbose details including BARs, capabilities, and more
            ```
        2.  **Verify Device Listing**:
            *   Check that the emulated device appears in the `lspci` output with the correct Vendor ID, Device ID, and Class Code.
            *   **Example Output (emulating an Intel NIC):**
                ```
                03:00.0 Network controller [0280]: Intel Corporation Ethernet Connection I219-V [8086:1570] (rev 21)
                ```
                (`8086` is Intel's Vendor ID, `1570` is the Device ID for I219-V, `0280` is the Network Controller Class Code).
            *   Use `lspci -vvv` to confirm that the BARs are enumerated with the correct sizes and types, matching your donor device's configuration.

#### **11.3.2 Testing Device Functionality**

Once the device is enumerated, the ultimate test is whether it functions like the original.

*   **Steps**:
    1.  **Install Necessary Drivers**:
        *   If the host OS doesn't automatically load a suitable driver, you will need to manually install the official drivers for your donor device. Download them from the manufacturer's website.
        *   Install them as per the manufacturer's instructions. If the emulation is successful, the driver should install and recognize your FPGA as the real hardware.
    2.  **Perform Functional Tests**:
        *   Run applications or utilities that would typically interact with the donor device.
        *   **Examples**:
            *   **Network Card**: Perform ping tests, browse the web, or initiate large file transfers to test throughput.
            *   **Storage Controller**: Attempt to format a simulated drive (if your emulation includes storage features), perform read/write operations, or run disk benchmarks.
            *   **USB Controller**: Connect USB devices (if your emulation includes USB host functionality) and test their detection and operation.
        *   Monitor the host system for expected behavior and performance characteristics.
    3.  **Monitor System Behavior**:
        *   Check for system stability (no BSODs on Windows, kernel panics on Linux).
        *   Look for device-specific errors in system logs (Event Viewer on Windows, `dmesg` or `journalctl` on Linux).
        *   Ensure that the emulated device behaves as expected under various workloads, including heavy data transfers or stress tests.

#### **11.3.3 Monitoring for Errors**

Proactive error monitoring is crucial for identifying subtle emulation issues that might not cause immediate crashes.

*   **Windows**:
    *   **Steps**:
        1.  **Check Event Viewer**: Press `Win + X` and select **Event Viewer**.
        2.  **Look for PCIe-Related Errors**: Navigate to **Windows Logs > System**. Filter or search for warnings, errors, or critical events related to "PCIe," "PCI Express," or events originating from the specific device driver (look for source names matching your emulated device's driver).
            *   Common errors include resource conflicts, driver initialization failures, or unexpected device responses.
*   **Linux**:
    *   **Steps**:
        1.  **Check `dmesg` Logs**: Open a terminal and type:
            ```bash
            dmesg | grep -i pci # Case-insensitive grep for pci messages
            dmesg | grep -i <VendorID> # Filter for your device's Vendor ID
            ```
        2.  **Identify Issues**: Look for messages indicating problems with PCIe link training, device initialization, memory allocation failures, driver probing errors, or unexpected DMA activity. The Linux kernel's PCIe subsystem is quite verbose.
    *   **Systemd Journal (Modern Linux)**:
        ```bash
        journalctl -b | grep -i pci # Current boot log
        ```

---

## **12. Advanced Debugging Techniques**

When issues arise, especially in complex PCIe device emulation, basic troubleshooting might not suffice. Advanced debugging tools and techniques provide deep visibility into the FPGA's internal logic and the PCIe bus, allowing you to identify and resolve problems efficiently.

### **12.1 The ILA: Vivado's Tiny Black Box Recorder**

The Integrated Logic Analyzer (ILA) is a powerful, configurable debug IP core provided by Xilinx that you can embed directly into your FPGA design. It allows you to monitor the real-time behavior of internal FPGA signals (wires and registers) without needing external probing hardware, functioning like a powerful, internal oscilloscope or logic analyzer.

#### **12.1.1 Inserting ILA Cores**

*   **Steps**:
    1.  **Plan Your Probes**: Identify the key signals you need to observe. For PCIe emulation, these often include:
        *   The AXI-Stream interfaces of the PCIe IP core (e.g., `s_axis_rx_tdata`, `s_axis_rx_tvalid`, `m_axis_tx_tdata`, `m_axis_tx_tready`).
        *   Internal state machine signals (`current_state`, `next_state`).
        *   BAR address decoding outputs (`bar_hit[0]`, `bar_hit[1]`).
        *   Custom register values (`custom_control_reg`, `custom_status_reg`).
        *   Interrupt request signals (`msi_trigger_signal`).
    2.  **Add ILA IP Core**:
        *   In Vivado, open the **IP Catalog** (usually in the **Flow Navigator** pane).
        *   Search for "ILA" (Integrated Logic Analyzer).
        *   Double-click on the "Debug Bridge" (for basic ILA) or "Integrated Logic Analyzer (ILA)" to open its customization GUI.
        *   Configure the ILA:
            *   Set the **Number of Capture Data Ports** (probes) you need.
            *   Set the **Width** of each probe to match the signals you plan to connect.
            *   Configure the **Sample Depth** (how many samples to store before/after a trigger). Larger depths consume more BRAM.
            *   Click "OK" and let Vivado generate the IP.
    3.  **Instantiate and Connect Signals**:
        *   Vivado will generate an `.xci` file for the ILA. You can instantiate it directly in your top-level SystemVerilog file (e.g., `pcileech_squirrel_top.sv`) or within a module where the signals of interest are available.
        *   **Example (in `pcileech_squirrel_top.sv` or a sub-module):**
            ```verilog
            // Assuming you've generated ila_0 from IP Catalog
            // Connect to your design's clock and signals of interest
            ila_0 your_ila_instance (
                .clk(clk_125mhz), // Connect to a stable clock in your design, typically the PCIe user clock
                .probe0(pcie_s_axis_rx_tdata),    // Example: PCIe inbound TLP data
                .probe1(pcie_s_axis_rx_tvalid),   // Example: PCIe inbound TLP valid
                .probe2(pcie_m_axis_tx_tdata),    // Example: PCIe outbound TLP data
                .probe3(my_bar_controller_state), // Example: State of your BAR logic
                .probe4(my_custom_register),      // Example: Value of a custom register
                // Add more probes as needed
                .probeN(signal_to_monitor_N)
            );
            ```
        *   **Alternative (Marking for Debugging):** For simpler signals, you can sometimes mark them directly in your HDL code for debugging. Use `(* mark_debug = "true" *) wire my_signal;` or `(* mark_debug = "true" *) reg my_register;`. Vivado will then automatically suggest adding these to an ILA.

#### **12.1.2 Configuring Trigger Conditions**

ILA is most powerful when you configure intelligent trigger conditions to capture data precisely when an event of interest occurs (e.g., an error, a specific TLP type, a state transition).

*   **Steps**:
    1.  **Generate Bitstream with ILA**: After inserting and connecting the ILA, you must run synthesis, implementation, and generate a new bitstream. The ILA core consumes FPGA resources and will be embedded in your design.
    2.  **Open Hardware Manager**: Program your FPGA with the ILA-enabled bitstream (Section 11.2). Then, in Vivado, open the Hardware Manager and connect to your target.
    3.  **Access the ILA Dashboard**: In the Hardware Manager, select your ILA instance (e.g., `hw_ila_1`). This will open the ILA dashboard.
    4.  **Define Triggers**:
        *   Select the probes you want to use as trigger inputs.
        *   Set specific **trigger patterns** (e.g., `0x4A` for `pcie_s_axis_rx_tdata` to trigger on a Completion TLP).
        *   Configure **trigger conditions** (e.g., "equal to," "not equal to," "rising edge," "falling edge").
        *   Set **trigger positions** (how many samples to capture *before* the trigger event, for pre-trigger visibility).
        *   You can set up multiple trigger sequences for complex event detection.
        *   **Example Scenarios for Triggers**:
            *   Trigger on a specific `Fmt/Type` in a received TLP to analyze incoming commands.
            *   Trigger when a specific register (`my_custom_register`) reaches a certain value.
            *   Trigger on a `pcie_m_axis_tx_tvalid` AND `pcie_m_axis_tx_tdata[3:0]` == `4'hC` (for a Memory Write TLP) to analyze outbound writes.
            *   Trigger on the assertion of an error signal.

#### **12.1.3 Capturing and Analyzing Data**

*   **Steps**:
    1.  **Run the Design**: Allow your host system to interact with the programmed FPGA, causing the events you want to debug.
    2.  **Arm the ILA**: In the ILA dashboard, click the **Run Trigger** button (often a green "Play" icon). The ILA will wait for the defined trigger condition.
    3.  **Capture Data**: Once the trigger condition is met, the ILA will capture a snapshot of the signals into its internal memory buffer.
    4.  **Analyze Waveforms**:
        *   The captured data will appear in the waveform viewer.
        *   Inspect the signal behavior over time. Zoom in, add cursors, and decode values.
        *   Look for:
            *   **Unexpected transitions**: Signals changing at the wrong time.
            *   **Incorrect values**: Registers holding incorrect data.
            *   **Protocol violations**: Your logic sending incorrect data on PCIe interfaces.
            *   **Timing issues**: If signals are not stable when expected (though full timing analysis is done in implementation, ILA shows runtime behavior).
        *   Compare the captured behavior against your expected design logic and the observed behavior of the donor device (if you captured it with a protocol analyzer).

### **12.2 PCIe Traffic Analysis Tools**

While ILA gives you internal FPGA visibility, external PCIe traffic analysis tools provide direct observability of the actual communication on the PCIe bus *between* your emulated device and the host. This is crucial for verifying protocol compliance and debugging link-level issues.

#### **12.2.1 PCIe Protocol Analyzers (Hardware)**

*   **Examples**:
    *   **Teledyne LeCroy PCIe Analyzers**: Gold standard for deep analysis, full protocol decoding, advanced triggering, and error injection capabilities.
    *   **Keysight PCIe Analyzers**: Another leading vendor with similar high-end features.
*   **Steps**:
    1.  **Set Up Analyzer**: Connect the hardware analyzer in-line between the host system's PCIe slot and your FPGA-based DMA device. This typically involves a special interposer card.
    2.  **Configure Capture Settings**: Use the analyzer's software to define what traffic to capture. You can filter by TLP type, address, Requester ID, error conditions, etc., to focus on relevant events.
    3.  **Capture Traffic**: Run your emulated device on the host. The analyzer will passively record all PCIe transactions.
    4.  **Analyze Results**:
        *   Use the analyzer's powerful software to view decoded TLPs, transaction lists, and waveform views.
        *   **Examine TLPs for compliance and correctness**: Are all fields correct? Is the sequence proper?
        *   **Identify any protocol violations or unexpected behaviors**: This is where you find why the driver might be failing (e.g., your device sends a Completion with data when the spec requires a Completion without data, or it responds too slowly).
        *   **Compare with Donor Device Captures**: Directly compare the captured traffic from your emulated device to the captures you made from the real donor device. This is the ultimate test of emulation accuracy.

#### **12.2.2 Software-Based Tools**

For basic PCIe bus inspection, or if a dedicated hardware analyzer is unavailable, some software tools can provide limited insights.

*   **Examples**:
    *   **Wireshark with PCIe Plugins**: While Wireshark is primarily for network traffic, with specialized hardware (e.g., network cards that expose PCIe traces to the OS, or specific capture hardware/drivers), it can sometimes capture and decode PCIe packets. This is highly dependent on the system.
    *   **ChipScope Pro (Legacy Xilinx, now part of Vivado)**: Integrated Logic Analyzer (ILA) is the modern equivalent, but ChipScope was the standalone tool.
    *   **`lspci` (Linux)**: As mentioned in Section 11.3.1, `lspci -vvv` provides extensive static configuration space information. You can combine it with `watch` or scripting to monitor changes over time.
    *   **`pcileech` client (from the PCILeech framework)**: The `pcileech` client software itself can perform read/write operations to memory and configuration space via your FPGA, and can be used to test basic DMA functionality. While not a "traffic analyzer," it's essential for testing the functional interface.
*   **Steps**:
    1.  **Install Necessary Tools/Plugins**: Ensure the tool is installed and any required drivers or plugins are configured.
    2.  **Monitor PCIe Bus**: Run the software tool to capture and display PCIe-related information.
    3.  **Analyze Communications**:
        *   Look for discrepancies in device configuration.
        *   If the tool supports it, analyze the structure of captured packets for anomalies or errors.
        *   Verify that your emulated device is responding to configuration requests correctly.

---

## **13. Troubleshooting: When the Host Says "Unknown Device" and You Question Every Life Choice**

This section provides solutions to common problems you may encounter during custom firmware development, bitstream programming, and hardware testing of your PCIe device emulation. Firmware debugging can be challenging, so a methodical approach is key.

### **13.1 Device Detection Issues**

**Problem**: Your FPGA-based DMA device, after programming, is not recognized by the host system, or it shows up with incorrect IDs (e.g., "Unknown device") or an error symbol in Device Manager/lspci.

#### **Possible Causes and Solutions**:

1.  **Incorrect Device IDs, Vendor IDs, Subsystem IDs, or Class Code**:
    *   **Cause**: The most common reason. There is a mismatch between the identification values programmed into your FPGA firmware and what the host operating system expects, or what you intend to emulate.
    *   **Solution**:
        *   **Verify**: Double-check all `cfg_deviceid`, `cfg_vendorid`, `cfg_subsysid`, `cfg_subsysvendorid`, `cfg_revisionid`, and `cfg_classcode` parameters in `pcileech_pcie_cfg_a7.sv` (or equivalent file) against your meticulously recorded donor device information (from Section 5).
        *   **Consistency**: Ensure these values are also consistently set in the Vivado PCIe IP Core customization GUI (Section 7.2.2).
        *   **Rebuild & Re-flash**: After any changes, always re-synthesize, re-implement, generate a new bitstream, and re-flash the FPGA (Section 11.1, 11.2).
        *   **Reboot Host**: Always reboot the host system after flashing, as Windows often needs a full restart to re-enumerate PCIe devices correctly.

2.  **PCIe Link Training Failure**:
    *   **Cause**: The fundamental PCIe link between the host's root complex and your FPGA card fails to establish. This happens before any configuration space reads. Symptoms include the device not appearing at all (`lspci` shows nothing at that bus/slot, or Device Manager shows a "PCI Express Root Port" error).
    *   **Solution**:
        *   **Physical Connections**: Ensure the FPGA board is seated firmly in the PCIe slot and all power connections are secure. Try a different PCIe slot if available.
        *   **Power**: Verify that the FPGA board is receiving adequate power. Some boards require auxiliary PCIe power connectors.
        *   **Link Speed/Width**:
            *   Check `Max Link Speed` and `Link Width` settings in your Vivado PCIe IP Core (Section 8.1.1).
            *   Try setting the link speed to a lower generation (e.g., Gen1 / 2.5 GT/s) and width to x1, even if your board supports higher. Sometimes, compatibility issues arise with specific motherboards at higher speeds.
            *   Check motherboard BIOS settings for PCIe slot speed options.
        *   **Reset**: Ensure the FPGA's reset logic is correctly implemented (e.g., synchronized to the PCIe reference clock) and asserted/de-asserted correctly upon power-up/reboot.
        *   **PCIe IP Core**: Ensure the PCIe IP core is correctly instantiated and its clocks and resets are properly connected in your top-level design.

3.  **Power Issues (Insufficient or Unstable Power)**:
    *   **Cause**: The FPGA board is not receiving enough stable power, or the power supply is noisy, leading to unreliable operation.
    *   **Solution**:
        *   **Verify Connections**: Double-check all power cables (main PCIe slot power, auxiliary PCIe power, external DC jack if used).
        *   **Power Supply**: Ensure your host system's power supply (PSU) has sufficient wattage and stable 12V rails. For high-power FPGAs, a weak PSU can cause issues.
        *   **External Power**: If the board has an external power jack, ensure it's used with the correct voltage and current rating.

4.  **Firmware Errors (Early Stage)**:
    *   **Cause**: Logic errors in your SystemVerilog code, particularly in the top-level module or the PCIe core's wrapper, that prevent the PCIe core from initializing or presenting itself correctly.
    *   **Solution**:
        *   **Vivado Messages**: Scrutinize Vivado's synthesis and implementation logs for **Critical Warnings** or **Errors** related to the PCIe IP core. These are often indicators of misconfigurations or improper connections.
        *   **ILA Debugging**: If the link attempts to train but fails, use an ILA (Section 12.1) connected to the PCIe IP core's status signals (e.g., `link_up`, `link_speed`, `link_width`) and AXI-Stream interfaces to see at what point the link negotiation fails or if the core is generating unexpected traffic.

### **13.2 Memory Mapping and BAR Configuration Errors**

**Problem**: The emulated device is detected, but when the host OS or a driver tries to access its memory-mapped registers or buffers (via BARs), it crashes, freezes, or reports errors.

#### **Possible Causes and Solutions**:

1.  **Incorrect BAR Sizes or Types (IP Core & Firmware)**:
    *   **Cause**: The BAR sizes or types (32-bit/64-bit, Memory/I/O, Prefetchable/Non-prefetchable) configured in your Vivado PCIe IP Core (Section 7.2.2) and/or handled in your `pcileech_tlps128_bar_controller.sv` do not match what the donor device actually provides. This can cause the host to allocate an incorrect address space or attempt unsupported accesses.
    *   **Solution**:
        *   **Cross-Verify**: Go back to your Arbor/protocol analyzer data (Section 5) and re-verify every single BAR configuration (size, type, prefetchable).
        *   **Consistency**: Ensure these match perfectly in the PCIe IP Core customization and that your `bar_controller` logic correctly handles the size (address decoding range) and type of each BAR.
         *   **BRAM Sizing**: If your BARs map to BRAMs, confirm the BRAM backs only the implemented aperture, not the full advertised BAR size. See Section 8.2 for the warehouse metaphor.

2.  **Address Decoding Errors in Firmware**:
    *   **Cause**: Your `pcileech_tlps128_bar_controller.sv` (or custom BAR logic) is misinterpreting the incoming PCIe addresses, leading to accesses to incorrect internal registers or memory locations.
    *   **Solution**:
        *   **Review Logic**: Meticulously review the `case` statements and address calculations in your `bar_controller`.
        *   **Simulation**: Develop specific test cases in your SystemVerilog test bench to simulate host read/write accesses to various offsets within each BAR. Verify that the internal `bar_hit` signals are correct and that the data is routed to/from the correct internal registers/BRAMs.
        *   **ILA Debugging**: Place ILA probes on the `req_addr`, `req_write`, `req_read`, `req_data`, `rsp_data`, and the internal signals related to your address decoding and register access within the `bar_controller`. Observe how the address is decoded and what data is being read/written in real time.

3.  **Overlapping Address Spaces (Internal)**:
    *   **Cause**: While the PCIe standard ensures that BARs from different devices don't overlap in the host's memory map, *internally* within your FPGA, you might accidentally map different logical components to the same physical address space within a single BAR.
    *   **Solution**:
        *   **Map Carefully**: When defining internal registers and memory blocks within a BAR, explicitly assign unique, non-overlapping offsets to each. Use `localparam` for these offsets to prevent errors.
        *   **Design Review**: A thorough design review of your `bar_controller` is necessary to ensure every address range is uniquely handled.

4.  **BRAM Access Issues**:
    *   **Cause**: Problems with interfacing your logic to the BRAM IP cores (e.g., incorrect BRAM clocking, asynchronous resets, wrong byte enables, or incorrect write enable logic).
    *   **Solution**:
        *   **BRAM Documentation**: Consult the Xilinx BRAM IP core documentation for correct instantiation and interface signals.
        *   **ILA**: Place ILA probes on the BRAM interface signals (address, write enable, data in, data out) to verify that your logic is sending the correct control signals to the BRAM.

### **13.3 DMA Performance and TLP Errors**

**Problem**: The device is detected and functionally appears to work, but data transfer rates are slow, or the system experiences intermittent crashes, hangs, or errors during large DMA operations. PCIe protocol analyzers report malformed TLPs or flow control issues.

#### **Possible Causes and Solutions**:

1.  **Malformed TLPs (Header/Payload)**:
    *   **Cause**: Your firmware is generating TLPs (especially Completions or outbound Memory Writes if your FPGA is acting as a DMA master) with incorrect headers, lengths, byte enables, or payloads. The host system's PCIe core or driver detects these as violations.
    *   **Solution**:
        *   **PCIe Protocol Analyzer**: This is the best tool here (Section 12.2.1). Capture traffic and meticulously compare your generated TLPs against the PCIe specification and, more importantly, against captures from your *real donor device*.
        *   **TLP Generation Logic**: Review your TLP assembly code (`pcileech_pcie_tlp_a7.sv` and related modules). Ensure all fields (Fmt, Type, Requester ID, Tag, Completion ID, Length, Byte Enables, Address) are correctly derived and packed into the TLP structure.
        *   **Error Checking**: Implement basic error checking in your firmware (e.g., checking for unexpected `req_valid` without `req_ready` or vice-versa).

2.  **Flow Control Issues**:
    *   **Cause**: PCIe uses a credit-based flow control mechanism. If your firmware (or the PCIe IP core's interaction with it) incorrectly manages credits, it can lead to deadlocks, timeouts, or dropped packets. Symptoms include a "stalled" PCIe link, timeouts, or low throughput.
    *   **Solution**:
        *   **PCIe IP Core Configuration**: PCIe credit-based flow control is managed by the link/hard IP layers. Your logic's job is to obey the user-stream handshake signals (`tvalid`/`tready`) and not overflow its own internal queues. Do not attempt to tune PCIe credit behavior from the IP GUI like a NIC ring buffer -- the defaults are correct for this class of endpoint.
        *   **User Logic Backpressure**: Your user logic that sends TLPs to the PCIe IP core (`m_axis_tx_*` interface) *must* respect the `m_axis_tx_tready` signal from the IP core. If `tready` is de-asserted, you *must* pause sending data. Failing to do so will overflow the core's buffers.
        *   **ILA Debugging**: Connect ILA probes to the flow control interface signals of the PCIe IP core and your user logic to observe if `tvalid`/`tready` handshake is working correctly.

3.  **Inefficient DMA Logic / Buffering Issues**:
    *   **Cause**: Your DMA engine implementation within the FPGA (the part that reads/writes data to/from host memory) is not optimized, causing bottlenecks. This can involve:
        *   Lack of pipelining.
        *   Inefficient use of BRAMs.
        *   Stalls due to external memory access latency.
        *   Small burst sizes.
    *   **Solution**:
        *   **Pipelining**: Break down long combinational paths into smaller, sequential stages using registers. This allows higher clock frequencies and better throughput.
        *   **Buffering**: Use FIFOs (First-In, First-Out buffers) to decouple sender and receiver logic, smoothing out data flow and preventing stalls.
        *   **Burst Transfers**: Utilize PCIe's ability to perform burst reads/writes for efficiency. Ensure your DMA logic requests and handles data in appropriate burst sizes.
        *   **Memory Bandwidth**: Ensure your BRAMs or external DDR memory interfaces are capable of supplying/consuming data fast enough for your desired DMA rates.
        *   **ILA**: Monitor your DMA engine's internal state, read/write pointers, and data path signals to identify bottlenecks.

4.  **Completion Timeout / Unsupported Request**:
    *   **Cause**: The host sends a request (e.g., MRd, CfgRd), but your FPGA device does not respond with a Completion TLP within the allowed timeout period, or it responds with an error status (e.g., Completion with Unsupported Request (UR) or Completer Abort (CA)).
    *   **Solution**:
        *   **Response Logic**: Verify that your `bar_controller` (for MRds) and `pcileech_pcie_cfg_a7.sv` (for CfgRds to custom config space) correctly identify the request and generate the appropriate Completion.
        *   **Timeout Value**: Review your donor device's expected completion latency. While PCIe defines default timeouts, some drivers might be sensitive.
        *   **ILA/Protocol Analyzer**: Crucial for pinpointing *why* a completion isn't sent or why it's malformed. Is the request TLP even reaching your user logic? Is your logic generating a response? Is the PCIe core successfully sending the response?

---

## **14. Timing, CDC, and Other Ways to Lose Your Weekend**

Achieving truly convincing emulation means making your FPGA-based device indistinguishable from the donor, not just in its ID, but in its behavior. This requires meticulous attention to timing, responsiveness, and subtle operational details.

### **14.1 Techniques for Accurate Timing Emulation**

Precise timing is paramount in hardware, especially for high-speed interfaces like PCIe. Mismatches can lead to driver timeouts, incorrect data interpretation, or system instability.

*   **Implement Timing Constraints (XDC Files)**:
    *   **Purpose**: Timing constraints are instructions to Vivado's synthesis and implementation tools, telling them how fast your design needs to operate. They define clock periods, input/output delays, and path delays.
    *   **Usage**: The PCILeech-FPGA project includes XDC files (e.g., `pcileech_squirrel_top.xdc`) that define the main clocks (e.g., `create_clock -name sys_clk_p -period 8.0 [get_ports sys_clk_p]`).
    *   **Refinement**: If your emulation requires very specific internal timing or reacts to time-sensitive commands, you might need to add further constraints to critical paths (`set_max_delay`, `set_input_delay`, `set_output_delay`) within your custom logic.
    *   **Goal**: Ensure Vivado reports **positive WNS (Worst Negative Slack)** for all paths after implementation, indicating the design meets its timing requirements.

*   **Use Clock Domain Crossing (CDC) Techniques**:
    *   **Purpose**: PCIe designs often involve multiple clock domains (e.g., a 125MHz PCIe user clock, a separate clock for your custom logic). Moving signals between these domains asynchronously (without proper synchronization) can lead to **metastability**, causing unreliable behavior.
    *   **Implementation**: Always use proper CDC circuits for signals crossing clock domains:
        *   **Two-Flip-Flop Synchronizers**: For single bit control signals.
        *   **Asynchronous FIFOs (First-In, First-Out)**: For multi-bit data paths, providing buffering and flow control between clock domains.
        *   **Gray Code Encoders/Decoders**: For counters or addresses crossing domains to ensure only one bit changes at a time.
    *   **Vivado Tools**: Vivado includes CDC analysis tools (e.g., `report_cdc`) that can identify potential metastability issues.

*   **Simulate Device Behavior with Time-Accurate Models**:
    *   **Advanced Test Benches**: Use SystemVerilog test benches that incorporate realistic timing delays or even provide time-accurate PCIe bus functional models (BFMs).
    *   **Verification**: This allows you to observe how your emulated device's internal state and external TLP generation/response timings behave under various conditions, ensuring they match your captured donor device behavior.

### **14.2 Dynamic Response to System Calls**

A truly accurate emulation doesn't just present the correct IDs; it also reacts intelligently and dynamically to the host system's commands and queries, mimicking the behavior of a real, active device.

*   **Implement State Machines for Device Control**:
    *   **Purpose**: Design robust SystemVerilog state machines that manage the device's operational modes, command processing, and data flow.
    *   **Responsiveness**: Ensure the state machine transitions logically and quickly in response to incoming commands (e.g., writes to control registers in a BAR, specific TLPs).
    *   **Graceful Handling**: The state machine should be able to handle unexpected or out-of-order requests gracefully, perhaps returning an error TLP or simply ignoring invalid commands, rather than crashing or freezing.

*   **Monitor and Respond to Host Commands (Beyond Simple Reads/Writes)**:
    *   **Configuration Writes**: Beyond initial enumeration, drivers often write to configuration space registers to enable features, set thresholds, or clear status bits. Your firmware must process these writes and update internal state accordingly.
    *   **Vendor-Specific Commands**: As discussed in Section 9.2, if the donor device has proprietary commands (accessed via custom registers or Vendor Defined Messages), your firmware must parse these commands and trigger the appropriate emulated behavior.
    *   **Power Management Commands**: React to host-initiated power state transitions (D0, D1, D3hot, etc.) by enabling/disabling internal logic and acknowledging the state change.
    *   **Interrupt Acknowledgment**: If the host driver acknowledges interrupts by writing to a specific register, ensure your firmware can detect this and clear the internal interrupt request.

*   **Optimize Firmware Logic for Responsiveness**:
    *   **Reduce Latency**: Critical data paths and control paths should be optimized to minimize combinational logic depth and pipeline stalls.
    *   **Parallelism**: Leverage the FPGA's inherent parallelism to perform multiple operations concurrently, improving throughput and response times.
    *   **Efficient Memory Access**: Optimize access to internal BRAMs or external DDR memory to ensure data is available when needed for DMA transfers or register reads.
    *   **Hardware Acceleration**: For complex computations or data manipulations that the donor device performs, consider implementing dedicated hardware accelerators on the FPGA rather than trying to perform them in a slow, software-like fashion.

---

## **15. Best Practices for Firmware Development**

Adhering to best practices in custom firmware development is crucial for maintaining code quality, facilitating collaboration (if working in a team), simplifying debugging, and ensuring the long-term maintainability and reliability of your project. This is especially true for security-sensitive applications.

### **15.1 Continuous Testing and Documentation**

*   **Regular, Incremental Testing**:
    *   **Unit Testing**: Test small, individual modules (e.g., a TLP parser, a register block) in isolation using dedicated test benches.
    *   **Integration Testing**: Verify that different modules work together correctly.
    *   **System Testing**: After flashing, perform end-to-end tests with the host system to ensure overall functionality.
    *   **Test Early, Test Often**: Test the firmware after each significant change, no matter how small, to catch issues early when they are easier to debug.

*   **Automated Testing (Advanced)**:
    *   For complex projects, implement automated test scripts (e.g., using Python with a hardware abstraction layer) on the host side to repeatedly verify functionality and performance.
    *   Consider integrating with Continuous Integration (CI) tools (e.g., Jenkins, GitLab CI) in a team environment to automate builds, tests, and static analysis on every code commit.

*   **Maintain Comprehensive Documentation**:
    *   **Design Documents**: Create and update documents that describe your firmware's architecture, including:
        *   **Block Diagrams**: Illustrating the major modules and their interconnections.
        *   **State Machine Diagrams**: For all stateful logic.
        *   **Interface Specifications**: Detailing input/output signals, timing, and protocols between modules.
        *   **Memory Maps**: For all BARs, defining register addresses, bit fields, and their functionality.
    *   **Code Comments**: Use clear, concise comments within your SystemVerilog code to explain complex logic, purpose of signals, and any non-obvious design choices.
    *   **Change Log/Commit Messages**: Maintain a change log or use detailed Git commit messages to track all modifications, bug fixes, and feature additions, explaining *why* changes were made.
    *   **User Guide**: For your custom firmware, a simple user guide explaining how to build, flash, and interact with the emulated device from the host side is invaluable.

### **15.2 Managing Firmware Versioning**

Proper version control is essential for tracking changes, collaborating effectively, and managing releases.

*   **Use Version Control Systems (VCS)**:
    *   **Git**: Strongly recommended. Use Git to manage your HDL source code, constraints files, and project scripts.
    *   **Organize Repository**: Maintain a clear directory structure (e.g., separate folders for `src`, `xdc`, `ip`, `scripts`, `doc`).
    *   **Branches**: Use feature branches for developing new capabilities or large changes. Merge back to a `main` or `develop` branch after thorough testing.
    *   **Regular Commits**: Commit frequently with atomic, meaningful commit messages.

*   **Tag Releases and Milestones**:
    *   **Stable Versions**: Use Git tags (e.g., `v1.0.0`, `v1.0.1_bugfix`) to mark stable, tested versions of your firmware. This makes it easy to revert or deploy a known good state.
    *   **Milestones**: Tag significant development milestones (e.g., "Basic Enumeration Working," "DMA Read/Write Functional").

*   **Backup and Recovery Strategy**:
    *   **Cloud-Based Repositories**: Host your Git repository on platforms like GitHub, GitLab, or Bitbucket. This provides off-site backups and facilitates collaboration.
    *   **Local Backups**: Even with cloud repositories, maintain regular local backups of your entire Vivado project directory (which can be very large due to generated files).

### **15.3 Security Considerations**

If you think "security" is a checklist item you can skip, you're already the kind of person who gets your firmware leaked on UC within 48 hours. DMA is the most powerful hardware primitive available to you -- it reads any physical address, it writes any physical address, it sits below the OS, below the hypervisor, below everything except the silicon itself. That power doesn't come with training wheels.

* Limit your DMA reads to exactly the addresses you need. Honeypots are real. Vanguard seeded decoy regions in June 2025 and watched thousands of DMA cards read them. Those people are banned. Don't be those people.
* If you're writing to memory, you're already making noise. Read-only DMA is quieter, but nothing is silent. IOMMU logs are active since August 2025. Your reads have a pattern. Real hardware has chaotic access patterns. A DMA card polling 0x1000 bytes every 16ms is a metronome. Metronomes are detectable.
* EFUSE DNA-lock your firmware. If you don't, someone WILL extract it, slap their own VID/DID on it, and sell it for $300 on Discord. You spent 60 hours in Vivado. They spent 60 seconds with OpenOCD. Lock. Your. Firmware.
* Test on a separate machine. Not a VM. Not your gaming PC. A completely separate box that has never logged into your main accounts. If you get HWID banned, it's permanent. Riot derives HWID from TPM 2.0 -- you're not spoofing that. You're buying a new motherboard.
* The pre-boot DMA CVEs (2025-11901 series) are fascinating but don't bet on them. Motherboard vendors patched those. If your target hasn't updated firmware since 2025, they have bigger problems than your DMA card.
    *   **Monitoring and Logging**: While difficult at the hardware level, unusual DMA activity or enumeration of unknown PCIe devices should trigger alerts in security monitoring systems.

*   **Secure Coding Practices for Firmware**:
    *   **Input Validation**: If your firmware accepts any inputs (e.g., via a UART debug interface, or internal registers written by the host), validate them rigorously to prevent buffer overflows, integer overflows, or unexpected behavior.
    *   **Least Privilege**: Design your firmware logic to only perform the operations absolutely necessary for its function. Avoid granting unnecessary capabilities.
    *   **State Management**: Implement robust state machines to prevent unintended behavior due to invalid state transitions.
---

## **16. Additional Resources**

To deepen your understanding and stay updated in the dynamic fields of FPGA development, PCIe, and hardware security, consult the following resources:

*   **Xilinx (AMD) Documentation**: Your primary source for all things Vivado and Xilinx FPGAs.
    *   **Main Documentation Portal**: [https://docs.amd.com/](https://docs.amd.com/) (formerly Xilinx.com/support/documentation).
    *   **Vivado Design Suite User Guides**:
        *   **UG900 - Getting Started**: Essential for new Vivado users.
        *   **UG901 - Logic Synthesis**: Deep dive into synthesis.
        *   **UG904 - Implementation**: Detailed guide on placement and routing.
        *   **UG912 - Tcl Command Reference Guide**: Invaluable for scripting.
        *   **UG939 - Debugging**: Comprehensive guide to ILA and other debug features.
    *   **PCI Express IP Core User Guide**: Critically important for understanding the Xilinx PCIe IP (e.g., **PG054 for 7 Series Integrated Block for PCI Express**). Search for "PCI Express" on the documentation portal. This details the core's configuration, interfaces, and limitations.

*   **PCI-SIG Specifications**: The definitive source for the PCIe standard.
    *   **PCI Express Base Specification**: The foundational document. While not publicly free, summaries and educational materials based on it are widely available. You can usually find information on their website: [https://pcisig.com/specifications](https://pcisig.com/specifications) (Note: Full specifications typically require PCI-SIG membership).

*   **FPGA Tutorials and Learning Platforms**:
    *   **FPGA4Fun**: [http://www.fpga4fun.com/](http://www.fpga4fun.com/) - A classic site with many practical FPGA projects and tutorials.
    *   **Verilog/VHDL Tutorials**:
        *   **ASIC World Verilog Tutorials**: [https://www.asic-world.com/verilog/index.html](https://www.asic-world.com/verilog/index.html) - Good fundamental Verilog reference.
        *   **VHDLwhiz**: [https://www.vhdlwhiz.com/](https://www.vhdlwhiz.com/) - VHDL reference and tutorials.
    *   **Stack Overflow (FPGA/Verilog/PCIe tags)**: [https://stackoverflow.com/questions/tagged/fpga](https://stackoverflow.com/questions/tagged/fpga) - Community-driven Q&A for specific technical problems.

*   **PCIe Protocol Analysis Tools**:
    *   **Teledyne LeCroy Protocol Analyzers**: [https://teledynelecroy.com/protocolanalyzer/](https://teledynelecroy.com/protocolanalyzer/) - Explore their range of high-performance PCIe analyzers and software.
    *   **Telescan PE Software**: [https://www.teledynelecroy.com/protocolanalyzer/pci-express/telescan-pe-software/resources/analysis-software](https://www.teledynelecroy.com/protocolanalyzer/pci-express/telescan-pe-software/resources/analysis-software) - A no-cost software tool that provides some PCIe analysis features (requires registration).

*   **PCILeech Community & Resources**:
    *   The `ufrisk/pcileech` GitHub repository is the core. Actively follow its updates and issues.
    *   Look for community forums or Discord servers dedicated to PCILeech or similar open-source DMA projects.

*   **Hardware Security & Reverse Engineering**:
    *   Books on hardware hacking, reverse engineering, and low-level system exploitation.
    *   Conferences like Black Hat, DEF CON, Recon, and Troopers often feature talks on PCIe and DMA attacks.
    *   Blogs and research papers from security researchers focusing on hardware.

---

## **17. Contact Information**

If you need assistance, have questions, or wish to collaborate on topics related to this guide, firmware development, or hardware security, please feel free to reach out. I'm available to provide guidance, troubleshoot complex problems, or discuss ideas in detail.

### **Discord**:
*   **User**: [**VCPU**](https://discord.com/users/196741541094621184)
*   **Server Invite Link**: [**Join the Hardware Hacking & Firmware Development Discord**](https://discord.gg/dS2gDUDQmV)

---

## **18. Support and Contributions**

Your support helps maintain and improve this guide and related projects. Creating and updating comprehensive technical documentation and open-source hardware projects requires significant time and effort.

### **Donations**

If you found this guide helpful and want to support ongoing work, consider contributing. Every donation, no matter how small, helps in continuing to create, share, and support the community through further research, development, and documentation efforts.

*   **Crypto Donations (LTC - Litecoin)**:
    *   **Address**: `MPMyQD5zgy2b2CpDn1C1KZ31KmHpT7AwRi`

**Special Bonus**: If you donate, please feel free to reach out on Discord (VCPU) to receive a personal thank you and possibly additional resources, early access to new content, or personalized assistance with your project.

**Note**: If you need me to review specific sections of your implementation, troubleshoot issues, or provide detailed feedback on your code, please mark the relevant sections within your code with `//VCPU-REVIEW//` comments and provide detailed explanations of the problems or questions you're encountering. This helps me focus my efforts and provide the most effective support.

May God bless your soul.

---

## **19. DMA Advances & State of the Art (2024-2026)**

This section consolidates the most relevant DMA/PCIe developments through 2026.

### **19.1 Hardware: what ships now**

*   **Artix-7 (this guide's scope)** remains the workhorse (Squirrel 35T, Enigma-X1 75T, ZDMA 100T) and is still what most open PCILeech firmware targets.
*   **CaptainDMA** line (35T / 75T / 100T / 4.1th / **M.2** / **M.2 100T**) is now officially supported in `ufrisk/pcileech-fpga` and flashes via **WCH CH347 / CH347_OpenOCD** rather than a separate JTAG dongle -- a major usability improvement. The M.2 100T variant pushes ~220 MB/s over USB-C.
*   **IMMORTAL DMA** (XC7A35T, USB-C) -- compact Artix-7 card achieving ~275 MB/s, emerging as a popular alternative to Squirrel/Screamer boards.
*   **M.2 / NGFF form factor** is the current stealth direction (CaptainDMA M.2, CaptainDMA M.2 100T, generic M.2-35T): far smaller than a full-height card, mounts via an M.2 slot or M.2→PCIe cable.
*   **MITM passthrough devices** sit *between* a real donor device and the host, passing its traffic through. Because genuine device behavior is preserved by construction, they defeat enumeration- and CFG-space-based detection that pure FPGA emulation struggles with -- at the cost of needing a real donor on the bench. Don't pay anyone for these. Anyone selling interposer boards is a scammer. Build it yourself.

### **19.2 Beyond 7-series: Gen3/Gen4/Gen5**

The 7-series PCIe hard block tops out at **Gen2**. If throughput or link width matters, move to:

*   **UltraScale+ (Kintex/Virtex UltraScale+)** -- Gen3/Gen4 integrated block (PCIe4C, PG213). Xilinx/Vivado IP docs: PG156 (US Gen3), PG213 (US+ Gen4), PG195 (XDMA), PG302 (QDMA), PG344 (Versal).
*   **Versal ACAP (Prime/Premium)** -- Gen4/Gen5 DMA & Bridge Subsystem (PG344).
*   **Wupper** (Nikhef / CERN ATLAS FELIX, GitLab) -- a mature, open **PCIe Gen3/Gen4/Gen5 DMA engine** ported to Virtex-7, Kintex UltraScale, Virtex UltraScale+, and Versal. It uses descriptor-based DMA (FIFOs + MSI-X interrupt controller) and Tcl-script project generation. A strong base when the PCILeech 7-series pipeline is the bottleneck.

### **19.3 Software / tooling refreshers**

*   **PCILeech** is at **v4.19 (Jan 2025)**; pairs with **MemProcFS v5.0+** and **LeechCore v2.21** (gRPC LeechAgent). KMD signatures include `WIN11_X64`.
*   **PCIeem** -- a Linux kernel framework for PCIe device emulation *in userspace* (virtual BARs backed by reserved RAM). Excellent for driver development and rapid protocol iteration without reflashing an FPGA.
*   **Automated firmware generation** (replaces much of the manual `.coe` workflow): live VFIO donor dumps to shadow `.coe` plus writemasks, dynamic capability generation, and full bitstream output. Config space `.coe` and writemask `.coe` generation from `lspci -xxxx` dumps or Telescan exports is standard. A ready-to-use converter is available (`tools/config_space_to_coe.py`).

### **19.4 Emulation fidelity: DUMP EMU → FULL EMU**

Modern detection punishes static "dump and replay" firmware. Current best practice:

*   **Shadow config space** -- emulate the *readable* view from a real donor dump while keeping internal state separate. The host accesses configuration space through Configuration Read/Write TLPs generated by the root complex. ECAM (Enhanced Configuration Access Mechanism) is the host CPU's memory-mapped addressing scheme -- the root complex translates ECAM transactions into standard Configuration Requests. A downstream endpoint receives only Configuration TLPs; it cannot distinguish whether the request originated from ECAM, firmware, or another software path. Shadow-config detection by host software requires comparing config reads through multiple software mechanisms (e.g., kernel driver vs. userspace MMIO), not comparing two different hardware interfaces at the endpoint.
*   **Writemasks / RW1C** -- implement correct Read-Only vs Read-Write-1-to-Clear behavior. A real device has registers that are read-only, registers that are read-write, and registers with RW1C semantics (writing a 1 clears the bit, writing a 0 has no effect). Anti-cheat drivers write patterns to config space and read them back. If your shadow config returns the written value on a read-only register, or fails to clear an RW1C bit, you are flagged.
*   **Donor spoofing** -- AX200/AX210 WiFi, Realtek/Intel NICs, audio codecs, NVMe; pair with **behavioral** responses (e.g., genuine WiFi beacon/packet behavior) because VGK/EAC now send *fake packets* and A/B probe device classes.
*   **VMD trick** -- present an NVMe behind Intel VMD (Volume Management Device) so the OS/anti-cheat only enumerates the VMD controller; detected by VGK/ACE in 2025, but instructive.
*   **EFUSE DNA locking / timed self-reset** -- a device DNA check on boot can verify the FPGA is the authorized unit, preventing firmware from being copied to another board. DNA is an identifier, not an encryption key. It provides anti-cloning (bind firmware to one chip) but does not provide confidentiality (bitstream can still be read from flash) or integrity (bitstream can be modified unless also signed/encrypted). Xilinx 7-series supports AES bitstream encryption with battery-backed key storage for confidentiality, and HMAC authentication for integrity. DNA alone is the weakest link in that chain. If firmware IP protection matters to you, implement the full Xilinx security model, not just a DNA comparison in user logic.

#### **Shadow config .coe format: from donor dump to Vivado**

The PCILeech pipeline feeds configuration space into the PCIe core via `.coe` (Coefficient) files  --  Xilinx's format for initializing Block RAM contents. A `.coe` file is a text file with a radix declaration and a comma-separated list of hex values. Vivado reads it during IP generation and bakes it into the BRAM initial contents.

**Scope: conventional configuration space only (offsets 0x00-0xFF, 256 bytes = 64 x 32-bit words).** This covers VID/DID, class code, BARs, conventional capabilities (PM, MSI/MSI-X, PCIe capability), and the capability pointer chain up to offset 0xFF. Extended capabilities (DSN, AER, etc.) begin at offset 0x100 and require a 4 KB config space model with 1024 DWORDs. The standard 64-DWORD `.coe` workflow does not implement extended capabilities. If you need DSN or AER emulation, you need a full 4 KB config-space BRAM, an extended-capability linked-list implementation in your config handler, and separate read/write masks for the extended region.

**`.coe` file structure for config space (256 bytes = 64 x 32-bit words):**

```
memory_initialization_radix=16;
memory_initialization_vector=
00000000,
00000000,
...
00000000;
```

Each 32-bit word corresponds to one DWORD offset in config space. Word 0 = offsets 0x00-0x03 (VID/DID). Word 1 = offsets 0x04-0x07 (Command/Status). Word 16 = offsets 0x40-0x43 (first capability structure). And so on through word 63 = offsets 0xFC-0xFF.

**Workflow:**

1. Dump donor config space with `lspci -xxxx` (Linux, raw hex dump of 4096 bytes for extended space) or Arbor export
2. Convert to `.coe` format: extract the first 256 bytes (64 DWORDs) into the vector list above. A `telescan_to_coe.py` or equivalent converter automates this. The `tools/config_space_to_coe.py` script in this repo handles it.
3. Place the `.coe` file in your board's `ip/` directory
4. In Vivado, the BRAM IP core (`bram_pcie_cfgspace.xci`) references this `.coe` as its init file
5. **Writemask**: a separate `.coe` file defines which bits are writable. For each 32-bit word in config space, the writemask uses a corresponding 32-bit mask where a 1 bit means "writable" and a 0 bit means "read-only." The `bar_controller` or `cfg` module checks this mask on every config write and rejects writes to read-only bits

**Writemask example for offset 0x04 (Command register):**

```
; Command register: bits 0 (IO Space), 1 (Memory Space), 2 (Bus Master) are writable
; bits 3-15 are reserved/read-only or controlled by other mechanisms
; Word 1 (offset 0x04): mask = 0x00000007
```

The donor's datasheet or a real device tested with config writes tells you which bits are actually writable. You cannot guess this from a static dump. A dump only tells you the current value, not what happens when you write to it. This is why writemask correctness requires either a datasheet, a second device you can probe, or inference from driver behavior.

**The full writemask is 64 x 32-bit masks matching the 64-word config space layout.** Each mask word corresponds to the same offset as the data `.coe`. The PCILeech `cfg` module uses both files: the data `.coe` initializes the read values, the writemask `.coe` gates writes.

### **19.5 The detection arms race -- or: Vanguard's 2025 DMA genocide, month by month**

Stock PCILeech firmware is dead. Not "dying." Dead. Vanguard loads at `SERVICE_BOOT_START` -- not at game launch, at BOOT. vgk.sys is in your kernel before you see the Windows login screen. You think you're clever changing VID/DID? They were past that in 2019. Here's the murder calendar:

| Date | Milestone | What Actually Happened |
|------|-----------|----------------------|
| **Jan 2025** | XHCI controller checks | Community reports indicate Vanguard began verifying XHCI capability registers, extended caps, and doorbell behavior on USB controller class devices. |
| **Mar 2025** | WiFi adapter checks | Community reports indicate Vanguard extended device-class validation to WiFi adapters. A spoofed AX200/AX210 cannot respond to MAC-level probe frames because an FPGA has no radio. |
| **Apr 2025** | Audio device checks | Community reports indicate Vanguard began validating audio codec register responses and jack detection state on HD Audio class devices. |
| **May 2025** | NVMe VMD enumeration | Community reports indicate Vanguard now enumerates through Intel VMD (Volume Management Device) to detect NVMe controllers hidden behind it. |
| **Jun 2025** | Honeypot memory regions deployed | Decoy data structures seeded in game memory. Read one, you're flagged. Silent. No error. No crash. Just a ban 2 weeks later while you're wondering what went wrong. |
| **Jul 2025** | Thunderbolt 4 scanning | They enumerate Thunderbolt-connected devices. PCIe over TB4 -- they're watching that bus too. |
| **Aug 2025** | IOMMU fault monitoring | Community reports indicate anti-cheat kernels observe IOMMU/DMA-remapping fault events. A device generating repeated translation faults exhibits a pattern distinct from legitimate hardware. Your DMA card reading player positions on a fixed interval isn't generating faults per se -- but the access pattern can be distinguished from genuine device traffic. |
| **Dec 2025** | UEFI firmware enforcement | Game won't launch on unpatched ASUS/Gigabyte/MSI/ASRock boards. VAN:Restriction error. The "Sleeping Bouncer" CVEs broke the trust model. |
| **Feb 2026** | EAC IOMMU mandatory (Fortnite) | Epic followed Riot. Competitive Fortnite now requires IOMMU. DMA in protected memory space = logged fault. |
| **May 2026** | Vanguard DMA hardware interference | Riot publicly claimed Vanguard can interfere with unauthorized DMA hardware on the bus. The exact mechanism is not documented -- the IOMMU constrains DMA address translation and does not itself grant the host a generic reflash capability. Any host-side interference would require the endpoint to expose a reachable vendor-specific control path. Riot posted a "$6000 paperweight" image and it went viral. |
| **Jun 2026** | Vanguard On-Demand | First kernel AC to become optional -- but only if your hardware trust chain is clean. TPM 2.0 + Secure Boot + patched firmware. Miss any of these and Vanguard stays mandatory. They're not making it easier; they're making the *incentive* different. You're not trying to bypass the AC -- you're trying to convince it your platform is clean. |

**Detection techniques (labeled by evidence tier; [C]=confirmed, [I]=inferred from observable behavior, [S]=speculative/community report):**

- **[I] Bus Master + no driver** -- some anti-cheat kernels reportedly check whether a device has Bus Master enabled in `PCI_COMMAND` (offset 0x04, bit 2) without a matching driver in the device stack. Bus mastering is normal for many endpoints and a device can be bus-master capable before a vendor driver loads; this is a heuristic, not a smoking gun.
- **[S] MSI interrupt silence** -- if your device has MSI enabled but never generates interrupts, that deviates from the behavior of real NICs and XHCI controllers. Whether any specific product uses this as a broad detection fingerprint is unconfirmed.
- **[C] TPM 2.0 hardware-backed HWID** -- Vanguard derives hardware IDs from the TPM on Windows 11. Ban evasion requires TPM replacement or attestation forgery.
- **[S] Config space signature scanning** -- community reports describe signature matching at offsets associated with capability pointers (0x40, 0x60). Specific offsets and signature content are not publicly documented. PCIe capability layouts are device-class-specific; fixed offsets are not a universal signature.
- **[I] Shadow-config-space detection** -- host software can compare config space reads through different access paths (kernel driver config read vs. userspace MMIO). Whether a specific product does so at a given version is unconfirmed without driver analysis.
- **[S] Interrupt line assertion check** -- a device whose interrupt line has never been asserted since enumeration may be distinguishable from real hardware. Implementation evidence for specific products is not publicly available.
- **~60,000 accounts banned per week** in Valorant (per Riot's public statements). Vanguard employs ex-cheat developers.

**What to know (rather than what to cargo-cult):**
- Changing VID/DID alone is not emulation. Configuration-space identity influences enumeration and driver binding. It does not confer functional compatibility -- a driver expects register semantics, DMA descriptor formats, completion behavior, and vendor-specific protocol that static identifiers cannot provide.
- Copying a config space dump without implementing correct MMIO access semantics gets you through enumeration. It will not get you through driver initialization for any non-trivial device.
- MITM passthrough devices preserve genuine device behavior by routing real hardware traffic. They address a different threat model than FPGA-only endpoint emulation. If you're buying one off Discord, the firmware you get won't be yours and the seller won't support it. Build your own or don't use one.
- Read-only DMA through an IOMMU-enforced domain may generate translation faults. Whether those faults are consumed and acted upon by anti-cheat software varies by product, version, platform configuration, and fault-reporting path.

### **19.6 MITM passthrough: why real hardware beats config-space faking**

Hardware man-in-the-middle interposers sit physically between a real donor PCIe device and the host's PCIe slot. The donor device is connected to one side of the interposer, the host to the other. The interposer's FPGA sits on the PCIe link, passively observing or selectively modifying TLPs as they pass through. Don't go buying interposer boards from anyone selling them on Discord -- they're all scammers. The architecture is sound, the sellers are not. Build it yourself or don't bother.

This architecture defeats config-space-based detection by construction:

- **Config reads come from real silicon.** When the host reads offset 0x00, it gets the donor's actual VID/DID from the donor's actual configuration space, returned by the donor's actual PCIe core. There is no shadow config to detect because there is no shadow -- the real device answers. No `.coe` file. No writemask. No `cfg_deviceid` register in SystemVerilog. The bytes come from atoms.
- **Behavioral probes pass automatically.** When Vanguard sends a fake WiFi packet to an AX200, the real AX200 responds with its real MAC firmware behavior. The interposer doesn't need to emulate WiFi -- it just passes the traffic. The host sees genuine device behavior because it IS genuine device behavior.
- **Interrupt timing, power state transitions, link training** -- all of it is real. The donor negotiated the link speed and width. The donor's config space capability chain was built by its own silicon. There is nothing to mismatch.

**The tradeoffs:**

- You need a physical donor card on the bench. This is a hardware requirement, not a firmware one.
- Interposer boards are specialized hardware with their own cost and availability constraints.
- Throughput depends on the interposer's FPGA and firmware. A naive passthrough adds latency; a well-designed one can operate at near-line rate.
- The interposer can still be detected if it modifies traffic in observable ways (altered timing, injected TLPs with wrong sequence numbers, flow control credit mismatches). Passive observation is safer than active modification.
- The host still sees a PCIe device in a slot. If the anti-cheat knows what devices SHOULD be in each slot (e.g., "slot 3 should be empty"), a new device appearing is suspicious regardless of how real it looks.

**When MITM beats FPGA-only emulation:**

The detection arms race is a game of observable differences. FPGA-only emulation creates an observable difference at the config space level unless the shadow is byte-perfect AND writemask-perfect AND behaviorally consistent. MITM eliminates the config-space attack surface entirely by outsourcing it to real hardware. The remaining attack surface is traffic modification observability and physical presence detection.

This is not a silver bullet. It shifts the problem from "convince the host I am a real device" to "do not let the host notice I am between it and a real device." Different threat model, different failure modes.

### **19.7 DMA defense (dual-use / blue-team)**

*   **DMAArmor** (NDSS 2026) -- FPGA **PCIe interposer** doing real-time TLP inspection with a kernel module that derives DMA-buffer lifetimes from the Linux DMA API; byte-granular spatial + immediate (TOCTOU-free) temporal revocation, extends to P2P and pre-boot.
*   **Redirected Address Tables / DmaProtect** (Meekolab 2025; PoC by cutecatsandvirtualmachines) -- software IOMMU-style remapping without VT-d.
*   **Deferred DMA Attack** (IEEE) -- documents IOMMU TOCTOU/bypass classes; read it to understand what defenders must close.
*   **Operational hardening**: enable IOMMU/VT-d + Secure Boot + measured boot + Kernel DMA Protection; update motherboard firmware; physically control chassis/ports on high-value endpoints.

### **19.8 Automated firmware tooling (2024-2026)**

The manual `.coe` workflow is increasingly replaced by open-source automation: live VFIO donor dumps to shadow `.coe` plus writemasks, dynamic capability generation, and full bitstream output. Config space `.coe` and writemask `.coe` generation from `lspci -xxxx` dumps or Telescan exports is now standard. A ready-to-use converter is available (`tools/config_space_to_coe.py`).



---

**End of Guide**

You made it. You now understand PCIe configuration space, how to dump a donor device with Arbor, how to modify SystemVerilog without Vivado vomiting 47 cryptic synthesis warnings at you, and why paying for firmware you didn't build is a losing game. Go build something. Break something. Fix it. Repeat until it works or you run out of FPGA boards. Ross Freeman gave us programmable silicon. Terry Davis showed us one man can write an entire operating system while the world called him crazy. The universe is absurd and you're just electrons moving through a Xilinx Artix-7 at 100 MHz. Make the electrons count.

May God bless your FPGA. He blessed Terry's. Yours can't be that much worse.
