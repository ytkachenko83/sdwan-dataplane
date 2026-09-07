# High-Performance SD-WAN Data Plane Engine (`sdwan-dataplane`)

A cutting-edge, high-performance software virtual appliance (vCPE) data plane engine built natively for **FreeBSD** utilizing **Netmap** for kernel-bypass packet forwarding. This system delivers deterministic, wire-speed packet processing, application-aware routing, and low-latency encrypted overlays for modern enterprise edge infrastructure.

---

## Technical Architecture Overview

The system implements a strict split-plane architecture to decouple routing policy definitions from high-speed frame manipulation:

### Virtual Appliance Architecture (vCPE Design)
* **Kernel-Bypass Packet I/O:** Leverages the native FreeBSD **Netmap** subsystem to map NIC ring descriptors directly into the application space, bypassing socket allocations and memory copies.
* **Single-Pass Parallel Execution Architecture:** Packets are processed concurrently within a single memory loop, handling overlay decryption, Layer-7 classification, and QoS matching before writing directly to the outbound tx-ring.
* **Lockless Symmetrical Multi-Processing (SMP):** Leverages ring arrays pinned to hardware CPU worker threads via `pthread_setaffinity_np()`, eliminating lock contention across the forwarding paths.

---

## Core System Features

### 1. Dynamic Path Selection & Link Aggregation
* Real-time active monitoring of multiple underlay circuits (MPLS, Broadband, Satellite, 4G/5G).
* Active out-of-band link probing tracking telemetry (loss, latency, jitter, and path performance metrics).
* Automated sub-second traffic shifting upon underlay circuit degradation or link failure thresholds.

### 2. Application-Aware Routing (AAR)
* First-Packet Classification engine utilizing deep packet signature parsing.
* Dynamic enforcement of path rules based on Layer 7 identity (e.g., routing real-time voice over low-jitter links while routing cloud backups over public broadband).

### 3. Secure Overlay Fabric Encryption
* Zero-Touch overlay configuration utilizing state-of-the-art wire-speed cryptographic tunnels (WireGuard or custom DTLS overlays).
* Automated key rotation mechanics handled out-of-band by the control plane.

### 4. Multi-Tenant Network Segmentation
* Native support for cryptographic traffic isolation over the same physical underlay circuits using unique Virtual Routing IDs (VRIDs).
* Complete isolation of routing matrices and forwarding boundaries to support enterprise multi-tenancy requirements.

### 5. WAN Optimization & Remediation
* **Forward Error Correction (FEC):** Configurable automated inline packet replication algorithms across diverse physical circuits to protect real-time application health on lossy loops.
* **Packet Reordering Engines:** Hardware-accelerated sliding window tracking to remediate packets traversing asynchronous paths back into correct sequence before delivery to local clients.

---
```
+---------------------------------------------------------------------------------+

|                                  CONTROL PLANE                                  |
|                                                                                 |
|     +---------------------------------------------------------------------+     |
|     |                      controlplane-daemon (Rust)                       |     |
|     |  - Orchestrates overlay network paths & populates routing tables    |     |
|     +---------------------------------------------------------------------+     |
+---------------------------------------+-----------------------------------------+
                                        |
                  Shared Memory IPC Ring / Netlink State Notifications
                                        |
+---------------------------------------v-----------------------------------------+

|                                   DATA PLANE                                    |
|                                                                                 |
|  +---------------------------------------------------------------------------+  |
|  |                           sdwan-dataplane Engine                          |  |
|  |  - High-performance, multi-threaded C/C++ execution runtime               |  |
|  |  - Pin-bound worker loops assigned to dedicated physical CPU cores        |  |
|  |                                                                           |  |
|  |  +-----------------------+   +--------------------+   +----------------+  |  |
|  |  |  Netmap Ring Buffer   |   |     DPI Engine     |   | Tunnel Crypter |  |  |
|  |  | - Direct driver rings |   | - Layer-7 parsing  |   | - AES-GCM      |  |  |
|  |  | - Zero-copy mapping   |   | - App identity     |   | - ChaCha20     |  |  |
|  |  +-----------+-----------+   +---------+----------+   +-------+--------+  |  |
|  |              |                         |                      |           |  |
|  |              +-------------------------+----------------------+           |  |
|  |                                        |                                  |  |
|  |                                        v (Single-Pass Forwarding Pipeline)|  |
|  +----------------------------------------+----------------------------------+  |
+-------------------------------------------+-------------------------------------+
                                            |
                  +-------------------------+-------------------------+

                  |                         |                         |
                  v                         v                         v
      +-----------------------+ +-----------------------+ +-----------------------+

      |      WAN Link A       | |      WAN Link B       | |      WAN Link C       |
      |  (Public Broadband)   | |     (Private MPLS)    | |     (5G Cellular)     |
      +-----------------------+ +-----------------------+ +-----------------------+
```
---

## Low-Level Implementation Details

### Telemetry Structure Tracker (`tunnel_telemetry.h`)
Below is the optimized core C layout used by the forwarding threads to track link degradation. It is structured explicitly to prevent variable alignment padding overhead inside memory-constrained cache line layouts:

```c
#include <stdint.h>
#include <sys/time.h>

#define HIST_WINDOW_SIZE 64

struct __attribute__((__packed__)) tunnel_telemetry {
    uint64_t total_tx_packets;       /* Accumulated egress count               */
    uint64_t total_rx_packets;       /* Accumulated ingress count              */
    uint64_t total_lost_packets;     /* Detected sequence discontinuities      */
    
    uint32_t current_latency_us;     /* Active round-trip time in microseconds */
    uint32_t current_jitter_us;      /* Calculated packet arrival variance     */
    float    instantaneous_loss_rate;/* Loss calculation over current window   */

    uint32_t rtt_history[HIST_WINDOW_SIZE]; /* Sliding history array for RTT   */
    uint8_t  history_index;          /* Pointer index for cyclic tracking      */
    uint8_t  link_state_flags;       /* Bitmask flags: 0x01 Active, 0x02 Alert */
};
```

---

## Performance & Build Requirements

### Recommended Hardware Virtual Machine Allocation
* **Host Processor Compatibility:** Apple Silicon (ARM64/AArch64) native via QEMU or UTM utilizing Apple Hypervisor Framework (`hvf`).
* **Symmetric Multiprocessing Config:** Minimum `-smp 8` cores to match parallel interface task affinity routing loops.
* **Memory Limits:** Minimum `-m 16G` memory envelope for buffering high-throughput ZFS telemetry log datasets.

### Target Prerequisites Compilation Matrix
```bash
# Core package installation required to prepare build environment
sudo pkg install -y git cmake gcc16 base64
```

### Compiling and Executing the Dataplane Workspace
```bash
# Clone repository
git clone git@github.com:organization/sdwan-dataplane.git
cd sdwan-dataplane

# Construct make environments
mkdir build && cd build
cmake -DCMAKE_C_COMPILER=gcc16 -DCMAKE_CXX_COMPILER=g++16 ..
make -j\$(sysctl -n hw.ncpu)

# Execute the core dataplane on virtual interface loops using Netmap
sudo ./bin/sdwan_dataplane -i netmap:vtnet0 -m 1
```
