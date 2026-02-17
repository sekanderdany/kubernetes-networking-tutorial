# Cilium Overview

This document provides a **comprehensive overview of Cilium**, explaining what it is, how it works, its eBPF foundation, Hubble observability, and network policy capabilities.

---

## 1. Understanding Cilium

### What is Cilium?

**Cilium** is an open-source, cloud-native solution for providing, securing, and observing network connectivity between workloads. It is a **CNI plugin** for Kubernetes that leverages **eBPF (extended Berkeley Packet Filter)** technology in the Linux kernel.

### Origin and Background

- Created by **Isovalent** (acquired by Cisco)
- Open-source project hosted on GitHub
- Graduated project in the **CNCF (Cloud Native Computing Foundation)**
- First major networking solution to fully embrace eBPF
- Gaining rapid adoption in the cloud-native ecosystem

### Core Philosophy

Cilium operates on a simple but powerful principle:

> **"Secure and observe everything at the kernel level without proxy overhead"**

### What Makes Cilium Different

| Aspect | Traditional CNIs (Calico, Flannel) | Cilium |
|--------|-----------------------------------|---------|
| **Technology** | iptables, IPVS, VXLAN | eBPF |
| **Layer** | L3/L4 | L3/L4/L7 |
| **Observability** | Limited logs | Deep observability |
| **Performance** | Good, but with overhead | Near-native kernel speed |
| **Protocol Awareness** | Basic | Advanced (HTTP, gRPC, Kafka, etc.) |
| **Security** | Network policies | L7 network policies + auth |

### Cilium Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │   Control Plane  │          │   Control Plane  │         │
│  │                  │          │                  │         │
│  │  ┌────────────┐  │          │  ┌────────────┐  │         │
│  │  │Cilium Agent│  │          │  │Cilium Agent│  │         │
│  │  │(cilium)    │  │          │  │(cilium)    │  │         │
│  │  └────────────┘  │          │  └────────────┘  │         │
│  │         ↓        │          │         ↓        │         │
│  │  ┌────────────┐  │          │  ┌────────────┐  │         │
│  │  │Hubble Relay│  │          │  │Hubble Relay│  │         │
│  │  └────────────┘  │          │  └────────────┘  │         │
│  └──────────────────┘          └──────────────────┘         │
│                                                              │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │      Node 1      │          │      Node 2      │         │
│  │                  │          │                  │         │
│  │  ┌────────────┐  │          │  ┌────────────┐  │         │
│  │  │Cilium Agent│  │          │  │Cilium Agent│  │         │
│  │  │(cilium)    │  │          │  │(cilium)    │  │         │
│  │  └──────┬─────┘  │          │  └──────┬─────┘  │         │
│  │         │        │          │         │        │         │
│  │  ┌──────▼─────┐  │          │  ┌──────▼─────┐  │         │
│  │  │  eBPF Maps │  │          │  │  eBPF Maps │  │         │
│  │  │ (Data)     │  │          │  │ (Data)     │  │         │
│  │  └──────┬─────┘  │          │  └──────┬─────┘  │         │
│  │         │        │          │         │        │         │
│  │  ┌──────▼─────┐  │          │  ┌──────▼─────┐  │         │
│  │  │  eBPF Progs│  │          │  │  eBPF Progs│  │         │
│  │  │ (Kernel)   │  │          │  │ (Kernel)   │  │         │
│  │  └──────┬─────┘  │          │  └──────┬─────┘  │         │
│  │         │        │          │         │        │         │
│  │  ┌──────▼─────┐  │          │  ┌──────▼─────┐  │         │
│  │  │    Pods    │  │          │  │    Pods    │  │         │
│  │  └────────────┘  │          │  └────────────┘  │         │
│  └──────────────────┘          └──────────────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

1. **Cilium Agent** (`cilium`)
   - Runs on each node as a DaemonSet
   - Manages eBPF programs and maps
   - Interacts with Kubernetes API server
   - Enforces network policies
   - Provides API for configuration

2. **eBPF Maps**
   - In-kernel data structures
   - Store configuration and state
   - Shared between eBPF programs and user space

3. **eBPF Programs**
   - Run in kernel space
   - Hook into various kernel subsystems
   - Process network traffic
   - Enforce policies

4. **Cilium CLI** (`cilium`)
   - Command-line interface for management
   - Debugging and troubleshooting
   - Status reporting

5. **Hubble**
   - Network observability platform
   - Service map visualization
   - Flow monitoring and tracing

---

## 2. How Cilium Works

### High-Level Operation

Cilium works by:

1. **Loading eBPF programs** into the Linux kernel
2. **Attaching programs** to various hook points (network, sockets, etc.)
3. **Using eBPF maps** to store and share configuration
4. **Processing packets** at kernel level for maximum performance
5. **Enforcing policies** at multiple layers (L3, L4, L7)

### Packet Processing Flow

```
1. Packet Enters Network Interface
   ↓
2. eBPF XDP Hook (Optional)
   - Early filtering
   - Load balancing
   - DDoS protection
   ↓
3. eBPF TC (Traffic Control) Hook
   - Network policy enforcement
   - Packet filtering
   - NAT/Port forwarding
   ↓
4. Socket Lookup
   - eBPF socket programs
   - L7 protocol filtering
   ↓
5. Packet Delivered to Application
```

### Cilium Agent Responsibilities

The Cilium agent (running on each node) handles:

#### 1. Node Management

- Discovers other nodes in the cluster
- Exchanges identity information
- Maintains node-to-node connectivity

#### 2. Identity Management

- Assigns unique identities to pods/services
- Maps identities to IP addresses
- Shares identity information across cluster

#### 3. Policy Distribution

- Receives network policies from Kubernetes
- Compiles policies to eBPF bytecode
- Loads eBPF programs into kernel
- Updates eBPF maps with policy rules

#### 4. Health Monitoring

- Monitors connectivity to other nodes
- Detects network partitions
- Reports status to API server

#### 5. IPAM (IP Address Management)

- Manages IP address allocation for pods
- Integrates with Kubernetes CNI requirements

### Communication Between Agents

```
Cilium Agent (Node 1)          Cilium Agent (Node 2)
        ↓                              ↑
        └──────────────────────────────┘
                   Gossip Protocol
         - Identity exchange
         - Policy synchronization
         - Heartbeat/health checks
```

### Cilium Agent - Kubernetes Integration

```
┌─────────────────────────────────────┐
│     Kubernetes API Server          │
│                                     │
│  - Pods                             │
│  - Services                         │
│  - Network Policies                 │
│  - Endpoints                        │
└──────────────┬──────────────────────┘
               ↓ (Watches resources)
┌─────────────────────────────────────┐
│      Cilium Agent (per node)        │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  Identity Manager            │    │
│  │  - Assigns identities        │    │
│  │  - Maps IPs to identities    │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  Policy Engine              │    │
│  │  - Compiles K8s policies    │    │
│  │  - Generates eBPF bytecode  │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  eBPF Manager               │    │
│  │  - Loads eBPF programs       │    │
│  │  - Updates eBPF maps        │    │
│  └─────────────────────────────┘    │
└──────────────┬──────────────────────┘
               ↓ (Loads programs)
┌─────────────────────────────────────┐
│         Linux Kernel                │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  eBPF Programs              │    │
│  │  - XDP                      │    │
│  │  - TC (Traffic Control)     │    │
│  │  - Socket Operations        │    │
│  │  - Cgroup hooks             │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  eBPF Maps                  │    │
│  │  - Identity mappings        │    │
│  │  - Policy rules             │    │
│  │  - Connection tracking      │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

### Node Discovery and Identity Exchange

When Cilium starts on a node:

1. **Identity Assignment**
   - Agent assigns itself a unique identity
   - Registers with etcd (or CRD-based storage)

2. **Node Discovery**
   - Agent discovers other nodes via Kubernetes API
   - Initiates gossip protocol for node-to-node communication

3. **Identity Exchange**
   - Each node shares its identity information
   - Pods/services get identities based on labels
   - Identity-to-IP mappings are distributed

4. **Policy Synchronization**
   - Network policies are distributed across agents
   - Each agent compiles and loads policies locally

---

## 3. eBPF (Extended Berkeley Packet Filter)

### What is eBPF?

**eBPF (extended Berkeley Packet Filter)** is a revolutionary technology that allows **safe and efficient sandboxed programs** to run in the Linux kernel without modifying kernel source code or loading kernel modules.

### Evolution from BPF

```
BPF (Berkeley Packet Filter)
   ↓ (1992)
   - Original packet filtering
   - tcpdump, libpcap

cBPF (Classic BPF)
   ↓ (2014)
   - Extended capabilities
   - Still packet-focused

eBPF (Extended BPF)
   ↓ (2014+)
   - General-purpose kernel programming
   - Network, security, observability, tracing
   - Just-In-Time (JIT) compilation
```

### Why eBPF is Revolutionary

#### Before eBPF

```
User Space              Kernel Space
    ↓                        ↓
Application          →  System Call
                          ↓
                      Kernel Function
                          ↓
                      (Kernel Module)
                          ↓
                      Back to User Space

Problems:
- Kernel modules are dangerous (can crash system)
- Complex to develop and maintain
- Security risks
- Hard to debug
```

#### With eBPF

```
User Space              Kernel Space
    ↓                        ↓
Application          →  System Call
                          ↓
                      eBPF Program
                      (Verified & JIT compiled)
                          ↓
                      Back to User Space

Benefits:
- Safe (verifier prevents crashes)
- Fast (JIT compilation to native code)
- No kernel modifications needed
- Easy to develop and debug
- Portable across kernel versions
```

### eBPF Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        User Space                            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ Application  │  │ Monitoring   │  │ eBPF Loader      │    │
│  │              │  │ Tools        │  │ (Cilium, etc.)   │    │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘    │
│         │                  │                  │               │
│         └──────────────────┴──────────────────┘               │
│                            │                                  │
└────────────────────────────┼──────────────────────────────────┘
                             │ Syscalls / Netlink
┌────────────────────────────┼──────────────────────────────────┐
│                         Kernel Space                         │
│                            ↓                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   eBPF Verifier                         │ │
│  │  - Validates program safety                             │ │
│  │  - Checks for loops, bounds, invalid memory access     │ │
│  └──────────────────┬─────────────────────────────────────┘ │
│                     │ Safe Program                           │
│                     ↓                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   JIT Compiler                         │ │
│  │  - Compiles to native machine code                     │ │
│  │  - Optimizes for performance                           │ │
│  └──────────────────┬─────────────────────────────────────┘ │
│                     │ Native Code                          │
│                     ↓                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   eBPF Programs                         │ │
│  │                                                     │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │ │
│  │  │ XDP     │ │ TC      │ │ Socket  │ │ Trace   │    │ │
│  │  │ Hook    │ │ Hook    │ │ Ops     │ │ Points  │    │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   eBPF Maps                            │ │
│  │  - Key-value stores in kernel                         │ │
│  │  - Shared between programs and user space             │ │
│  │  - Persist across program executions                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              eBPF Hook Points                          │ │
│  │                                                     │ │
│  │  Network    Socket    Tracepoint  Kprobe  Uprobe     │ │
│  │   Interfaces  Ops                                   │ │
│  └────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### eBPF Key Concepts

#### 1. eBPF Programs

- Small programs written in a restricted C-like language
- Loaded into kernel and attached to hooks
- Execute when the hook is triggered

**Example hooks:**
- **XDP**: Early packet processing at driver level
- **TC (Traffic Control)**: Network stack packet processing
- **Socket Operations**: Socket-level filtering and operations
- **Tracepoints**: Kernel trace events
- **Kprobes/Uprobes**: Dynamic function tracing

#### 2. eBPF Maps

- Data structures stored in kernel memory
- Shared between eBPF programs and user space
- Support various types: hash, array, LPM trie, etc.

**Common map types:**
- `BPF_MAP_TYPE_HASH`: Generic hash map
- `BPF_MAP_TYPE_ARRAY`: Fixed-size array
- `BPF_MAP_TYPE_LPM_TRIE`: Longest prefix match for IP addresses
- `BPF_MAP_TYPE_PERCPU_HASH`: Per-CPU hash map

#### 3. eBPF Verifier

- Critical safety component
- Validates eBPF programs before loading
- Checks for:
  - No infinite loops
  - No invalid memory access
  - No kernel crashes possible
  - Bounded execution time

#### 4. JIT (Just-In-Time) Compilation

- Converts eBPF bytecode to native machine code
- Executes at near-native kernel speed
- Optimized for the specific CPU architecture

### eBPF Capabilities

#### Networking

- **Packet filtering**: Advanced filtering beyond simple BPF
- **Load balancing**: eBPF-based L4/L7 load balancing
- **NAT/Port forwarding**: Network address translation
- **DDoS protection**: Early packet dropping at XDP
- **QoS**: Traffic shaping and prioritization

#### Security

- **System call filtering**: Seccomp integration
- **Access control**: Fine-grained permissions
- **Network policy enforcement**: L3-L7 policies
- **Process monitoring**: Track process behavior

#### Observability

- **Tracing**: Hook into any kernel function
- **Profiling**: CPU profiling and flame graphs
- **Monitoring**: Real-time metrics collection
- **Debugging**: Kernel-level debugging

### eBPF vs Traditional Approaches

| Aspect | Traditional (iptables, kernel modules) | eBPF |
|--------|----------------------------------------|------|
| **Performance** | Multiple table traversals, overhead | Single pass, JIT-compiled |
| **Flexibility** | Limited filtering capabilities | Programmable, dynamic |
| **Safety** | Kernel modules can crash system | Verified, sandboxed |
| **Updates** | Often requires kernel reload | Dynamic updates |
| **Abstraction** | Layer 3/4 only | Layer 3-7+ |
| **Observability** | Limited | Deep visibility |

### eBPF System Requirements

- Linux kernel version: **4.4+** (basic), **5.10+** (recommended)
- Architecture: x86_64, ARM64, etc.
- Feature requirements:
  - `CONFIG_BPF=y` (BPF support)
  - `CONFIG_BPF_SYSCALL=y` (BPF system call)
  - `CONFIG_BPF_JIT=y` (JIT compilation)

### eBPF Ecosystem

```
                    ┌─────────────────┐
                    │   eBPF          │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐        ┌────▼────┐
    │Networking│        │Security  │        │Observability│
    └────┬────┘         └────┬────┘        └────┬────┘
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐        ┌────▼────┐
    │ Cilium  │         │Falco    │        │BCC, bpftrace│
    │Katran   │         │Tetragon │        │Pixie   │
    │Suricata │         │         │        │Parca   │
    └─────────┘         └─────────┘        └─────────┘
```

---

## 4. eBPF for Container Networking

### Why eBPF is Perfect for Container Networking

Container networking has unique challenges that eBFP solves elegantly:

1. **Dynamic environment**: Containers are constantly created/destroyed
2. **High scale**: Thousands of pods and services
3. **Complex policies**: Fine-grained security requirements
4. **Performance**: Minimal overhead is critical
5. **Observability**: Deep visibility needed

### eBPF Hook Points Used by Cilium

#### 1. XDP (eXpress Data Path)

**Location**: Earliest point in packet reception (driver level)

**Capabilities**:
- Ultra-fast packet processing
- DDoS protection
- Load balancing at line rate
- Packet filtering before network stack

**Performance**: Millions of packets per second

**Use Cases**:
- DDoS mitigation
- L4 load balancing
- Network policing

```
Packet → NIC → XDP eBPF → Drop/Forward/Pass → Network Stack
```

#### 2. TC (Traffic Control) Hooks

**Location**: Network stack ingress/egress

**Capabilities**:
- Network policy enforcement
- NAT/Port forwarding
- Packet modification
- Connection tracking

**Use Cases**:
- Kubernetes network policies
- Service load balancing
- Ingress/egress filtering

```
Ingress: Packet → Network Stack → TC eBPF → Application
Egress: Application → TC eBPF → Network Stack → Packet Out
```

#### 3. Socket Operations

**Location**: Socket creation, binding, connection

**Capabilities**:
- L7 protocol filtering
- Socket-level policies
- Application-aware networking

**Use Cases**:
- HTTP/gRPC filtering
- API-level policies
- Service mesh integration

```
Application → Socket → Socket eBPF → Network Stack
```

#### 4. Cgroup Hooks

**Location**: Cgroup operations for containers

**Capabilities**:
- Container-level policies
- Resource accounting
- Per-container observability

**Use Cases**:
- Per-pod policies
- Container monitoring
- Rate limiting

### Cilium's eBPF-Based Networking Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Container/Pod                           │
│                    (Application)                            │
└────────────────────────┬────────────────────────────────────┘
                         ↓
┌────────────────────────▼────────────────────────────────────┐
│                 eBPF Socket Hooks                           │
│  - L7 protocol filtering (HTTP, gRPC, Kafka, etc.)          │
│  - Socket-level policies                                    │
│  - Application awareness                                    │
└────────────────────────┬────────────────────────────────────┘
                         ↓
┌────────────────────────▼────────────────────────────────────┐
│                 eBPF TC Hooks (Ingress/Egress)              │
│  - Network policy enforcement                                │
│  - NAT/Port forwarding                                      │
│  - Connection tracking                                      │
│  - Rate limiting                                            │
└────────────────────────┬────────────────────────────────────┘
                         ↓
┌────────────────────────▼────────────────────────────────────┐
│                 eBPF XDP Hook (Ingress only)                 │
│  - Early packet filtering                                   │
│  - DDoS protection                                          │
│  - High-performance load balancing                          │
└────────────────────────┬────────────────────────────────────┘
                         ↓
┌────────────────────────▼────────────────────────────────────┐
│              Physical Network Interface                      │
└─────────────────────────────────────────────────────────────┘
```

### eBPF Maps Used by Cilium

Cilium uses various eBPF maps to store state and configuration:

#### 1. Identity Map

```c
// Maps IP addresses to security identities
BPF_MAP_TYPE_HASH: ip_to_identity
{
    "10.244.0.5": Identity 12345,
    "10.244.0.6": Identity 12346
}
```

#### 2. Policy Map

```c
// Stores network policy rules
BPF_MAP_TYPE_HASH: policy_rules
{
    Identity 12345 -> [Allow: TCP 80, Allow: TCP 443],
    Identity 12346 -> [Deny: All]
}
```

#### 3. Endpoint Map

```c
// Stores pod endpoint information
BPF_MAP_TYPE_HASH: endpoints
{
    Identity 12345: {
        ip: "10.244.0.5",
        mac: "00:11:22:33:44:55",
        namespace: "default",
        labels: ["app=nginx"]
    }
}
```

#### 4. CT Map (Connection Tracking)

```c
// Tracks active connections
BPF_MAP_TYPE_HASH: ct_entries
{
    tuple(src=10.244.0.5, dst=10.244.0.6, sport=54321, dport=80): {
        state: ESTABLISHED,
        created: timestamp,
        seen: timestamp
    }
}
```

#### 5. NAT Map

```c
// NAT translations for services
BPF_MAP_TYPE_HASH: nat_entries
{
    service_port: 80: {
        backend_ips: ["10.244.0.7", "10.244.0.8"],
        algorithm: "maglev"
    }
}
```

### Packet Flow Through Cilium's eBPF

#### Ingress Flow (Packet Entering Pod)

```
1. Physical NIC receives packet
   ↓
2. XDP eBPF hook (optional)
   - Check if DDoS attack
   - Early filtering
   - Pass to network stack
   ↓
3. Network stack processing
   ↓
4. TC eBPF hook (ingress)
   - Extract source IP
   - Lookup identity from IP
   - Check network policies
   - Allow/Deny based on policy
   ↓
5. Socket lookup
   ↓
6. Socket eBPF hook (optional)
   - L7 protocol filtering
   - Application-specific checks
   ↓
7. Deliver to application
```

#### Egress Flow (Packet Leaving Pod)

```
1. Application sends packet
   ↓
2. Socket eBPF hook (optional)
   - Check application policies
   - L7 protocol enforcement
   ↓
3. Socket lookup
   ↓
4. TC eBPF hook (egress)
   - Extract destination IP
   - Lookup destination identity
   - Check network policies
   - Apply NAT if needed
   - Allow/Deny based on policy
   ↓
5. Network stack processing
   ↓
6. XDP eBPF hook (optional - for egress to other pods)
   ↓
7. Send via physical NIC
```

### Identity-Based Networking

Cilium uses **security identities** instead of IP addresses for policy enforcement:

#### Why Identities?

- **Pods are ephemeral**: IPs change frequently
- **Policies are identity-based**: Labels define security posture
- **Performance**: Identity lookups are faster than label matching
- **Scalability**: Easier to manage thousands of pods

#### Identity Assignment

```
Pod Labels: {
    app: "nginx",
    tier: "frontend",
    env: "production"
}
    ↓
Cilium Identity Manager
    ↓
Hash function or sequential ID
    ↓
Identity: 12345

Identity 12345 = {
    namespace: "default",
    labels: ["app=nginx", "tier=frontend", "env=production"]
}
```

#### Identity Distribution

```
Node 1                 Node 2                 Node 3
  ↓                      ↓                      ↓
Pod A (10.244.0.5)    Pod B (10.244.1.5)    Pod C (10.244.2.5)
Identity: 12345       Identity: 12346       Identity: 12347
    ↓                      ↓                      ↓
Gossip Protocol Exchange
    ↓                      ↓                      ↓
All nodes have:
- Identity 12345 = 10.244.0.5
- Identity 12346 = 10.244.1.5
- Identity 12347 = 10.244.2.5
```

### Network Policy Enforcement with eBPF

Cilium compiles Kubernetes Network Policies to eBPF bytecode:

#### Kubernetes Network Policy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

#### Cilium Compilation to eBPF

```
1. Parse Kubernetes Network Policy
   ↓
2. Map labels to identities
   - app=nginx → Identity 12345
   - app=frontend → Identity 12346
   ↓
3. Compile to eBPF bytecode
   ↓
4. Load into TC eBPF hook
   ↓
5. Update Policy Map

eBPF Policy Map Entry:
{
    src_identity: 12346,
    dst_identity: 12345,
    protocol: TCP,
    port: 80,
    action: ALLOW
}
```

#### eBPF Policy Lookup (Pseudocode)

```c
int tc_policy(struct __sk_buff *skb) {
    // Extract packet info
    __be32 src_ip = extract_src_ip(skb);
    __be32 dst_ip = extract_dst_ip(skb);
    __u16 dst_port = extract_dst_port(skb);
    
    // Lookup identities
    __u32 src_identity = lookup_identity(src_ip);
    __u32 dst_identity = lookup_identity(dst_ip);
    
    // Lookup policy
    struct policy_rule *rule = lookup_policy(
        src_identity, 
        dst_identity, 
        TCP, 
        dst_port
    );
    
    if (rule && rule->action == ALLOW) {
        return TC_ACT_OK;  // Allow packet
    } else {
        return TC_ACT_SHOT; // Drop packet
    }
}
```

### Performance Advantages

#### 1. Kernel-Space Processing

- **No context switches**: Processing happens in kernel
- **No userspace proxy**: Direct packet handling
- **JIT compilation**: Native machine code execution

#### 2. Single Pass Processing

Traditional iptables:
```
Packet → Table 1 → Table 2 → Table 3 → ... → Table N → Action
```

Cilium eBPF:
```
Packet → Single eBPF lookup → Action
```

#### 3. Identity-Based Lookups

- **O(1) lookup time**: Hash map lookups
- **No label matching at runtime**: Labels already resolved to identities
- **Scalable**: Millions of identities supported

#### 4. Connection Tracking

- **Stateful enforcement**: Track connection state
- **Fast path for established connections**: Bypass policy checks
- **Efficient NAT**: In-kernel NAT without proxy overhead

### Comparison: Cilium vs Traditional CNI

| Aspect | Calico (iptables) | Cilium (eBPF) |
|--------|-------------------|---------------|
| **Policy Enforcement** | Multiple iptables chains | Single eBPF lookup |
| **Context Switches** | Multiple (userspace daemon) | None (kernel-space) |
| **Connection Tracking** | iptables conntrack | eBPF CT map |
| **L7 Filtering** | Requires proxy | Native eBPF |
| **Policy Updates** | Reload iptables rules | Update eBPF maps |
| **Scale Limit** | Thousands of policies | Tens of thousands |
| **Latency** | ~10-50 μs | ~1-5 μs |
| **CPU Usage** | Higher (iptables overhead) | Lower (eBPF efficient) |

---

## 5. Hubble

### What is Hubble?

**Hubble** is a fully distributed **network security and observability platform** built on top of Cilium. It provides deep visibility into network traffic and security policies for container workloads.

### Core Philosophy

> **"You can't secure what you can't see"**

Hubble provides the observability needed to understand, debug, and secure networked applications.

### Hubble Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │      Node 1      │          │      Node 2      │         │
│  │                  │          │                  │         │
│  │  ┌────────────┐  │          │  ┌────────────┐  │         │
│  │  │Cilium Agent│  │          │  │Cilium Agent│  │         │
│  │  │(cilium)    │  │          │  │(cilium)    │  │         │
│  │  └─────┬──────┘  │          │  └─────┬──────┘  │         │
│  │        │          │          │        │          │         │
│  │  ┌─────▼──────┐  │          │  ┌─────▼──────┐  │         │
│  │  │Hubble Client│  │          │  │Hubble Client│  │         │
│  │  │(embedded)  │  │          │  │(embedded)  │  │         │
│  │  └─────┬──────┘  │          │  └─────┬──────┘  │         │
│  │        │          │          │        │          │         │
│  │  ┌─────▼──────┐  │          │  ┌─────▼──────┐  │         │
│  │  │  eBPF Progs │  │          │  │  eBPF Progs │  │         │
│  │  │  (Flows)    │  │          │  │  (Flows)    │  │         │
│  │  └────────────┘  │          │  └────────────┘  │         │
│  └─────────┬────────┘          └─────────┬────────┘         │
│            │                            │                   │
│            └────────────┬───────────────┘                   │
│                         ↓ gRPC                               │
│            ┌─────────────────────────────┐                   │
│            │      Hubble Relay           │                   │
│            │  - Aggregates flow data     │                   │
│            │  - Provides unified API     │                   │
│            │  - Caches flow information   │                   │
│            └──────────────┬──────────────┘                   │
│                           ↓                                   │
│            ┌─────────────────────────────┐                   │
│            │      Hubble UI              │                   │
│            │  - Service map              │                   │
│            │  - Flow visualization        │                   │
│            │  - Real-time monitoring      │                   │
│            └─────────────────────────────┘                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Hubble Client (Embedded in Cilium)

- Runs in each Cilium agent
- Collects flow events from eBPF programs
- Sends flows to Hubble Relay via gRPC
- Receives flow queries and responds

#### 2. Hubble Relay

- Centralized service (Deployment)
- Aggregates flow data from all Hubble clients
- Provides unified gRPC API for flow queries
- Caches flow information for performance
- Scales horizontally

#### 3. Hubble UI

- Web-based interface
- Service map visualization
- Flow visualization and filtering
- Real-time monitoring dashboard

### Flow Data Collection

#### How Flows are Captured

1. **eBPF Hook Points**
   - TC (Traffic Control) hooks capture all packets
   - eBPF programs create flow events

2. **Flow Information Captured**
   ```c
   struct flow_event {
       __u64 timestamp;           // When flow occurred
       __u32 src_identity;       // Source pod identity
       __u32 dst_identity;       // Destination pod identity
       __be32 src_ip;           // Source IP
       __be32 dst_ip;           // Destination IP
       __u16 src_port;          // Source port
       __u16 dst_port;          // Destination port
       __u8 protocol;           // TCP/UDP/ICMP, etc.
       __u8 verdict;            // Allowed/Dropped
       __u8 l7_protocol;        // HTTP, gRPC, etc.
       __u8 direction;          // Ingress/Egress
       char l7_data[...];       // L7 payload info (optional)
   };
   ```

3. **Flow Generation**
   - New flows on SYN (TCP) or first packet (UDP)
   - Flow updates on connection state changes
   - Flow termination on FIN/RST (TCP) or timeout (UDP)

4. **Flow Event Queue**
   - eBPF programs write flow events to perf buffer
   - Hubble client reads from perf buffer
   - Client sends flows to Relay via gRPC

#### Flow Lifecycle

```
Pod A connects to Pod B on port 80

1. SYN packet → TC eBPF hook
   ↓
2. eBPF creates flow event:
   {
       src: Pod A (Identity 12345),
       dst: Pod B (Identity 12346),
       port: 80,
       verdict: ALLOW,
       l7: HTTP
   }
   ↓
3. eBPF sends to perf buffer
   ↓
4. Hubble client reads from perf buffer
   ↓
5. Client sends to Hubble Relay
   ↓
6. Relay caches and stores flow
   ↓
7. UI/CLI can query flow
```

### Hubble Capabilities

#### 1. Service Map

**Visualizes the service mesh**:

```
┌─────────────────────────────────────────────────────────┐
│                     Service Map                           │
│                                                          │
│          ┌──────────┐                                    │
│          │ Frontend │                                    │
│          │ (3 pods) │                                    │
│          └────┬─────┘                                    │
│               │                                           │
│         ┌─────┴─────┐                                    │
│         ↓           ↓                                    │
│  ┌──────────┐  ┌──────────┐                              │
│  │  API     │  │  Auth    │                              │
│  │ (2 pods) │  │ (1 pod)  │                              │
│  └────┬─────┘  └──────────┘                              │
│       │                                                 │
│       ↓                                                 │
│  ┌──────────┐                                          │
│  │ Database │                                          │
│  │ (1 pod)  │                                          │
│  └──────────┘                                          │
│                                                          │
│  Lines indicate traffic flow                            │
│  Thickness = Traffic volume                             │
│  Color = Success/Failure rate                           │
└─────────────────────────────────────────────────────────┘
```

**Features:**
- Auto-discovery of services and pods
- Visual representation of dependencies
- Real-time traffic flow visualization
- Drill-down into specific flows

#### 2. Flow Monitoring

**Detailed flow inspection**:

```
Source                      Destination
┌──────────────────┐        ┌──────────────────┐
│ default/frontend │ ────→  │ default/backend  │
│ Pod: frontend-7 │        │ Pod: backend-5   │
│ IP: 10.244.0.5   │        │ IP: 10.244.1.5   │
│ Port: 54321     │        │ Port: 80         │
└──────────────────┘        └──────────────────┘
       │                             │
       │                             │
       │  Protocol: HTTP             │
       │  Method: GET                │
       │  Path: /api/users           │
       │  Status: 200                │
       │  Latency: 15ms              │
       │  Verdict: ALLOW             │
       │                             │
       └─────────────────────────────┘
```

**Features:**
- Real-time flow monitoring
- L7 protocol details (HTTP, gRPC, Kafka, etc.)
- Connection metadata (latency, status codes, etc.)
- Filter by source, destination, namespace, labels, etc.

#### 3. Flow Tracing

**Follow request paths**:

```
1. User Request
   ↓
2. Ingress → Frontend Pod (HTTP GET /api/data)
   ↓
3. Frontend → Auth Service (HTTP POST /validate)
   ↓
4. Auth Service → Redis (GET token:12345)
   ↓
5. Auth Service → Frontend (200 OK)
   ↓
6. Frontend → Backend Service (HTTP GET /data)
   ↓
7. Backend → Database (SELECT * FROM data)
   ↓
8. Database → Backend (Result set)
   ↓
9. Backend → Frontend (200 OK)
   ↓
10. Frontend → User (Response)

Hubble shows the entire trace with timing!
```

**Features:**
- End-to-end request tracing
- Service-to-service dependencies
- Latency analysis per hop
- Error detection and pinpointing

#### 4. Security Policy Visualization

**See what policies are enforced**:

```
Policy: allow-nginx-ingress

Allowed Flows:
┌────────────────────────────────────────────┐
│ frontend → nginx:80    ✅ ALLOWED         │
│ frontend → nginx:443   ✅ ALLOWED         │
│ backend → nginx:80     ✅ ALLOWED         │
└────────────────────────────────────────────┘

Dropped Flows:
┌────────────────────────────────────────────┐
│ external → nginx:22    ❌ DROPPED         │
│ db → nginx:80          ❌ DROPPED         │
└────────────────────────────────────────────┘
```

**Features:**
- Visualize policy enforcement in real-time
- Identify dropped traffic and why
- Debug policy issues
- Policy compliance verification

#### 5. DNS Observability

**Monitor DNS queries and responses**:

```
DNS Query Flow:
┌────────────────────────────────────────────┐
│ Source: frontend-pod (10.244.0.5)          │
│ Query: backend-service.default.svc.cluster.local│
│ Type: A                                    │
│ → DNS Server (CoreDNS: 10.96.0.10)         │
│ ← Response: 10.96.123.45                   │
│ Latency: 2ms                               │
│ Verdict: ALLOWED                           │
└────────────────────────────────────────────┘
```

**Features:**
- Monitor all DNS queries
- Track DNS latency
- Identify DNS issues
- Detect DNS-based attacks

### Hubble CLI

The `hubble` CLI provides powerful filtering and querying capabilities.

#### Basic Flow Commands

```bash
# View all flows
hubble observe

# Filter by namespace
hubble observe --namespace default

# Filter by pod
hubble observe --pod frontend-7

# Filter by destination
hubble observe --to-pod backend-5

# Filter by protocol
hubble observe --protocol http

# Filter by verdict (dropped packets)
hubble observe --verdict dropped

# Filter by L7 protocol
hubble observe --l7 http

# Show flow details
hubble observe --verbose

# Follow specific flow
hubble observe --follow
```

#### Advanced Filtering

```bash
# Filter by labels
hubble observe --labels app=nginx,tier=frontend

# Filter by service
hubble observe --to-service backend-service

# Filter by IP
hubble observe --ip 10.244.0.5

# Filter by port
hubble observe --port 80

# Filter by method (HTTP)
hubble observe --http-method GET

# Filter by path (HTTP)
hubble observe --http-path /api/users

# Filter by status code (HTTP)
hubble observe --http-status 200,404,500

# Combine filters
hubble observe \
  --namespace default \
  --labels app=frontend \
  --to-service backend-service \
  --protocol http \
  --http-method GET
```

#### Flow Analysis

```bash
# Show flow statistics
hubble observe --summary

# Show dropped flows only
hubble observe --verdict dropped --summary

# Show L7 protocol statistics
hubble observe --l7 --summary

# Export flows to file
hubble observe --output flows.json

# Convert to PCAP for analysis
hubble observe --format pcap > capture.pcap
```

### Hubble UI Features

#### 1. Real-Time Dashboard

- Live flow monitoring
- Traffic volume metrics
- Error rate tracking
- Latency visualization

#### 2. Service Map

- Interactive service topology
- Zoom in/out for different views
- Click services for detailed flow info
- Color-coded by health/error rate

#### 3. Flow Details Panel

- Comprehensive flow information
- L7 payload details
- Connection timeline
- Related flows

#### 4. Search and Filter

- Search by namespace, pod, service, IP
- Filter by protocol, port, verdict
- Time range selection
- Save and share filters

### Hubble Integration

#### With Prometheus/Grafana

```yaml
# Hubble metrics to Prometheus
apiVersion: v1
kind: Service
metadata:
  name: hubble-metrics
  namespace: cilium
spec:
  ports:
  - name: metrics
    port: 9965
    targetPort: 9965
```

**Metrics exposed:**
- `hubble_flows_processed_total`
- `hubble_flows_dropped_total`
- `hubble_flows_processed_latency_seconds`
- `hubble_flows_verdict_total{verdict="forwarded/dropped"}`

#### With Grafana Dashboards

Import Hubble dashboards for:
- Traffic overview
- Security policy enforcement
- L7 protocol analysis
- DNS monitoring

### Use Cases

#### 1. Debugging Network Issues

```bash
# Why can't my frontend talk to backend?
hubble observe --from-pod frontend --to-pod backend --verdict dropped

# Check dropped flows and reason
hubble observe --verdict dropped --verbose
```

#### 2. Understanding Service Dependencies

- Generate service map
- Identify unexpected dependencies
- Visualize communication patterns

#### 3. Security Auditing

```bash
# Check for blocked traffic that should be allowed
hubble observe --verdict dropped --labels app=sensitive

# Verify policies are enforced
hubble observe --namespace production --summary
```

#### 4. Performance Analysis

```bash
# Find slow HTTP requests
hubble observe --l7 http --verbose | grep high latency

# Analyze latency distribution
hubble observe --summary
```

#### 5. Incident Response

```bash
# Investigate DDoS attack
hubble observe --verdict dropped --summary

# Trace malicious traffic source
hubble observe --ip <suspicious-ip>

# Export evidence
hubble observe --output incident.json
```

---

## 6. Cilium Network Policy

### What are Cilium Network Policies?

**Cilium Network Policies** extend the standard Kubernetes NetworkPolicy CRD with additional capabilities, providing fine-grained, layer 3-7 security controls for Kubernetes workloads.

### Policy Types

Cilium supports multiple policy types:

#### 1. CiliumNetworkPolicy (Enhanced Kubernetes Policy)

Extends standard `NetworkPolicy` with:
- L7 protocol awareness
- DNS filtering
- Egress rules (before K8s 1.18)
- CIDR rules with identities
- Advanced selectors

#### 2. CiliumClusterwideNetworkPolicy

Cluster-wide policies that apply across all namespaces, useful for:
- Global security baseline
- Network segmentation
- Compliance requirements

#### 3. CiliumNetworkPolicy (L7 Policy)

Application-layer policies for:
- HTTP method filtering
- Path-based routing
- Header-based rules
- gRPC service filtering

### Policy Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes API Server                     │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Kubernetes NetworkPolicy                               │ │
│  │  - Standard K8s resource                               │ │
│  │  - L3/L4 only                                          │ │
│  │  - Ingress only (until 1.18)                           │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  CiliumNetworkPolicy                                   │ │
│  │  - Extended K8s NetworkPolicy                          │ │
│  │  - L3/L4/L7 support                                    │ │
│  │  - DNS filtering                                       │ │
│  │  - Advanced selectors                                  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  CiliumClusterwideNetworkPolicy                         │ │
│  │  - Cluster-wide policies                                │ │
│  │  - Apply to all namespaces                              │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────┘
                            ↓ Watch
┌───────────────────────────▼─────────────────────────────────┐
│                      Cilium Agent                            │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Policy Compiler                                         │ │
│  │  - Parses YAML policies                                 │ │
│  │  - Resolves labels to identities                        │ │
│  │  - Generates eBPF bytecode                              │ │
│  └─────────────────────────────────────────────────────────┘ │
│                            ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  eBPF Policy Maps                                       │ │
│  │  - Store compiled rules                                 │ │
│  │  - Fast lookup for traffic                              │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────┘
                            ↓ Load
┌───────────────────────────▼─────────────────────────────────┐
│                    Linux Kernel (eBPF)                      │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  TC eBPF Hooks (Ingress/Egress)                         │ │
│  │  - Lookup identities                                    │ │
│  │  - Check policy maps                                    │ │
│  │  - Enforce Allow/Deny                                   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Socket eBPF Hooks                                      │ │
│  │  - L7 protocol filtering                                │ │
│  │  - Application-aware rules                             │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Policy Enrichment Flow

```
1. User creates NetworkPolicy
   ↓
2. Kubernetes API Server stores policy
   ↓
3. Cilium Agent watches for policies
   ↓
4. Policy Compiler:
   - Parse policy YAML
   - Resolve pod labels to identities
   - Calculate policy rules
   ↓
5. Generate eBPF bytecode
   ↓
6. Update eBPF Policy Maps
   ↓
7. eBPF programs enforce policy in kernel
   ↓
8. Traffic is allowed/denied based on policy
```

### L3/L4 Policy Examples

#### Basic Ingress Policy

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-nginx-ingress"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: nginx
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

#### Egress Policy (Beyond Standard K8s)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-backend-egress"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  - toEndpoints:
    - matchLabels:
        app: database
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
```

#### Combined Ingress/Egress Policy

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "full-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
  egress:
  - toEndpoints:
    - matchLabels:
        app: database
    toPorts:
    - ports:
      - port: "3306"
        protocol: TCP
```

#### CIDR-Based Policy

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-external-api"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  - toCIDR:
    - 192.168.1.0/24
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

### L7 Policy Examples

#### HTTP Method Filtering

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "http-method-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/users"
        - method: "POST"
          path: "/api/users"
```

#### HTTP Path Filtering

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "http-path-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/health"
        - method: "GET"
          path: "/api/status"
```

#### HTTP Header Filtering

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "http-header-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - headers:
          - name: "Authorization"
            value: "Bearer .*"
```

#### gRPC Service Filtering

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "grpc-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: grpc-server
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: grpc-client
    toPorts:
    - ports:
      - port: "9090"
        protocol: TCP
      rules:
        grpc:
        - service: "my.service.v1.UserService"
          method: "GetUser"
        - service: "my.service.v1.UserService"
          method: "CreateUser"
```

### DNS Policy

#### DNS Name Filtering

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-dns-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  - toFQDNs:
    - matchName: "api.external.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

### Advanced Selectors

#### Label-Based Selectors

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "label-selector-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: nginx
      tier: frontend
  ingress:
  - fromEndpoints:
    - matchLabels:
        env: production
        team: backend
```

#### Identity-Based Selectors

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "identity-selector-policy"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchExpressions:
      - key: "io.kubernetes.pod.namespace"
        operator: In
        values: ["default", "production"]
```

### Policy Enforcement Flow

```
1. Packet arrives at TC eBPF hook
   ↓
2. Extract source/destination IPs
   ↓
3. Lookup source/destination identities from IP
   ↓
4. Query eBPF Policy Map:
   - Key: (src_identity, dst_identity, protocol, port)
   - Value: Allow/Deny action
   ↓
5. If L4 policy allows:
   - Check if L7 policy exists
   ↓
6. If L7 policy:
   - Parse application payload (HTTP, gRPC, etc.)
   - Match against L7 rules
   ↓
7. Decision:
   - ALLOW: Forward packet
   - DENY: Drop packet
   ↓
8. Log decision (if enabled)
```

### Policy Order of Evaluation

Cilium evaluates policies in this order:

1. **Deny policies first**
2. **Allow policies next**
3. **If no policies match**: Default deny (if default policy is set)

```
Incoming packet
   ↓
Check Deny Policies
   ↓ (if any deny matches)
→ DENY
   ↓ (if no deny matches)
Check Allow Policies
   ↓ (if any allow matches)
→ ALLOW
   ↓ (if no allow matches)
→ DENY (default)
```

### Default Policies

#### Default Allow (No Restrictions)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "default-allow"
  namespace: default
spec:
  endpointSelector:
    matchLabels: {}  # All pods
  ingress:
  - {}  # Allow all ingress
  egress:
  - {}  # Allow all egress
```

#### Default Deny (Zero Trust)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "default-deny"
  namespace: default
spec:
  endpointSelector:
    matchLabels: {}  # All pods
  ingress: []  # Deny all ingress
  egress: []  # Deny all egress
```

### Cluster-Wide Policies

#### Global Egress to External API

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "allow-external-api"
spec:
  endpointSelector:
    matchLabels: {}  # All pods
  egress:
  - toFQDNs:
    - matchName: "api.external.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

#### Global Deny Inter-Namespace Traffic

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "deny-cross-namespace"
spec:
  endpointSelector:
    matchLabels: {}  # All pods
  ingress:
  - fromEndpoints:
    - matchExpressions:
      - key: "io.kubernetes.pod.namespace"
        operator: NotIn
        values: []  # Must be in same namespace
```

### Policy Best Practices

#### 1. Start with Default Deny

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "default-deny"
  namespace: production
spec:
  endpointSelector:
    matchLabels: {}
  ingress: []
  egress: []
```

#### 2. Allow Essential Traffic

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-dns"
  namespace: production
spec:
  endpointSelector:
    matchLabels: {}
  egress:
  - toEndpoints:
    - matchLabels:
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
```

#### 3. Use Labels, Not IPs

❌ **Bad:**
```yaml
toCIDR:
- "10.244.0.5/32"  # IP-based (can change)
```

✅ **Good:**
```yaml
toEndpoints:
- matchLabels:
    app: database  # Label-based (more stable)
```

#### 4. Be Specific with L7 Rules

❌ **Too broad:**
```yaml
rules:
  http:
  - method: "GET"
```

✅ **Specific:**
```yaml
rules:
  http:
  - method: "GET"
    path: "/api/health"
  - method: "GET"
    path: "/api/status"
```

#### 5. Document Policy Intent

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-frontend-to-api"
  namespace: production
  annotations:
    description: "Allows frontend pods to reach API pods on port 8080"
    owner: "platform-team"
    ticket: "SEC-123"
spec:
  # ... policy spec ...
```

### Policy Troubleshooting

#### Check Policy Status

```bash
# List all policies
cilium networkpolicy list

# View policy details
cilium networkpolicy get <policy-name>

# Check if policy is enforced
cilium networkpolicy status <policy-name>
```

#### Debug Dropped Traffic

```bash
# Use Hubble to see dropped flows
hubble observe --verdict dropped

# Check why traffic was dropped
hubble observe --verdict dropped --verbose

# Filter by specific pod
hubble observe --from-pod <pod-name> --verdict dropped
```

#### View eBPF Policy Maps

```bash
# Check policy maps in eBPF
cilium bpf policy get

# Dump specific policy
cilium bpf policy list
```

---

## 7. Key Takeaways (Architect Notes)

### Cilium Fundamentals

- **eBPF-powered**: Uses kernel-level technology for performance and flexibility
- **Identity-based**: Uses security identities, not IP addresses, for policies
- **Layer 3-7 awareness**: Can filter at network and application layers
- **Integrated observability**: Hubble provides deep visibility
- **Kubernetes-native**: Extends standard K8s NetworkPolicy

### eBPF Advantages

- **Performance**: Near-native kernel speed
- **Safety**: Verified and sandboxed
- **Flexibility**: Programmable without kernel modifications
- **Observability**: Deep kernel-level visibility
- **Future-proof**: Actively developed and improving

### Hubble Value

- **Deep visibility**: See all network traffic
- **Real-time debugging**: Identify issues quickly
- **Service mapping**: Understand dependencies
- **Security auditing**: Verify policy enforcement
- **Incident response**: Investigate attacks or issues

### Network Policy Capabilities

- **Fine-grained control**: L3/L4/L7 policies
- **Zero trust**: Default deny model
- **Application awareness**: HTTP/gRPC/Kafka policies
- **DNS security**: FQDN-based policies
- **Cluster-wide**: Apply policies globally

### When to Use Cilium

✅ **Use Cilium when:**
- High performance is critical
- Advanced security policies needed
- Deep observability required
- L7 filtering desired
- Microservices architecture
- Large-scale clusters

❌ **Consider alternatives when:**
- Simple networking only needed
- Older kernel versions (< 4.4)
- Very small clusters
- Learning curve is a concern

---

## 8. Common Interview / Exam Questions

**Q1: What is Cilium?**
A: A cloud-native networking, security, and observability solution using eBPF.

**Q2: What is eBPF?**
A: Extended Berkeley Packet Filter - a kernel technology for running safe programs in kernel space.

**Q3: How does Cilium differ from Calico?**
A: Cilium uses eBPF for L3-L7 policies and observability; Calico uses iptables for L3-L4 only.

**Q4: What is Hubble?**
A: Cilium's observability platform for network visibility and security monitoring.

**Q5: What eBPF hooks does Cilium use?**
A: XDP, TC (Traffic Control), Socket operations, Cgroup hooks.

**Q6: What are Cilium identities?**
A: Security identities assigned to pods based on labels, used for policy enforcement.

**Q7: What L7 protocols does Cilium support?**
A: HTTP, gRPC, Kafka, DNS, and more.

**Q8: What is a CiliumNetworkPolicy?**
A: Extended Kubernetes NetworkPolicy with L7 support and advanced features.

**Q9: How does Cilium enforce policies?**
A: Compiles policies to eBPF bytecode, loads into kernel, enforces at packet level.

**Q10: What is the Cilium agent?**
A: DaemonSet running on each node, manages eBPF programs and policies.

---

## 9. Next Topics to Learn

- **Cilium Installation**: Deploy Cilium on Kubernetes
- **Cilium CLI**: Managing Cilium with command line
- **Hubble Deep Dive**: Advanced observability features
- **Cilium Service Mesh**: Built-in service mesh capabilities
- **Cilium Gateway**: Ingress and gateway API
- **Cilium Bandwidth Manager**: QoS and traffic shaping
- **Cilium Encryption**: WireGuard-based encryption
- **Cilium Multi-cluster**: Networking across clusters

---

## 10. Practical Commands

### Cilium Status and Management

```bash
# Check Cilium status
cilium status

# View cluster connectivity
cilium connectivity test

# Check Cilium version
cilium version

# View Cilium configuration
cilium config view

# List all endpoints
cilium endpoint list

# Get endpoint details
cilium endpoint get <endpoint-id>

# Check policy status
cilium policy get <policy-id>

# List all identities
cilium identity list

# Get identity details
cilium identity get <identity-id>
```

### Hubble Commands

```bash
# Check Hubble status
hubble status

# Observe all flows
hubble observe

# Filter flows
hubble observe --namespace default --pod frontend-7

# View dropped flows
hubble observe --verdict dropped

# Show flow summary
hubble observe --summary

# Export flows
hubble observe --output flows.json
```

### Policy Management

```bash
# List all network policies
cilium networkpolicy list

# Get policy details
cilium networkpolicy get <policy-name>

# Delete a policy
cilium networkpolicy delete <policy-name>

# Check policy enforcement
cilium policy get --labels app=nginx
```

---

**End of Cilium Overview**