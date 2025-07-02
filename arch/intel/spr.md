# SST
* Speed Select Technology is an umbrella term for a collection of features that offer more granular control over CPU performance
  * **SST-Performance Profile** – Allows the CPU to run in one of three performance profiles (defined by core count, base frequency, and more)
  * **SST-Base Frequency** – Enables some CPU cores to run at a higher base frequency in return for other cores running at a lower base frequency
  * **SST-Core Power** – Allows software to prioritize which cores will receive excess power after satisfying minimum requirements
  * **SST-Turbo Frequency** – Allows software-selected cores to achieve a higher max turbo frequency by reducing other cores’ max turbo frequency

# Intel DLB
* Intel DLB improves the system performance related to handling network data on multi-core Intel Xeon Scalable processors:
  * Distributed Processing
    * Intel DLB enables the efficient distribution of network processing across multiple CPU cores/threads;
  * Dynamic Load Balancing
    * Intel DLB dynamically distributes network data across multiple CPU cores for processing as the system load varies;
  * Dynamic Network Processing Reordering
    * Intel DLB restores the order of networking data packets processed simultaneously on CPU cores;
* Scalable
  * Distributes processing across multiple cores to increase throughput
  * No spinlock penalties
* Dynamic
  * Dynamically redistributes processing across multiple cores as system load varies
* Atomic
  * Dynamically allocates a core with a flow based on its load, maintaining its atomicity instead of static core allocation

# DSA
* DSA is an accelerator built into SPR CPU that performs common data mover operations.
* DSA improves performance of applications reliant on data movement. Cores offload data movement operations to DSA, freeing CPU cycles for higher priority work.
* DSA can be used by/with…
  * Hypervisors: DSA aids with VM redundancy and availability via assisting VM migration and check pointing
  * Storage Apps: improves performance by offloading data movement between memory and IO
  * Persistent Memory: includes logic to aid data movement to persistent memory
  * Networking Apps: improves packet processing throughput via data copy offload
  * OS: may improve OS performance by offloading cache flushing, page zeroing, memory moves, etc.
* DSA supports data movement between CPU caches and/or all EGS platform-compatible attached memory and IO devices. DSA can be shared concurrently across OS/VMM, virtual machines, containers, and application processes.

# TDX
* Protection Barriers
  * Protects VMs from other VMs and hypervisor
* Independence
  * Each tenant memory is protected with a unique encryption key
* Simplified deployment
  * No app code modification required, and VM Live Migration support.
* Confidentiality
  * Don’t have to trust the service provider with your secrets
* Remote attestation
  * Provides remote attestation to verify that your workload in a trust domain is authentic and untampered.

# Acronyms

| Abbreviation   | Full Name                            | Description                                                         |
| -------------- | ------------------------------------ | ------------------------------------------------------------------- |
| SPR            | Sapphire Rapids CPU                  |
| EGS            | Eagle Stream platform                |
| UPI            | Ultra Path Interconnect              |
| QAT            | QuickAssist Technology               | accelerates cryptography and data de/compression.                   |
| DLB            | Dynamic Load Balancer                | for efficient load balancing across CPU cores.                      |
| SHA Extensions | Secure Hash Algorithm Extensions     |
| CXL            | Compute Express Link                 |
| IAX            | In-Memory Analytics Accelerator      | increases query throughput and decrease memory footprint in analytics |
| AMX            | Advanced Matrix Extensions           | extending built-in AI acceleration capabilities on Xeon Scalable |
| DSA            | Data Streaming Accelerator           |
| HBM            | High Bandwidth Memory                | to accelerate memory-bandwidth sensitive workloads                  |
| AiA            | Accelerator Interfacing Architecture | Low latency user-space work dispatch and synchronization between software and accelerators.  |
| SVM            | Intel Shared Virtual Memory          |
| SIOV           | Intel Scalable IO Virtualization     |
| IPU            | Intel Platform Update                | provides updates for functional issues and security vulnerabilities that may affect the platform and related HW/SW components |
| ESU            | End of Servicing Updates
| ESUN           | End of Servicing Updates Notification|
| VRoD           | VR-on-DIMM architecture              | improved power delivery and DRAM yields
| SPC            | Sockets Per Channel                  |
| SST            | Speed Select Technology              | is an umbrella term for a collection of features that offer more granular control over CPU performance. |
| TMUL           | Tile Matrix Multiply                 | Set of matrix multiply instructions, the first operators on TILEs |
| VMD            | Volume Management Device             |
| VROC           | Virtual RAID on CPU                  |
| RAS            | Reliability Availability Serviceability |
| SGX            | Software Guard Extensions            | Provides fine grain data protection via application isolation in memory. |
| TEM            | Trusted Environment Mode             |
| PFR            | Platform Firmware Resilience         |
| CET            | Control flow Enforcement Tech        | Hardening platform against stack return attacks. |
| ROP            | Return Oriented Programming          |
| COP            | Call Oriented Programming            |
| SHSTK          | Shadow Stack                         |
| IBT            | Indirect Branch Tracking             |
| TME-MK         | Total Memory Encryption - Multi-Key  |
| TCB            | Trusted Compute Base                 |
| SPDM           | Security Protocol and Data Model     |
| VT-RP          | Virtualization Technology - Redirect Protection | renders page table attacks ineffective by enabling VMM to enforce desired guest linear address translations. |
| EPT            | Extended Page Table                  |
| RCL            | Restricted linear checks             | a CPU capability that enables the VMM to efficiently secure guest linear address translations to assert how the final page table is mapped. |
| PLR            | Protected Linear Range               |
| TDX            | Trust Domain Extensions              |
| SEAM           | Secure-Arbitration Mode              |
| S-IOV          | Scalable IO Virtualization           |
| PMT            | Platform Monitoring Technology       | a unified solution and standard method for discovering, collecting and managing telemetry data across components. |
| PAT            | Platform Analysis Technologies       |
| LBR            | Last Branch Record                   |
| ACD            | Autonomous Crash Dump                |
| TOS            | Total cost of ownership              |
| LOM            | Ethernet LAN On Motherboard          |
| AICs           | Ethernet PCIe Add-In Cards           |
| ADQ            | Application Device Queues            |
| DDP            | Dynamic Device Personalization       |
| NVO            | Network Virtualization Overlay       |
| RDMA           | Remote Direct Memory Access          |
| DCB            | Data Center Bridge                   |
| SMB            | Server Message Block                 |
| AVX            | Advanced Vector Extensions           |
| HFNI           | Half Float New Instructions          |
| UPF            | User Plane Function                  |
| RDT            | Resource Director Technology         | brings visibility and control over how shared resources such as last-level cache (LLC) and memory bandwidth are used by applications, virtual machines (VMs), and containers. |
| RMID           | Resource Monitoring ID               |
| CMT            | Cache Monitoring Technology          |
| CAT            | Cache Allocation Technology          |
| MBM            | Memory Bandwidth Monitoring          |
| MBA            | Memory Bandwidth Allocation          |
| TCO            | Total Cost of Ownership              |
| TXT            | Trusted Execution Technology         |
| S3M            | Secured Startup Services Module      | is CPU IP subsystem that provides a consolidated hardware and firmware infrastructure to deliver a diverse set of features including platform boot and security related aspects. It is designed to support integrated boot and many features that required a controller and firmware infrastructure.
| CMR            | Convertible Memory Region            |