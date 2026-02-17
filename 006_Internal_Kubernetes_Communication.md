# Internal Kubernetes Communication

This document provides a comprehensive guide to **internal Kubernetes communication**, explaining how pods, services, and other components communicate within a Kubernetes cluster.

---

## 1. Kubernetes Network Model

### The Core Principles

Kubernetes networking is built on a simple but powerful set of principles that all networking implementations must follow:

### The Four Fundamental Rules

1. **All pods can communicate with all other pods without NAT**
   - No Network Address Translation between pods
   - Direct pod-to-pod communication
   - Pods see each other's real IP addresses

2. **All nodes can communicate with all pods on that node without NAT**
   - Node can reach pods directly
   - kubelet and node components can talk to pods
   - No NAT required for node-to-pod communication

3. **Pods see the same IP as they do from outside the cluster**
   - No "double NAT" issues
   - Consistent addressing model
   - Simplifies application design

4. **No port mapping required**
   - Each pod gets its own IP
   - Applications use standard ports
   - No complex port coordination

### The Flat Network Model

```
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                           │
│                                                              │
│  Node 1              Node 2              Node 3              │
│  ┌────────┐          ┌────────┐          ┌────────┐         │
│  │ Pod A  │          │ Pod C  │          │ Pod E  │         │
│  │10.244.1.5│         │10.244.2.5│         │10.244.3.5│         │
│  └────┬───┘          └────┬───┘          └────┬───┘         │
│       │                    │                    │                │
│       └────────────────────┴────────────────────┘                │
│                            │                                  │
│                    Flat Network (10.244.0.0/16)               │
│                    - No NAT between pods                     │
│                    - Direct IP communication                │
│                    - Pods see each other's real IPs           │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Model Matters

#### Before Kubernetes Network Model

```
Problem: How do we connect containers across different hosts?

Traditional Solutions:
❌ Host port mapping - Doesn't scale
❌ Overlay networks - Complex, performance overhead
❌ Static IPs - Difficult to manage
❌ NAT everywhere - Debugging nightmares
```

#### With Kubernetes Network Model

```
Solution: Simple, flat network model

Benefits:
✅ Pods act like VMs on the same network
✅ No port conflicts between pods
✅ Direct IP-to-IP communication
✅ Simplified application design
✅ Easier debugging and monitoring
```

### IP Address Management

#### CIDR Allocation

Kubernetes uses CIDR (Classless Inter-Domain Routing) blocks for IP allocation:

```
Cluster CIDR: 10.244.0.0/16
    ↓
Node 1: 10.244.1.0/24
Node 2: 10.244.2.0/24
Node 3: 10.244.3.0/24
```

#### Pod IP Assignment

```
Each pod gets an IP from its node's subnet:

Node 1 (10.244.1.0/24):
- Pod A: 10.244.1.5
- Pod B: 10.244.1.6
- Pod C: 10.244.1.7

Node 2 (10.244.2.0/24):
- Pod D: 10.244.2.5
- Pod E: 10.244.2.6
- Pod F: 10.244.2.7
```

### The CNI's Role

The Kubernetes networking model doesn't specify **how** networking should be implemented - only **what** must work.

```
Kubernetes defines the MODEL:
- All pods can communicate without NAT
- Each pod gets unique IP
- No port conflicts

CNI implementations provide the HOW:
- How to allocate IPs?
- How to route packets?
- How to enforce policies?

Common CNIs:
- Calico (BGP routing)
- Flannel (VXLAN overlay)
- Cilium (eBPF)
- Weave Net (Mesh)
```

### Key Takeaways

✅ **Flat Network**: Pods communicate directly without NAT
✅ **IP-per-Pod**: Each pod has a unique IP address
✅ **No Port Mapping**: Applications use standard ports
✅ **Model Independence**: Different CNIs implement the same model
✅ **Simplified Design**: Applications don't need complex networking logic

---

## 2. Pod-to-Pod on Same Node

### Understanding Same-Node Communication

When two pods reside on the same Kubernetes node, communication happens through the node's internal networking stack.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Node 1                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │      Pod A      │         │      Pod B      │          │
│  │   (nginx)       │         │   (app)         │          │
│  │  10.244.1.5     │         │  10.244.1.6     │          │
│  │                 │         │                 │          │
│  │ ┌─────────────┐ │         │ ┌─────────────┐ │          │
│  │ │ eth0       │ │         │ │ eth0       │ │          │
│  │ │10.244.1.5 │ │         │ │10.244.1.6 │ │          │
│  │ └──────┬──────┘ │         │ └──────┬──────┘ │          │
│  └────────┼─────────┘         └────────┼─────────┘          │
│           │                            │                      │
│           └────────────┬───────────────┘                      │
│                        │                                    │
│                ┌───────▼────────┐                            │
│                │   veth pair    │                            │
│                │  (host bridge) │                            │
│                └───────┬────────┘                            │
│                        │                                    │
│                ┌───────▼────────┐                            │
│                │  cni0 bridge   │                            │
│                │  (Linux bridge)│                            │
│                └───────┬────────┘                            │
│                        │                                    │
│                ┌───────▼────────┐                            │
│                │ Routing Table  │                            │
│                └───────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Communication Flow

#### Step-by-Step Process

1. **Pod A Sends Packet**
   ```
   Pod A (10.244.1.5) → Pod B (10.244.1.6)
   ```

2. **Packet Leaves Pod A's Network Namespace**
   - Exits via `eth0` interface
   - Reaches veth pair on host

3. **Bridge Processing**
   - Packet arrives at `cni0` bridge
   - Bridge learns MAC addresses
   - Packet is forwarded to Pod B's veth

4. **Packet Enters Pod B's Network Namespace**
   - Packet arrives at Pod B's `eth0` interface
   - Application receives packet

### How veth Pairs Work

Virtual ethernet pairs are the key mechanism for same-node pod communication:

```
┌─────────────────────────────────────────────────────────────┐
│                    Node Host Namespace                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              cni0 Bridge                        │    │
│  │                                                      │    │
│  │  ┌─────┐                  ┌─────┐            │    │
│  │  │veth0│                  │veth2│            │    │
│  │  └──┬──┘                  └──┬──┘            │    │
│  └─────┼───────────────────────┼──────────────────┘    │
│        │                       │                          │
│        │ veth pair            │ veth pair              │
│        └───────────────────────┘                          │
│                                │                          │
└────────────────────────────────┼──────────────────────────────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         │         │              │         │               │
    ┌────▼────┐  │       ┌────▼────┐  │       ┌────▼────┐
    │ Pod A   │  │       │ Pod B   │  │       │ Pod C   │
    │ eth0    │  │       │ eth0    │  │       │ eth0    │
    └─────────┘  │       └─────────┘  │       └─────────┘
                   │                   │
              (connected to         (connected to
              veth0)               veth2)
```

**Key Points:**
- Each veth pair connects a pod to the host
- One end in pod namespace, one end in host namespace
- Packets flow seamlessly between namespaces
- Bridge connects all veth pairs

### Bridge-Based Networking

#### How the Bridge Works

```bash
# On the host, check bridge
ip addr show cni0

# Typical output:
8: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 4a:5e:4c:3d:2b:1a brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.1/24 brd 10.244.1.255 scope global cni0
       valid_lft forever preferred_lft forever
```

**Bridge Role:**
- Acts like a virtual switch
- Connects all pods on the node
- Learns MAC addresses for efficiency
- Broadcasts unknown destinations

#### Learning Process

```
Initial State:
- Bridge doesn't know which MAC is on which veth
- Packets are flooded to all veths

After Learning:
- Bridge learns: MAC_A is on veth0, MAC_B is on veth1
- Packets are sent directly to the correct veth
- No unnecessary flooding
```

### Routing Table

#### Host Routing Table

```bash
# Check routing table
ip route

# Example routing for same-node communication:
10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
```

**Interpretation:**
- Destination 10.244.1.0/24 (local pod subnet)
- Use `cni0` bridge
- Source from 10.244.1.1 (bridge IP)

#### Pod Routing Table

```bash
# Check pod routing
kubectl exec -it pod-a -- ip route

# Example:
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
10.244.1.0/24 dev eth0 scope link src 10.244.1.5
```

**Interpretation:**
- Default gateway is 169.254.1.1 (host)
- Local subnet is 10.244.1.0/24
- Pod IP is 10.244.1.5

### Practical Example

#### Scenario: Pod A Communicates with Pod B

```bash
# Create two pods on same node
kubectl run pod-a --image=nginx --restart=Never
kubectl run pod-b --image=busybox --restart=Never -- sleep 3600

# Get pod IPs
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

echo "Pod A IP: ${POD_A_IP}"
echo "Pod B IP: ${POD_B_IP}"

# Test communication from Pod B to Pod A
kubectl exec pod-b -- wget -qO- http://${POD_A_IP}
```

**Expected Output:**
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
</body>
</html>
```

#### Verify Communication

```bash
# Check connectivity
kubectl exec pod-b -- ping -c 3 ${POD_A_IP}

# Trace route (depends on CNI)
kubectl exec pod-b -- traceroute ${POD_A_IP}

# Check network interfaces
kubectl exec pod-a -- ip addr
kubectl exec pod-b -- ip addr
```

### Performance Characteristics

#### Latency

Same-node pod communication has very low latency:

```
Typical latencies:
- Without CNI overhead: < 0.1ms
- With bridge-based CNI: ~0.2-0.5ms
- With eBPF-based CNI (Cilium): ~0.1-0.3ms
```

#### Throughput

High throughput due to:

- Memory-to-memory transfer (mostly)
- No network traversals
- Minimal context switches

### Troubleshooting Same-Node Communication

#### Issue: Pods Cannot Communicate

**Symptoms:**
- Connection timeout
- Connection refused
- No route to host

**Diagnosis:**

```bash
# Check if pods are on same node
kubectl get pods -o wide

# Check pod IPs
kubectl get pods -o wide

# Test connectivity
kubectl exec pod-a -- ping pod-b-pod-ip

# Check bridge status
ip addr show cni0

# Check routing
ip route
```

**Common Causes:**
1. **Network Policy Blocking**
   ```bash
   # Check network policies
   kubectl get networkpolicy
   ```

2. **Bridge Misconfiguration**
   ```bash
   # Check bridge
   ip link show cni0
   ```

3. **Firewall Rules**
   ```bash
   # Check iptables
   iptables -L -n -v
   ```

---

## 3. Pod-to-Pod Across Nodes

### Understanding Cross-Node Communication

When pods are on different nodes, communication must traverse the physical network between nodes.

### Architecture Overview

```
┌─────────────────────────────────┐      ┌─────────────────────────────────┐
│          Node 1               │      │          Node 2               │
├─────────────────────────────────┤      ├─────────────────────────────────┤
│                                 │      │                                 │
│  ┌──────────────────┐          │      │  ┌──────────────────┐          │
│  │      Pod A      │          │      │  │      Pod B      │          │
│  │   10.244.1.5   │          │      │  │   10.244.2.5   │          │
│  │  eth0: 10.244.1.5 │          │      │  │  eth0: 10.244.2.5 │          │
│  └────────┬─────────┘          │      │  └────────┬─────────┘          │
│           │                     │      │           │                     │
│    ┌──────▼──────┐             │      │    ┌──────▼──────┐             │
│    │   veth0     │             │      │    │   veth1     │             │
│    └──────┬──────┘             │      │    └──────┬──────┘             │
│           │                     │      │           │                     │
│    ┌──────▼──────┐             │      │    ┌──────▼──────┐             │
│    │  cni0 bridge │             │      │    │  cni0 bridge │             │
│    │ 10.244.1.1  │             │      │    │ 10.244.2.1  │             │
│    └──────┬──────┘             │      │    └──────┬──────┘             │
│           │                     │      │           │                     │
│    ┌──────▼──────┐             │      │    ┌──────▼──────┐             │
│    │ Routing     │             │      │    │ Routing     │             │
│    │ Table       │             │      │    │ Table       │             │
│    └──────┬──────┘             │      │    └──────┬──────┘             │
│           │                     │      │           │                     │
│    ┌──────▼──────┐             │      │    ┌──────▼──────┐             │
│    │ Physical    │────────────│──────┼────│ Physical    │             │
│    │ Interface   │    192.168.1.11/24    │    │ Interface   │             │
│    │ eth0        │             │      │    │ eth0        │             │
│    └─────────────┘             │      │    └─────────────┘             │
└─────────────────────────────────┘      └─────────────────────────────────┘
         │                                          │
         └──────────────────────────────────────────────┘
                    Physical Network
```

### Communication Flow

#### Step-by-Step Process

1. **Pod A (Node 1) Sends Packet to Pod B (Node 2)**
   ```
   Source: 10.244.1.5
   Destination: 10.244.2.5
   ```

2. **Packet Leaves Pod A's Namespace**
   - Exits via `eth0`
   - Reaches Node 1's bridge

3. **Node 1 Routing**
   - Check routing table
   - Determine 10.244.2.0/24 is on Node 2
   - Route to physical interface

4. **Physical Network Transmission**
   - Packet encapsulated (depending on CNI)
   - Sent to Node 2's physical IP

5. **Node 2 Receives Packet**
   - Decapsulate (if encapsulated)
   - Route to appropriate pod

6. **Packet Enters Pod B's Namespace**
   - Arrives at Pod B's `eth0`
   - Application receives packet

### Cross-Node Routing Mechanisms

Different CNIs use different mechanisms for cross-node communication.

#### Method 1: Pure Routing (Calico - BGP)

```
┌─────────────────────────────────────────────────────────────┐
│                    Pure Routing (BGP)                    │
│                                                              │
│  Node 1                 Node 2                 Node 3         │
│  ┌────────┐              ┌────────┐              ┌────────┐        │
│  │Pod A   │              │Pod C   │              │Pod E   │        │
│  │10.244. │              │10.244. │              │10.244. │        │
│  │1.5      │              │2.5      │              │3.5      │        │
│  └────┬───┘              └────┬───┘              └────┬───┘        │
│       │                       │                       │               │
│  ┌────▼─────┐            ┌────▼─────┐            ┌────▼─────┐        │
│  │ Routing  │            │ Routing  │            │ Routing  │        │
│  │ Table    │            │ Table    │            │ Table    │        │
│  │10.244.2.│            │10.244.1.│            │10.244.2.│        │
│  │0/24 →   │            │0/24 →   │            │0/24 →   │        │
│  │Node 2    │            │Node 1    │            │Node 3    │        │
│  └────┬─────┘            └────┬─────┘            └────┬─────┘        │
│       │                       │                       │               │
│       └───────────┬───────────┴───────────┐               │
│                   │                           │               │
│            ┌──────▼───────────────────────▼──────┐            │
│            │      Physical Network           │            │
│            │    (No encapsulation)         │            │
│            └────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- No encapsulation overhead
- BGP protocol for route distribution
- Direct pod IP routing
- Best performance

**Advantages:**
- Lowest latency
- Highest throughput
- No tunnel overhead

**Disadvantages:**
- Requires BGP support in network
- More complex network setup
- May need underlying network support

#### Method 2: VXLAN Overlay (Flannel)

```
┌─────────────────────────────────────────────────────────────┐
│                  VXLAN Overlay                         │
│                                                              │
│  Node 1                 Node 2                 Node 3         │
│  ┌────────┐              ┌────────┐              ┌────────┐        │
│  │Pod A   │              │Pod C   │              │Pod E   │        │
│  │10.244. │              │10.244. │              │10.244. │        │
│  │1.5      │              │2.5      │              │3.5      │        │
│  └────┬───┘              └────┬───┘              └────┬───┘        │
│       │                       │                       │               │
│  ┌────▼─────────┐        ┌────▼─────────┐        ┌────▼─────────┐   │
│  │VXLAN Tunnel │        │VXLAN Tunnel │        │VXLAN Tunnel │   │
│  │IP: 10.244. │        │IP: 10.244. │        │IP: 10.244. │   │
│  │1.0/24     │        │2.0/24     │        │3.0/24     │   │
│  └────┬─────────┘        └────┬─────────┘        └────┬─────────┘   │
│       │                       │                       │               │
│  ┌────▼─────────┐        ┌────▼─────────┐        ┌────▼─────────┐   │
│  │Physical     │        │Physical     │        │Physical     │   │
│  │Interface    │        │Interface    │        │Interface    │   │
│  │192.168.1.11│       │192.168.1.12│       │192.168.1.13│   │
│  └────┬─────────┘        └────┬─────────┘        └────┬─────────┘   │
│       │                       │                       │               │
│       └───────────┬───────────┴───────────┐               │
│                   │                           │               │
│            ┌──────▼───────────────────────▼──────┐            │
│            │      Physical Network           │            │
│            │  (Encapsulated traffic)        │            │
│            └────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- VXLAN encapsulation
- UDP port 4789
- Overlay network
- Works with any underlying network

**Packet Flow:**
```
Pod A → Pod B:
1. Pod A sends packet to 10.244.2.5
2. Node 1 encapsulates in VXLAN
   - Outer: 192.168.1.11 → 192.168.1.12
   - Inner: 10.244.1.5 → 10.244.2.5
3. Physical network transmission
4. Node 2 decapsulates
5. Deliver to Pod B
```

**Advantages:**
- Works with any network
- No special network requirements
- Easy to set up

**Disadvantages:**
- Encapsulation overhead (~50 bytes)
- Higher latency
- Lower throughput

#### Method 3: eBPF-Based (Cilium)

```
┌─────────────────────────────────────────────────────────────┐
│               eBPF-Based (Cilium)                       │
│                                                              │
│  Node 1                 Node 2                 Node 3         │
│  ┌────────┐              ┌────────┐              ┌────────┐        │
│  │Pod A   │              │Pod C   │              │Pod E   │        │
│  │10.244. │              │10.244. │              │10.244. │        │
│  │1.5      │              │2.5      │              │3.5      │        │
│  └────┬───┘              └────┬───┘              └────┬───┘        │
│       │                       │                       │               │
│  ┌────▼───────────────────────────▼────────────────────┐        │
│  │           eBPF Programs (TC hooks)              │        │
│  │  - Policy enforcement                              │        │
│  │  - Connection tracking                             │        │
│  │  - Direct routing                                 │        │
│  └────┬────────────────────────────────────────────┘        │
│       │                       │                       │               │
│  ┌────▼─────────┐        ┌────▼─────────┐        ┌────▼─────────┐   │
│  │Physical     │        │Physical     │        │Physical     │   │
│  │Interface    │        │Interface    │        │Interface    │   │
│  │192.168.1.11│       │192.168.1.12│       │192.168.1.13│   │
│  └────┬─────────┘        └────┬─────────┘        └────┬─────────┘   │
│       │                       │                       │               │
│       └───────────┬───────────┴───────────┐               │
│                   │                           │               │
│            ┌──────▼───────────────────────▼──────┐            │
│            │      Physical Network           │            │
│            │  (Direct routing, optional    │            │
│            │   VXLAN for compatibility)    │            │
│            └────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- eBPF in kernel for performance
- Can use direct routing or VXLAN
- Layer 3-7 awareness
- Identity-based networking

**Advantages:**
- Highest performance (eBPF)
- Flexible (L3-L7 policies)
- Advanced observability
- Identity-based security

**Disadvantages:**
- Requires newer kernel (4.19+)
- More complex setup

### Routing Table Example

#### Node 1 Routing Table

```bash
# On Node 1
ip route

# Example output:
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.11
10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
10.244.2.0/24 via 192.168.1.12 dev eth0 proto bird onlink
10.244.3.0/24 via 192.168.1.13 dev eth0 proto bird onlink
```

**Interpretation:**
- Local pods: 10.244.1.0/24 (via cni0 bridge)
- Remote pods: Via other nodes (192.168.1.12, 192.168.1.13)

### Practical Example

#### Scenario: Pod A (Node 1) Communicates with Pod B (Node 2)

```bash
# Create pods on different nodes
kubectl run pod-a --image=nginx --restart=Never --labels=node=1
kubectl run pod-b --image=busybox --restart=Never --sleep 3600 --labels=node=2

# Check which nodes pods are on
kubectl get pods -o wide

# Get pod IPs
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

echo "Pod A IP: ${POD_A_IP} (Node 1)"
echo "Pod B IP: ${POD_B_IP} (Node 2)"

# Test communication
kubectl exec pod-b -- wget -qO- http://${POD_A_IP}
```

#### Verify Cross-Node Communication

```bash
# Trace route (depends on CNI)
kubectl exec pod-b -- traceroute ${POD_A_IP}

# Ping test
kubectl exec pod-b -- ping -c 3 ${POD_A_IP}

# Check latency
kubectl exec pod-b -- ping -c 10 ${POD_A_IP} | tail -1
```

### Performance Comparison

#### Latency Comparison

| CNI Type | Same-Node | Cross-Node | Overhead |
|----------|-----------|------------|----------|
| **Calico (BGP)** | ~0.2ms | ~1-2ms | Low |
| **Flannel (VXLAN)** | ~0.3ms | ~2-3ms | Medium |
| **Cilium (eBPF)** | ~0.1ms | ~1-2ms | Very Low |
| **Weave Net** | ~0.3ms | ~2-4ms | Medium-High |

#### Throughput Comparison

| CNI Type | Throughput | Limiting Factor |
|----------|-----------|----------------|
| **Calico (BGP)** | ~40 Gbps | Physical network |
| **Flannel (VXLAN)** | ~30 Gbps | Encapsulation |
| **Cilium (eBPF)** | ~45 Gbps | Physical network |
| **Weave Net** | ~25 Gbps | Encryption + Encapsulation |

### Troubleshooting Cross-Node Communication

#### Issue: Cross-Node Communication Fails

**Symptoms:**
- Timeout connecting to pods on other nodes
- "No route to host" errors
- Inconsistent connectivity

**Diagnosis:**

```bash
# 1. Check node network connectivity
# Get node IPs
kubectl get nodes -o wide

# SSH to a node and test
ping <other-node-ip>

# 2. Check routing tables
# On Node 1
ip route | grep 10.244

# On Node 2
ip route | grep 10.244

# 3. Check CNI logs
kubectl logs -n kube-system -l k8s-app=<cni-name> --tail=100

# 4. Check network policies
kubectl get networkpolicy -A

# 5. Test with Hubble (if using Cilium)
hubble observe --verdict dropped
```

**Common Causes:**
1. **Missing Routes**
   - Routes not propagated
   - BGP peering issues (Calico)
   - VXLAN tunnel issues (Flannel)

2. **Network Policies Blocking**
   - Default deny policies
   - Cross-namespace restrictions

3. **Firewall Rules**
   - Port 4789 blocked (VXLAN)
   - BGP port 179 blocked (Calico)
   - eBPF communication blocked (Cilium)

4. **MTU Issues**
   - Fragmentation due to encapsulation
   - MTU mismatch

---

## 4. Internal Communication

### Understanding Internal Communication Patterns

Internal communication in Kubernetes follows several patterns depending on the use case.

### Communication Patterns

#### 1. Direct Pod-to-Pod

```
┌────────┐              ┌────────┐
│ Pod A  │ ────────────→ │ Pod B  │
│ Client │              │ Server │
└────────┘              └────────┘
```

**Use Cases:**
- Microservices communication
- Internal API calls
- Direct service dependencies

**Pros:**
- Fast (direct IP)
- Low latency
- No service abstraction overhead

**Cons:**
- Pod IP changes (not stable)
- No load balancing
- Tight coupling

#### 2. Service-Based Communication

```
┌────────┐              ┌──────────────┐              ┌────────┐
│ Pod A  │ ────────────→ │  Service    │ ────────────→ │ Pod B  │
│ Client │              │  (ClusterIP)│              │ Server │
└────────┘              └──────────────┘              └────────┘
```

**Use Cases:**
- Stable endpoint
- Load balancing
- Service discovery

**Pros:**
- Stable IP/hostname
- Automatic load balancing
- Service abstraction

**Cons:**
- Extra hop (service → pod)
- Slightly higher latency

#### 3. DNS-Based Communication

```
┌────────┐              ┌──────────┐              ┌────────┐
│ Pod A  │ ────────────→ │ CoreDNS  │ ────────────→ │ Pod B  │
│ Client │    DNS:     │          │              │ Server │
└────────┘  backend   └──────────┘              └────────┘
            .default
            .svc.cluster.local
```

**Use Cases:**
- Service discovery by name
- Environment-based routing
- Service aliases

**Pros:**
- Human-readable names
- Automatic service discovery
- Namespace-aware

**Cons:**
- DNS resolution latency
- DNS caching issues

#### 4. Ingress/Egress Communication

```
┌─────────┐     ┌─────────────┐     ┌─────────┐
│External │ ──→ │ Ingress     │ ──→ │ Pods    │
│Clients  │     │ Controller  │     │         │
└─────────┘     └─────────────┘     └─────────┘

┌─────────┐     ┌─────────────┐     ┌─────────┐
│ Pods    │ ──→ │ Egress      │ ──→ │External │
│         │     │ Controller  │     │Services │
└─────────┘     └─────────────┘     └─────────┘
```

**Use Cases:**
- External access to services
- Controlling outbound traffic
- API gateway

### Communication Flow Examples

#### Example 1: Microservices Architecture

```
┌──────────┐
│Frontend  │
│  Pod     │
└────┬─────┘
     │
     ↓ DNS: api-service.default.svc.cluster.local
     ↓
┌─────────────┐
│ API Service │ (Load balancing)
│ ClusterIP  │
└────┬────────┘
     ↓
┌──────────┐
│API Pod 1 │
│Backend   │
└──────────┘
     ↓
     ↓ DNS: database.default.svc.cluster.local
     ↓
┌─────────────┐
│ DB Service │
│ ClusterIP  │
└────┬────────┘
     ↓
┌──────────┐
│DB Pod    │
│Database  │
└──────────┘
```

#### Example 2: Multi-Tier Application

```
┌──────────┐
│Web Tier │
│Multiple  │
│Pods     │
└────┬─────┘
     │
     ↓
┌─────────────┐
│ Web Service│
└────┬────────┘
     │
     ↓
┌──────────┐
│App Tier │
│Multiple  │
│Pods     │
└────┬─────┘
     │
     ↓
┌─────────────┐
│ App Service│
└────┬────────┘
     │
     ↓
┌──────────┐
│Data Tier│
│Multiple  │
│Pods     │
└──────────┘
```

### Communication Protocols

#### Common Protocols in Kubernetes

| Protocol | Use Case | Default Port |
|----------|-----------|--------------|
| **HTTP/HTTPS** | Web services, APIs | 80, 443 |
| **gRPC** | Microservices | Varies |
| **TCP** | Custom protocols | Varies |
| **UDP** | DNS, custom protocols | Varies |
| **WebSocket** | Real-time | 80/443 |

#### Protocol-Specific Considerations

**HTTP/HTTPS:**
- Stateful connections
- Headers for routing
- Load balancing by connection

**gRPC:**
- HTTP/2 based
- Streaming support
- Service mesh integration

**TCP:**
- Connection-based
- Custom protocols
- Requires health checks

**UDP:**
- Connectionless
- DNS queries
- Streaming protocols

### Connection Pooling

#### Client-Side Connection Pooling

```
┌─────────────────────────────────────────────────────────┐
│                  Pod A (Client)                      │
│                                                           │
│  ┌───────────────────────────────────────────────────┐    │
│  │           Connection Pool                        │    │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │    │
│  │  │ Conn│ │ Conn│ │ Conn│ │ Conn│    │    │
│  │  │  1  │ │  2  │ │  3  │ │  4  │    │    │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘    │    │
│  └─────┼───────┼───────┼───────┼──────┘    │
│        │       │       │       │              │
│        └───────┴───────┴───────┘              │
│                    │                             │
└────────────────────┼─────────────────────────────┘
                     │
                     │
                ┌────▼────┐
                │ Service │
                └─────────┘
```

**Benefits:**
- Reduced connection overhead
- Better performance
- Resource efficiency

**Implementation:**
```python
# Example: HTTP connection pool in Python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Create session with connection pool
session = requests.Session()

# Configure connection pool
adapter = HTTPAdapter(
    pool_connections=10,
    pool_maxsize=20,
    max_retries=Retry(total=3)
)
session.mount('http://', adapter)
session.mount('https://', adapter)

# Use session for multiple requests
response1 = session.get('http://api-service:8080/data')
response2 = session.get('http://api-service:8080/users')
```

### Keep-Alive Connections

#### TCP Keep-Alive

```
Client ─────────────────────────────────→ Server
         ↓ Keep-Alive packets         ↑
```

**Benefits:**
- Faster subsequent requests
- Reduced connection overhead
- Better resource utilization

**Configuration:**
```yaml
# Keep-Alive in Nginx
upstream backend {
    server api-service:8080;
    keepalive 32;  # Maintain 32 connections
}

server {
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

---

## 5. Network Policies

### What are Network Policies?

**Network Policies** are Kubernetes resources that control traffic flow between pods and network endpoints. They act like a **firewall** for pods.

### Default Behavior

#### Default Allow

If no network policies exist, **all traffic is allowed**:

```
All Pods:
  Ingress: ✅ Allowed from anywhere
  Egress: ✅ Allowed to anywhere
```

**Security Risk:**
- No traffic restrictions
- Pods can reach any other pod
- External access unrestricted

#### Default Deny

After applying the first network policy to a namespace:

```
Namespace with Network Policies:
  Ingress: ❌ Denied (unless explicitly allowed)
  Egress: ❌ Denied (unless explicitly allowed)
```

**Security Benefit:**
- Zero-trust model
- Explicitly allowed traffic only
- Defense in depth

### Network Policy Structure

#### YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  podSelector:              # Which pods does this apply to?
    matchLabels:
      app: nginx
  policyTypes:              # What traffic types?
  - Ingress
  - Egress
  ingress:                  # Ingress rules
  - from:                   # Who can send traffic?
    - podSelector:           # From specific pods
        matchLabels:
          role: frontend
    - ipBlock:              # From specific IPs
      - 192.168.1.0/24
    ports:                   # Which ports?
    - protocol: TCP
      port: 80
  egress:                   # Egress rules
  - to:                     # Where can traffic go?
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
```

### Key Concepts

#### 1. podSelector

Selects which pods the policy applies to:

```yaml
podSelector:
  matchLabels:
    app: nginx        # Single label
    tier: frontend     # Multiple labels

# Or match all pods
podSelector: {}

# Or complex selector
podSelector:
  matchExpressions:
  - key: app
    operator: In
    values: [nginx, apache]
  - key: env
    operator: NotIn
    values: [production]
```

#### 2. policyTypes

Specifies which direction(s) to control:

```yaml
# Ingress only
policyTypes:
- Ingress

# Egress only
policyTypes:
- Egress

# Both directions
policyTypes:
- Ingress
- Egress
```

#### 3. ingress Rules

Control incoming traffic:

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - protocol: TCP
    port: 80
```

#### 4. egress Rules

Control outgoing traffic:

```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        app: database
  ports:
  - protocol: TCP
    port: 5432
```

### Policy Examples

#### Example 1: Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # All pods
  policyTypes:
  - Ingress
  ingress: []  # No ingress rules = deny all
```

**Effect:**
- All pods in `production` namespace
- Cannot receive traffic from any source
- Outgoing traffic still allowed

#### Example 2: Allow Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-from-frontend
  namespace: default
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
    - protocol: TCP
      port: 443
```

**Effect:**
- Only pods with label `app=frontend`
- Can access pods with label `app=nginx`
- Only on ports 80 and 443

#### Example 3: Allow DNS

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}  # All pods
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}  # Any namespace
    ports:
    - protocol: UDP
      port: 53
```

**Effect:**
- All pods can make DNS queries
- UDP port 53 to any namespace
- Required for pod DNS resolution

#### Example 4: Allow External Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
      - 8.8.8.8/32  # Google DNS
      - 1.1.1.1/32  # Cloudflare DNS
    ports:
    - protocol: UDP
      port: 53
```

**Effect:**
- Pods with `app=backend`
- Can reach specific external IPs
- Only on UDP port 53

### Cilium Extended Policies

Cilium extends Kubernetes NetworkPolicy with additional capabilities.

#### L7 Policy (HTTP)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: http-policy
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

**Capabilities:**
- HTTP method filtering
- Path-based rules
- Header-based rules

#### DNS Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  - toFQDNs:
    - matchName: "api.example.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

**Capabilities:**
- FQDN-based egress
- DNS name resolution
- Dynamic IP handling

### Policy Best Practices

#### 1. Start with Default Deny

```yaml
# Step 1: Apply default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### 2. Add Essential Policies

```yaml
# Step 2: Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

#### 3. Add Application-Specific Policies

```yaml
# Step 3: Allow frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### 4. Use Labels, Not IPs

❌ **Bad:**
```yaml
to:
- ipBlock:
  - 10.244.1.5/32  # Pod IP changes!
```

✅ **Good:**
```yaml
to:
- podSelector:
    matchLabels:
      app: database  # Stable label
```

### Troubleshooting Network Policies

#### Check Policy Status

```bash
# List all policies
kubectl get networkpolicy -A

# Describe a policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check if policy is applied
kubectl get pods -n <namespace> --show-labels
```

#### Debug Dropped Traffic

```bash
# With Cilium/Hubble
hubble observe --verdict dropped

# Check policy enforcement
cilium policy get --labels app=nginx

# View policy details
cilium networkpolicy get <policy-id>
```

#### Test Policy

```bash
# Create test pods
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Test connectivity
wget -O- http://<target-pod-ip>:<port>

# Check logs
kubectl logs -n kube-system -l k8s-app=<cni-name>
```

---

## 6. Services

### What are Kubernetes Services?

A **Service** is an abstraction that defines a logical set of pods and a policy to access them. Services provide:
- **Stable IP address** (ClusterIP)
- **DNS name** for service discovery
- **Load balancing** across pods
- **Service discovery** mechanism

### Service Types

#### 1. ClusterIP (Default)

```
┌─────────────┐
│   Service  │
│ ClusterIP  │ 10.96.123.45
└──────┬──────┘
       │
       ├──→ Pod 1 (10.244.1.5)
       ├──→ Pod 2 (10.244.1.6)
       └──→ Pod 3 (10.244.2.5)
```

**Characteristics:**
- Internal cluster IP only
- Not accessible from outside
- Load balances across pods
- Default service type

**Use Cases:**
- Internal microservices
- Backend services
- Internal APIs

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

#### 2. NodePort

```
┌─────────────┐
│   Service  │
│  NodePort  │
└──────┬──────┘
       │
       ├──→ Pod 1
       ├──→ Pod 2
       └──→ Pod 3
       │
       │ Exposed on all nodes
       │ Port: 30001
       ↓
┌───────────────────────────────┐
│ Node 1  Node 2  Node 3   │
│ :30001   :30001   :30001   │
└───────────────────────────────┘
```

**Characteristics:**
- Exposes service on each node's IP
- Static port (30000-32767 range)
- External access via `<NodeIP>:<NodePort>`
- Not recommended for production

**Use Cases:**
- Testing
- Development
- Simple external access

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30001
```

#### 3. LoadBalancer

```
                    ┌─────────────┐
                    │   Service  │
                    │LoadBalancer │
                    └──────┬──────┘
                           │
                    ┌──────────┴──────────┐
                    │                     │
                    ↓                     ↓
              ┌─────────┐          ┌─────────┐
              │  LB 1  │          │  LB 2  │
              └────┬────┘          └────┬────┘
                   │                   │
              ┌────┴───────────────────┴────┐
              │     Traffic to Nodes       │
              └──────┬───────┬───────┬───┘
                     ↓       ↓       ↓
                 ┌──────┐ ┌──────┐ ┌──────┐
                 │Node 1│ │Node 2│ │Node 3│
                 └──────┘ └──────┘ └──────┘
```

**Characteristics:**
- Creates external load balancer
- Cloud provider integration
- Automatic IP allocation
- Production-ready

**Use Cases:**
- Production deployments
- Cloud-native applications
- High availability

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

#### 4. ExternalName

```
┌─────────────┐
│   Service  │
│ExternalName │
└──────┬──────┘
       │
       │ DNS CNAME
       ↓
┌─────────────────────┐
│ external.example.com│
└─────────────────────┘
```

**Characteristics:**
- Maps to external DNS name
- No proxying
- DNS CNAME record
- Useful for external services

**Use Cases:**
- Accessing external databases
- Third-party service integration
- Legacy system integration

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: default
spec:
  type: ExternalName
  externalName: database.example.com
```

### Service Discovery

#### DNS Resolution

Kubernetes services are discoverable via DNS:

```
Service DNS Format:
<service-name>.<namespace>.svc.cluster.local

Examples:
backend-service.default.svc.cluster.local
api-service.production.svc.cluster.local
```

#### DNS Query Flow

```
┌──────────┐
│   Pod    │
│ (Client) │
└────┬─────┘
     │
     │ DNS Query: backend-service.default.svc.cluster.local
     ↓
┌─────────────┐
│  CoreDNS   │
│  Service   │
└────┬───────┘
     │
     │ DNS Response: 10.96.123.45
     ↓
┌──────────┐
│   Pod    │
│ (Client) │
└────┬─────┘
     │
     │ Connect to 10.96.123.45
     ↓
┌─────────────┐
│  Service   │
│ ClusterIP  │
└────┬───────┘
     │
     │ Load balance
     ↓
┌──────────┐
│   Pod    │
│ (Server) │
└──────────┘
```

### Service Endpoints

#### What are Endpoints?

Endpoints are the actual pod IPs that a service routes traffic to:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: backend-service
  namespace: default
subsets:
- addresses:
  - ip: 10.244.1.5
    targetRef:
      name: backend-pod-1
  - ip: 10.244.1.6
    targetRef:
      name: backend-pod-2
  - ip: 10.244.2.5
    targetRef:
      name: backend-pod-3
  ports:
  - name: http
    port: 8080
    protocol: TCP
```

#### View Endpoints

```bash
# List endpoints
kubectl get endpoints

# Describe specific service
kubectl describe service backend-service

# View endpoints in detail
kubectl get endpoints backend-service -o yaml
```

### Service Load Balancing

#### kube-proxy Role

kube-proxy is the component that implements service load balancing:

```
┌─────────────────────────────────────────────────────────┐
│                   kube-proxy                        │
│  (Runs on each node)                              │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          Load Balancing Methods               │    │
│  │                                                 │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐    │    │
│  │  │ iptables│ │IPVS     │ │ eBPF    │    │    │
│  │  │         │ │         │ │         │    │    │
│  │  │ Rules   │ │ Virtual  │ │ Kernel  │    │    │
│  │  │         │ │ Server  │ │         │    │    │
│  │  └─────────┘ └─────────┘ └─────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

#### Load Balancing Algorithms

**Random (iptables):**
- Randomly selects pod
- Simple implementation
- Basic distribution

**Round Robin (IPVS):**
- Sequential selection
- Better distribution
- Health checks

**Maglev (eBPF - Cilium):**
- Consistent hashing
- Minimal rehashing
- High performance

### Service Examples

#### Example 1: Web Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
```

#### Example 2: Multi-Port Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
  - name: metrics
    port: 9091
    targetPort: 9091
```

#### Example 3: Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
  namespace: default
spec:
  type: ClusterIP
  clusterIP: None  # Headless service
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

**Headless Service:**
- No ClusterIP
- DNS returns pod IPs directly
- Used with StatefulSets
- Each pod gets its own DNS record

### Service Best Practices

#### 1. Use Meaningful Service Names

✅ **Good:**
```yaml
metadata:
  name: backend-api-service
```

❌ **Bad:**
```yaml
metadata:
  name: svc-123
```

#### 2. Add Labels for Discovery

```yaml
metadata:
  name: backend-service
  labels:
    app: backend
    tier: backend
    environment: production
```

#### 3. Document Ports

```yaml
metadata:
  name: backend-service
  annotations:
    description: "Backend API service"
    documentation: "https://docs.example.com/api"
spec:
  ports:
  - name: http-api
    port: 80
    targetPort: 8080
```

#### 4. Use Selectors Correctly

```yaml
selector:
  app: backend        # Specific
  tier: backend       # Additional context
  environment: prod   # Environment-specific
```

### Troubleshooting Services

#### Check Service Status

```bash
# List services
kubectl get services

# Describe service
kubectl describe service <service-name>

# Check endpoints
kubectl get endpoints <service-name>

# Check service DNS
kubectl run test --rm -it --image=busybox -- nslookup <service-name>.<namespace>.svc.cluster.local
```

#### Test Service Connectivity

```bash
# Create test pod
kubectl run test --rm -it --image=curlimages/curl -- sh

# Test service
curl http://<service-name>.<namespace>.svc.cluster.local

# Test specific port
curl http://<service-name>:<port>

# Test with verbose output
curl -v http://<service-name>:<port>
```

#### Debug Load Balancing

```bash
# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100

# Check iptables rules
iptables -L -n -v | grep <service-ip>

# Check IPVS (if using IPVS)
ipvsadm -Ln
```

---

## 7. Pod DNS

### Kubernetes DNS Architecture

Kubernetes uses **CoreDNS** for DNS resolution within the cluster.

```
┌─────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              CoreDNS Deployment            │    │
│  │                                                      │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │    │
│  │  │CoreDNS 1│ │CoreDNS 2│ │CoreDNS 3│     │    │
│  │  └────┬────┘ └────┬────┘ └────┬────┘     │    │
│  └───────┼────────────┼────────────┼──────────┘    │
│          │            │            │                │
│          └────────────┴────────────┘                │
│                      │                             │
│              ┌───────▼────────────┐                │
│              │  CoreDNS Service  │                │
│              │  ClusterIP:      │                │
│              │  10.96.0.10      │                │
│              └───────────────────┘                │
│                                                       │
│  ┌───────────────────────────────────────────────────┐   │
│  │                  Pods                         │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐      │   │
│  │  │ Pod A  │ │ Pod B  │ │ Pod C  │      │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘      │   │
│  └──────┼────────────┼────────────┼───────────┘   │
│         │            │            │                  │
│         └────────────┴────────────┘                  │
│                      │                               │
│              DNS Queries to 10.96.0.10                │
└───────────────────────────────────────────────────────────────┘
```

### DNS Resolution Flow

#### Step-by-Step Process

1. **Application Makes DNS Query**
   ```
   Pod A → DNS Query: backend-service.default.svc.cluster.local
   ```

2. **Query Goes to Cluster DNS**
   - Resolv.conf points to 10.96.0.10
   - Query sent to CoreDNS service

3. **CoreDNS Resolves Name**
   - Looks up service in Kubernetes API
   - Returns ClusterIP: 10.96.123.45

4. **Response Returned to Pod**
   - Pod receives IP address
   - Application can now connect

### DNS Record Types

#### 1. Service DNS

```
Format: <service-name>.<namespace>.svc.cluster.local

Examples:
- backend-service.default.svc.cluster.local
- api-service.production.svc.cluster.local
- web-service.staging.svc.cluster.local
```

#### 2. Pod DNS

```
Format: <pod-ip-address>.<namespace>.pod.cluster.local

Examples:
- 10-244-1-5.default.pod.cluster.local
- 10-244-2-5.production.pod.cluster.local
```

#### 3. Headless Service DNS

```
Format: <statefulset-name>.<namespace>.svc.cluster.local

Returns: A records for all pods

Examples:
- mongo-0.mongo.default.svc.cluster.local
- mongo-1.mongo.default.svc.cluster.local
- mongo-2.mongo.default.svc.cluster.local
```

### Pod DNS Configuration

#### /etc/resolv.conf

```bash
# Check pod DNS configuration
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

**Example Output:**
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

**Fields:**
- `search`: DNS search domains
- `nameserver`: DNS server IP (CoreDNS)
- `ndots`: Number of dots before absolute name

#### Custom DNS Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: "None"  # Override cluster DNS
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - default.svc.cluster.local
    - mycompany.com
  containers:
  - name: app
    image: nginx
```

### DNS Policies

#### Default DNS Policy

```yaml
spec:
  dnsPolicy: ClusterFirst  # Default
```

**Behavior:**
- Use cluster DNS first
- Fall back to configured nameservers
- Recommended for most pods

#### DNS Policy Options

**ClusterFirst (Default):**
```yaml
dnsPolicy: ClusterFirst
```
- Use CoreDNS
- Fall back to /etc/resolv.conf

**Default:**
```yaml
dnsPolicy: Default
```
- Inherit node's DNS configuration

**None:**
```yaml
dnsPolicy: None
```
- Skip cluster DNS
- Use custom DNS config

**ClusterFirstWithHostNet:**
```yaml
dnsPolicy: ClusterFirstWithHostNet
```
- For pods with host network
- Use cluster DNS

### DNS Troubleshooting

#### Check DNS Resolution

```bash
# Test DNS resolution
kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Test service DNS
kubectl exec -it <pod-name> -- nslookup backend-service.default.svc.cluster.local

# Test external DNS
kubectl exec -it <pod-name> -- nslookup google.com
```

#### Check CoreDNS

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml
```

#### Common DNS Issues

**Issue 1: DNS Resolution Fails**

```bash
# Check if DNS server is reachable
kubectl exec -it <pod-name> -- ping 10.96.0.10

# Check DNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check network policies
kubectl get networkpolicy -A
```

**Issue 2: Slow DNS Resolution**

```bash
# Check DNS caching
kubectl exec -it <pod-name> -- cat /etc/resolv.conf

# Test multiple queries
time kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Check CoreDNS performance
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i latency
```

**Issue 3: DNS Timeouts**

```bash
# Check CoreDNS pod status
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check DNS timeouts
kubectl exec -it <pod-name> -- nslookup -timeout=1 google.com

# Check network connectivity
kubectl exec -it <pod-name> -- telnet 10.96.0.10 53
```

---

## 8. Service Mesh

### What is a Service Mesh?

A **Service Mesh** is a dedicated infrastructure layer for managing service-to-service communication within a microservices architecture.

### Why Service Mesh?

#### Challenges Without Service Mesh

```
┌─────────────────────────────────────────────────────────────┐
│                 Without Service Mesh                 │
│                                                              │
│  Service A ──────────→ Service B                       │
│       │                   │                              │
│       │                   │                              │
│       └───┼───────────────┘                              │
│           │                                             │
│  Issues:                                               │
│  ❌ No observability                                    │
│  ❌ No traffic management                              │
│  ❌ No security                                      │
│  ❌ No retry logic                                    │
│  ❌ No circuit breaking                               │
│  ❌ No tracing                                      │
└─────────────────────────────────────────────────────────────┘
```

#### Benefits With Service Mesh

```
┌─────────────────────────────────────────────────────────────┐
│                 With Service Mesh                   │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐               │
│  │  Service A   │      │  Service B   │               │
│  │              │      │              │               │
│  │  ┌────────┐  │      │  ┌────────┐  │               │
│  │  │ Sidecar│  │──────│  │ Sidecar│  │               │
│  │  └────────┘  │      │  └────────┘  │               │
│  └──────────────┘      └──────────────┘               │
│           │                   │                              │
│           └───────────┬───────┘                              │
│                       │                                     │
│              ┌────────▼────────┐                             │
│              │  Service Mesh   │                             │
│              │  (Istio/Cilium)│                             │
│              └─────────────────┘                             │
│                                                              │
│  Benefits:                                               │
│  ✅ Observability (metrics, logs, traces)                │
│  ✅ Traffic management (load balancing, routing)          │
│  ✅ Security (mTLS, auth)                           │
│  ✅ Resilience (retries, timeouts, circuit breaking)  │
│  ✅ Policy enforcement                                │
└─────────────────────────────────────────────────────────────┘
```

### Service Mesh Architecture

#### Components

**1. Data Plane (Sidecars)**
- Proxy running alongside each service
- Handles all ingress/egress traffic
- Enforces policies

**2. Control Plane**
- Manages sidecars
- Provides configuration
- Collects telemetry

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                 Service Mesh Architecture              │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐               │
│  │  Service A   │      │  Service B   │               │
│  │              │      │              │               │
│  │  ┌────────┐  │      │  ┌────────┐  │               │
│  │  │Sidecar │  │──────│  │Sidecar │  │               │
│  │  │Proxy   │  │      │  │Proxy   │  │               │
│  │  └────────┘  │      │  └────────┘  │               │
│  └──────┬───────┘      └──────┬───────┘               │
│         │                      │                            │
│         └──────────┬───────────┘                            │
│                    │                                       │
│        ┌───────────▼────────────┐                           │
│        │      Control Plane    │                           │
│        │  (Istiod / Cilium) │                           │
│        └───────────┬────────────┘                           │
│                    │                                       │
│        ┌───────────▼────────────┐                           │
│        │    Observability     │                           │
│        │ (Prometheus, Grafana,│                           │
│        │  Jaeger, Kiali)   │                           │
│        └───────────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### Popular Service Mesh Solutions

#### 1. Istio

**Architecture:**
- Envoy sidecars (data plane)
- Istiod control plane
- Kubernetes CRDs for configuration

**Features:**
- Traffic management
- Security (mTLS)
- Observability
- Policy enforcement

**Pros:**
- Feature-rich
- Large community
- Kubernetes-native

**Cons:**
- Complex setup
- Resource overhead
- Steep learning curve

#### 2. Linkerd

**Architecture:**
- Rust-based proxy
- Lightweight
- Simple setup

**Features:**
- mTLS
- Observability
- Traffic splitting
- Retry logic

**Pros:**
- Simple to use
- Low resource usage
- Good performance

**Cons:**
- Fewer features than Istio
- Less traffic management

#### 3. Cilium Service Mesh

**Architecture:**
- eBPF-based (no sidecars)
- Direct kernel integration
- Identity-based networking

**Features:**
- Zero sidecar overhead
- L7 visibility
- Network policies
- Built-in observability

**Pros:**
- Highest performance
- No sidecar overhead
- Integrated with CNI
- Hubble for observability

**Cons:**
- Requires eBPF support
- Newer solution
- Less mature than Istio

#### Comparison Table

| Feature | Istio | Linkerd | Cilium |
|----------|--------|----------|----------|
| **Sidecar** | Envoy | Rust-based | None (eBPF) |
| **Performance** | Good | Very Good | Excellent |
| **Complexity** | High | Low | Medium |
| **mTLS** | Yes | Yes | Yes |
| **Traffic Split** | Yes | Yes | Yes |
| **Observability** | Excellent | Good | Excellent |
| **K8s Native** | Yes | Yes | Yes |
| **Resource Usage** | High | Low | Very Low |

### Service Mesh Features

#### 1. Traffic Management

**Load Balancing:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: backend-service
spec:
  host: backend-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # Round robin, least conn, random
```

**Traffic Splitting:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: backend-service
spec:
  hosts:
  - backend-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: backend-service
        subset: v2
      weight: 10  # 10% to v2
    - destination:
        host: backend-service
        subset: v1
      weight: 90  # 90% to v1
```

**Circuit Breaking:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: backend-service
spec:
  host: backend-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

#### 2. Security

**mTLS:**
- Automatic mutual TLS between services
- Certificate rotation
- Zero config needed

**Service-to-Service Auth:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: backend-service
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: STRICT  # or PERMISSIVE
```

**Request Auth:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  selector:
    matchLabels:
      app: api
  jwtRules:
  - issuer: "https://auth.example.com"
    jwks: "https://auth.example.com/jwks.json"
```

#### 3. Observability

**Metrics:**
- Request rate
- Success rate
- Latency (p50, p95, p99)
- Error rate

**Tracing:**
- Distributed tracing
- Request flow
- Performance analysis

**Logging:**
- Access logs
- Error logs
- Security events

### Cilium Service Mesh

#### Why Cilium Service Mesh?

**Traditional Service Mesh:**
```
Service → Sidecar → Network → Sidecar → Service
          ↓          ↓          ↓
        High      Medium     High
      Latency    Latency    Latency
```

**Cilium Service Mesh:**
```
Service → eBPF → Network → eBPF → Service
          ↓       ↓         ↓
         Very    Very     Very
        Low      Low      Low
      Latency   Latency   Latency
```

#### Features

**1. Layer 7 Visibility:**
- HTTP, gRPC, Kafka visibility
- No sidecar instrumentation
- Kernel-level observability

**2. Identity-Based:**
- Pod identities for security
- Fine-grained policies
- No IP-based rules

**3. High Performance:**
- eBPF in kernel
- No sidecar overhead
- Minimal latency

**4. Integrated Observability:**
- Hubble for visualization
- Built-in metrics
- Flow tracing

### Service Mesh Best Practices

#### 1. Start Simple

```bash
# Install with basic configuration
cilium install --set mesh.enabled=true

# Enable basic features only
cilium hubble enable
```

#### 2. Gradual Rollout

```yaml
# Use traffic splitting
apiVersion: cilium.io/v2alpha1
kind: CiliumNetworkPolicy
metadata:
  name: traffic-split
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        app: backend
    toPorts:
    - ports:
      - port: "8080"
      rules:
        l7:
          - weight: 90  # 90% to v1
          l7:
            http:
              path: /v1/*
          - weight: 10  # 10% to v2
          l7:
            http:
              path: /v2/*
```

#### 3. Monitor Everything

```bash
# Observe all flows
hubble observe

# Check service mesh metrics
cilium metrics list

# View service graph
cilium service graph
```

---

## 9. Key Takeaways (Architect Notes)

### Kubernetes Network Model

- **Flat network**: Pods communicate directly without NAT
- **IP-per-pod**: Each pod has a unique IP
- **No port mapping**: Applications use standard ports
- **CNI agnostic**: Different CNIs implement the same model

### Communication Patterns

- **Pod-to-Pod**: Direct IP communication
- **Service-based**: Stable endpoints, load balancing
- **DNS-based**: Service discovery by name
- **Ingress/Egress**: External access control

### Network Policies

- **Default allow**: All traffic allowed initially
- **Default deny**: After first policy, traffic blocked
- **Zero trust**: Explicitly allow only required traffic
- **Label-based**: Use labels, not IPs

### Services

- **ClusterIP**: Internal service with stable IP
- **LoadBalancer**: External load balancer (cloud)
- **NodePort**: External access via node port
- **Headless**: Direct pod access (StatefulSets)

### DNS

- **CoreDNS**: Kubernetes DNS server
- **Service DNS**: `<service>.<namespace>.svc.cluster.local`
- **Pod DNS**: `<pod-ip>.<namespace>.pod.cluster.local`
- **Custom DNS**: Can be overridden per pod

### Service Mesh

- **Optional**: Not required for basic Kubernetes
- **Sidecar vs eBPF**: Traditional vs modern approaches
- **Benefits**: Observability, security, traffic management
- **Trade-offs**: Complexity vs features

---

## 10. Common Interview / Exam Questions

**Q1: What are the four Kubernetes networking rules?**
A: 1) All pods can communicate without NAT, 2) All nodes can communicate with all pods without NAT, 3) Pods see the same IP as outside, 4) No port mapping required.

**Q2: How do pods on the same node communicate?**
A: Through the node's bridge (cni0) and veth pairs, without leaving the physical network.

**Q3: How do pods on different nodes communicate?**
A: Via physical network, using routing (Calico) or encapsulation (Flannel VXLAN), depending on CNI.

**Q4: What is the default network policy behavior?**
A: Default allow (no policies = all traffic allowed), but after first policy in namespace, becomes default deny.

**Q5: What are the main Kubernetes service types?**
A: ClusterIP (default, internal), NodePort (node access), LoadBalancer (external LB), ExternalName (DNS alias).

**Q6: How does DNS work in Kubernetes?**
A: CoreDNS provides DNS resolution, service DNS format is `<service>.<namespace>.svc.cluster.local`.

**Q7: What is a service mesh?**
A: Infrastructure layer for managing service-to-service communication with observability, security, and traffic management.

**Q8: What's the difference between Istio and Cilium Service Mesh?**
A: Istio uses Envoy sidecars, Cilium uses eBPF in kernel (no sidecars), making Cilium more performant.

**Q9: Why use labels instead of IPs in network policies?**
A: Labels are stable across pod restarts, IPs change, making policies more maintainable.

**Q10: What is a headless service?**
A: Service with clusterIP: None, DNS returns pod IPs directly, used with StatefulSets.

---

## 11. Next Topics to Learn

- **Ingress Controllers**: External access to services
- **Network Policies Deep Dive**: Advanced policy scenarios
- **Service Mesh Implementation**: Hands-on with Istio or Cilium
- **Multi-Cluster Networking**: Connecting multiple clusters
- **Network Performance**: Tuning and optimization

---

## 12. Practical Commands

### Pod Communication

```bash
# Get pod IPs
kubectl get pods -o wide

# Test pod-to-pod
kubectl exec pod-a -- ping pod-b-pod-ip

# Test with curl
kubectl exec pod-a -- curl http://pod-b-pod-ip:80

# Trace route
kubectl exec pod-a -- traceroute pod-b-pod-ip
```

### Services

```bash
# List services
kubectl get services

# Describe service
kubectl describe service <service-name>

# Get endpoints
kubectl get endpoints <service-name>

# Test service DNS
kubectl run test --rm -it --image=busybox -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test service connectivity
kubectl run test --rm -it --image=curlimages/curl -- curl http://<service-name>:<port>
```

### Network Policies

```bash
# List policies
kubectl get networkpolicy -A

# Describe policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# View policy YAML
kubectl get networkpolicy <policy-name> -o yaml

# Test policy
kubectl run test --rm -it --image=busybox -- wget -O- http://<target-pod-ip>:<port>
```

### DNS

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS
kubectl exec <pod-name> -- nslookup kubernetes.default

# Test service DNS
kubectl exec <pod-name> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

### Service Mesh (Cilium)

```bash
# Check service mesh status
cilium status | grep Mesh

# Observe flows
hubble observe

# View service graph
cilium service graph

# Check metrics
cilium metrics list
```

---

**End of Internal Kubernetes Communication**