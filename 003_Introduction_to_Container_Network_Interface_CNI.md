# Introduction to Container Network Interface (CNI)

This document provides a **comprehensive introduction to the Container Network Interface (CNI)** standard, explaining what it is, how it works, and how Kubernetes uses it to implement networking.

---

## 1. What is CNI?

### Definition

**CNI (Container Network Interface)** is a **specification** and a set of **libraries** that define how container runtimes should configure network interfaces for containers.

### Key Points

- Created by **CNCF (Cloud Native Computing Foundation)**
- Originally developed for **rkt** container runtime
- Now used by **Kubernetes** and other container orchestration systems
- **Plugin-based architecture** - different implementations exist
- Language-agnostic specification
- Focuses **only on network connectivity**

### What CNI Is NOT

- ❌ Not a network implementation itself
- ❌ Not a runtime (like Docker or containerd)
- ❌ Not specific to Kubernetes (though commonly used)
- ❌ Not concerned with container lifecycle management

### What CNI Is

- ✅ A **specification** for network plugin interface
- ✅ A **plugin architecture** for container networking
- ✅ Responsible for **network setup and teardown**
- ✅ The standard used by **Kubernetes** for pod networking

---

## 2. How CNI Works

### High-Level Concept

CNI provides a **contract** between:
- **Container Runtime** (e.g., kubelet, containerd)
- **Network Plugin** (e.g., Calico, Flannel, Cilium)

### The CNI Model

```
Container Runtime
       ↓
   Calls CNI Plugin
       ↓
  CNI Plugin Executable
       ↓
  Network Configuration
```

### How It Works - Step by Step

#### Step 1: Container Runtime Request

When a container/pod needs networking:

```
kubelet → CNI Plugin (with ADD command)
```

#### Step 2: Plugin Execution

The CNI plugin:
- Receives **environment variables**
- Reads **configuration file** (JSON format)
- Performs network setup actions
- Returns **result** to runtime

#### Step 3: Network Setup Actions

Typical actions include:
- Create **network namespace**
- Create **veth pairs**
- Assign **IP address**
- Set up **routing**
- Configure **iptables** or **eBPF**

#### Step 4: Container Teardown

When container/pod is deleted:

```
kubelet → CNI Plugin (with DEL command)
```

The plugin:
- Removes IP address
- Deletes interfaces
- Cleans up routes
- Removes firewall rules

---

## 3. How CNI Works on Kubernetes

### Kubernetes Networking Requirements

Kubernetes networking model requires:

1. **Flat network** - All pods can communicate without NAT
2. **IP-per-Pod** - Each pod gets a unique IP
3. **Cross-node communication** - Pods on different nodes can talk
4. **Service abstraction** - Stable IP for pod groups

### Kubernetes + CNI Integration

```
┌─────────────────────────────────────┐
│         Kubernetes Cluster          │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ kubelet      │  │ kubelet      │ │
│  │              │  │              │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │ │
│  │ │ CNI      │ │  │ │ CNI      │ │ │
│  │ │ Plugin   │ │  │ │ Plugin   │ │ │
│  │ └──────────┘ │  │ └──────────┘ │ │
│  └──────────────┘  └──────────────┘ │
│         ↓                ↓          │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ Pod 1        │  │ Pod 2        │ │
│  │ 10.244.0.2   │  │ 10.244.0.3   │ │
│  └──────────────┘  └──────────────┘ │
│                                     │
└─────────────────────────────────────┘
```

### The kubelet Role

**kubelet** is the component that:
- Manages pods on a node
- Invokes CNI plugins when:
  - Pod is created (`ADD` command)
  - Pod is deleted (`DEL` command)
- Passes configuration to the plugin
- Receives results (IP, MAC, routes)

### CNI Plugin Location

By default, kubelet looks for CNI plugins in:
```
/opt/cni/bin/
```

### Network Configuration Location

kubelet reads CNI configuration from:
```
/etc/cni/net.d/
```

### Example Workflow

1. **User creates a pod** via kubectl
2. **kubelet** schedules pod to a node
3. **kubelet** creates network namespace for pod
4. **kubelet** calls CNI plugin with `ADD` command
5. **CNI plugin**:
   - Reads config from `/etc/cni/net.d/`
   - Creates network interfaces
   - Assigns IP
   - Sets up routing
6. **CNI plugin** returns IP address to kubelet
7. **kubelet** stores pod IP in API server
8. **Pod is now networked**

When pod is deleted:
1. **kubelet** calls CNI plugin with `DEL` command
2. **CNI plugin** cleans up network resources

---

## 4. CNI Specifications

### Official Specification

The CNI specification is defined at:
https://github.com/containernetworking/cni

### Core Specification Components

1. **Plugin Executable**
   - Must be executable binary/script
   - Must accept commands via arguments
   - Must receive configuration via stdin or file

2. **Required Commands**

Every CNI plugin MUST implement:
- **ADD** - Set up network for container
- **DEL** - Remove network configuration
- **CHECK** - Verify network configuration (optional)
- **VERSION** - Report plugin version (optional)

3. **Environment Variables**

Required environment variables:
- **CNI_COMMAND** - Command to execute (ADD/DEL/CHECK/VERSION)
- **CNI_CONTAINERID** - Container identifier
- **CNI_NETNS** - Network namespace path
- **CNI_IFNAME** - Interface name inside container
- **CNI_ARGS** - Additional arguments
- **CNI_PATH** - Path to plugin executables

4. **Standard Input**

Configuration passed via stdin (JSON format)

5. **Output Format**

Plugin MUST output JSON result

### Version Compatibility

CNI uses semantic versioning:
- Major version changes indicate breaking changes
- Different implementations support different versions
- Kubernetes supports CNI specification v0.4.0 and v1.0.0+

---

## 5. Configuration File Format

### Location and Naming

- Directory: `/etc/cni/net.d/`
- Format: JSON
- Naming convention: `<name>.conf` or `<name>-<type>.conflist`

### Single Plugin Configuration (.conf)

Example simple configuration:

```json
{
  "cniVersion": "0.4.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ]
  }
}
```

### Multi-Plugin Configuration (.conflist)

For multiple plugins chained together:

```json
{
  "cniVersion": "0.4.0",
  "name": "kubernetes-network",
  "plugins": [
    {
      "type": "calico",
      "ipam": {
        "type": "calico-ipam"
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

### Configuration Fields Explained

| Field | Description | Required |
|-------|-------------|----------|
| `cniVersion` | CNI spec version | Yes |
| `name` | Network name | Yes |
| `type` | Plugin type | Yes |
| `ipam` | IPAM plugin config | Yes |
| `dns` | DNS configuration | No |
| `plugins` | List of plugins (for .conflist) | No |

### IPAM Configuration

IPAM (IP Address Management) plugins handle IP assignment:

```json
"ipam": {
  "type": "host-local",    // or "calico-ipam", "dhcp", etc.
  "subnet": "10.244.0.0/16",
  "gateway": "10.244.0.1",
  "routes": [
    {
      "dst": "0.0.0.0/0"
    }
  ]
}
```

---

## 6. Protocol for Interaction

### Plugin Invocation

The container runtime invokes plugins as follows:

```bash
CNI_COMMAND=ADD \
CNI_CONTAINERID=<container-id> \
CNI_NETNS=/run/netns/<ns-name> \
CNI_IFNAME=eth0 \
CNI_ARGS=IgnoreUnknown=1 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < config.json
```

### Input to Plugin

**Via Environment Variables:**
- `CNI_COMMAND` - The command to execute
- `CNI_CONTAINERID` - Unique container ID
- `CNI_NETNS` - Path to network namespace
- `CNI_IFNAME` - Interface name (default: eth0)
- `CNI_ARGS` - Additional args (key=value pairs)
- `CNI_PATH` - Path to search for plugins

**Via Standard Input:**
- JSON configuration
- Network parameters
- Plugin-specific options

### Output from Plugin

Plugin writes JSON to stdout:

**ADD Command Response:**

```json
{
  "cniVersion": "0.4.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "00:11:22:33:44:55",
      "sandbox": "/run/netns/ns-123"
    }
  ],
  "ips": [
    {
      "version": "4",
      "address": "10.244.0.5/32",
      "gateway": "10.244.0.1"
    }
  ],
  "routes": [
    {
      "dst": "0.0.0.0/0",
      "gw": "10.244.0.1"
    }
  ],
  "dns": {}
}
```

**DEL Command Response:**
- Usually empty response
- Success indicated by exit code 0

### Error Handling

Plugin should:
- Write error to stderr
- Exit with non-zero code on failure
- Include error details in JSON (if possible)

---

## 7. Execution of Network Configurations

### Execution Flow

```
1. Runtime prepares execution environment
   ↓
2. Runtime sets environment variables
   ↓
3. Runtime creates config (or reads from file)
   ↓
4. Runtime invokes plugin executable
   ↓
5. Plugin validates configuration
   ↓
6. Plugin performs network operations
   ↓
7. Plugin returns result (or error)
   ↓
8. Runtime processes result
```

### ADD Operation

When adding network to a container:

1. **Create/Identify Namespace**
   - If `CNI_NETNS` points to existing ns: use it
   - If not: create new namespace

2. **Create Interface**
   - Create veth pair
   - Move one end into container namespace
   - Set interface name to `CNI_IFNAME`

3. **Configure IP**
   - Assign IP address to interface
   - Set up routes
   - Configure gateway

4. **Additional Setup**
   - Set up NAT (if required)
   - Configure firewall rules
   - Set up DNS

5. **Return Result**
   - IP address
   - MAC address
   - Routes
   - DNS configuration

### DEL Operation

When removing network from a container:

1. **Identify Resources**
   - Find interfaces
   - Identify IP addresses
   - Identify routes

2. **Cleanup**
   - Remove IP addresses
   - Delete interfaces
   - Remove routes
   - Clean up firewall rules

3. **Remove Namespace** (if created by plugin)
   - Delete network namespace

### CHECK Operation (Optional)

Verify that network configuration is still valid:

1. Verify interface exists
2. Verify IP is assigned
3. Verify routes are active
4. Verify connectivity (optional)

---

## 8. Execution Order of Operations

### Multi-Plugin (Chained) Execution

When using multiple plugins (via `.conflist`), plugins are executed in order:

```
Plugin 1 (e.g., calico)
   ↓
Plugin 2 (e.g., portmap)
   ↓
Plugin 3 (e.g., tuning)
   ↓
Final Result
```

### ADD Operation Order

For multi-plugin setup:

1. **First Plugin:**
   - Sets up primary network
   - Creates interfaces
   - Assigns IP

2. **Subsequent Plugins:**
   - Receive previous plugin's output
   - Modify or extend configuration
   - Add features (NAT, port mapping, etc.)

3. **Final Output:**
   - Result of last plugin
   - Aggregate of all plugin actions

### DEL Operation Order

For multi-plugin setup:

1. **Execute in REVERSE order:**
   - Last plugin executes first
   - First plugin executes last

2. **Example:**
   ```
   Plugin 3 (tuning) - DELETE
   ↓
   Plugin 2 (portmap) - DELETE
   ↓
   Plugin 1 (calico) - DELETE
   ```

This ensures dependencies are cleaned up properly.

### Why Reverse Order?

- **Dependency cleanup**: Later plugins may depend on earlier ones
- **Resource ownership**: Later plugins might own resources created earlier
- **Safe teardown**: Ensures cleanup doesn't break remaining configuration

---

## 9. Execution Delegation

### What is Execution Delegation?

Execution delegation allows a **CNI plugin** to delegate work to **other plugins**.

### Use Cases

1. **Meta-plugins** - Plugins that coordinate other plugins
2. **Multi-network support** - Attach containers to multiple networks
3. **Feature composition** - Combine different network features

### How It Works

A plugin can:
- Call other CNI plugins
- Pass modified configuration
- Aggregate results from multiple plugins

### Example: Multi-CNI Plugin

```json
{
  "cniVersion": "0.4.0",
  "name": "my-multinet",
  "type": "multus",
  "delegates": [
    {
      "type": "calico",
      "name": "calico-network"
    },
    {
      "type": "bridge",
      "name": "bridge-network"
    }
  ]
}
```

### Delegation Flow

```
kubelet
   ↓
Multus (Meta-Plugin)
   ↓   ↓
Calico  Bridge
   ↓   ↓
Result aggregation
```

### Benefits

- **Flexibility**: Combine multiple network types
- **Separation of concerns**: Different plugins for different features
- **Extensibility**: Easy to add new capabilities

---

## 10. Result Types

### ADD Command Result

When a plugin successfully adds network configuration:

```json
{
  "cniVersion": "0.4.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "00:11:22:33:44:55",
      "sandbox": "/run/netns/ns-123"
    },
    {
      "name": "veth123",
      "mac": "00:11:22:33:44:56"
    }
  ],
  "ips": [
    {
      "version": "4",
      "address": "10.244.0.5/32",
      "gateway": "10.244.0.1"
    },
    {
      "version": "6",
      "address": "fd00:10:244::5/128",
      "gateway": "fd00:10:244::1"
    }
  ],
  "routes": [
    {
      "dst": "0.0.0.0/0",
      "gw": "10.244.0.1"
    },
    {
      "dst": "192.168.1.0/24"
    }
  ],
  "dns": {
    "nameservers": [
      "10.96.0.10"
    ],
    "domain": "cluster.local",
    "search": [
      "default.svc.cluster.local"
    ]
  }
}
```

### Result Fields

#### interfaces

List of network interfaces created/modified:

- **name**: Interface name
- **mac**: MAC address
- **sandbox**: Network namespace path (for container interfaces)
- Optional fields: `sandbox` (host namespace interfaces don't have this)

#### ips

List of IP addresses assigned:

- **version**: IP version (4 or 6)
- **address**: IP address with subnet mask
- **gateway**: Default gateway
- Optional fields: `interface` (index in interfaces array)

#### routes

List of routing rules:

- **dst**: Destination network
- **gw**: Gateway address (optional)

#### dns

DNS configuration:

- **nameservers**: List of DNS servers
- **domain**: Search domain
- **search**: Search domains
- **options**: DNS options

### DEL Command Result

- **Success**: No output required, exit code 0
- **Failure**: Error message to stderr, non-zero exit code

### CHECK Command Result

Similar to ADD, but verification-focused:

```json
{
  "cniVersion": "0.4.0"
}
```

### Error Result

On failure, plugin should output:

```json
{
  "cniVersion": "0.4.0",
  "code": 100,
  "msg": "Failed to create interface",
  "details": "veth creation failed: no such device"
}
```

---

## 11. Key Features

### Core Features of CNI

#### 1. Plugin-Based Architecture

- **Extensible**: Anyone can create a CNI plugin
- **Swappable**: Change networking without modifying runtime
- **Composable**: Chain multiple plugins together

#### 2. Declarative Configuration

- **JSON-based**: Easy to read and modify
- **Versioned**: Clear compatibility expectations
- **Standardized**: Same format across all plugins

#### 3. Runtime Agnostic

- **Not Kubernetes-specific**: Works with any runtime
- **Flexible**: Can be used by Docker, containerd, CRI-O, etc.
- **Portable**: Same plugin works across environments

#### 4. Stateless

- **No daemon required**: Plugins are executables
- **Simpler deployment**: No need for plugin process management
- **Easier debugging**: Direct plugin execution

#### 5. IPAM Plugin Support

- **Separate IP management**: IPAM plugins handle address allocation
- **Multiple strategies**: Static, DHCP, host-local, etc.
- **Flexible**: Choose appropriate IPAM for your needs

#### 6. Network Namespace Support

- **Isolation**: Each container gets its own network namespace
- **Security**: Prevents interference between containers
- **Flexibility**: Multiple network namespaces per host

#### 7. Multiple Networks

- **Multi-homing**: Containers can have multiple network interfaces
- **Different networks**: Attach to different network types
- **Use cases**: Service networks, management networks, etc.

#### 8. Version Management

- **Semantic versioning**: Clear compatibility rules
- **Version negotiation**: Runtime and plugin agree on version
- **Backward compatibility**: Older plugins can work with newer runtimes

### Advanced Features

#### Capability Support

Plugins can declare capabilities:

```json
{
  "type": "portmap",
  "capabilities": {
    "portMappings": true
  }
}
```

Runtime can then request capabilities:
```
CNI_ARGS=portMappings=[{"hostPort":8080,"containerPort":80}]
```

#### Chaining

Multiple plugins can work together:
- Plugin 1: Sets up basic network
- Plugin 2: Adds port mapping
- Plugin 3: Configures firewall rules

#### Delegation

Plugins can delegate to other plugins:
- Meta-plugins coordinate multiple networks
- Complex networking scenarios
- Multi-cloud or hybrid environments

---

## 12. Popular CNIs

### Overview

Many CNI implementations exist, each with different approaches and trade-offs.

### Cilium

**Type:** eBPF-based
**Approach:** Layer 3/4/7 networking + security

**Key Features:**
- Uses **eBPF** in Linux kernel for high performance
- Layer 7 awareness (HTTP, gRPC, Kafka, etc.)
- Advanced network policies
- Built-in observability
- Native Kubernetes integration
- Service mesh support

**Best For:**
- High-performance requirements
- Advanced security needs
- Microservices architectures
- L7 policy enforcement

**Architecture:**
```
┌─────────────────────────────────────┐
│          Cilium (eBPF)              │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌──────┐ │
│  │ BPF Maps│  │ Datapath│  │Agent │ │
│  └─────────┘  └─────────┘  └──────┘ │
└─────────────────────────────────────┘
```

### Calico

**Type:** BGP + iptables/IPVS
**Approach:** Layer 3 networking

**Key Features:**
- Pure Layer 3 networking (no overlay)
- BGP-based routing
- Network policies
- Supports VXLAN, IPIP, and BGP modes
- Integrates with cloud providers
- Network policy enforcement

**Best For:**
- Data center deployments
- BGP familiarity
- Layer 3 networking preference
- Network policy requirements

**Architecture:**
```
┌─────────────────────────────────────┐
│          Calico                     │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌──────┐ │
│  │ BGP     │  │ Felix   │  │Typha │ │
│  │ Speaker │  │ (iptables)│      │ │
│  └─────────┘  └─────────┘  └──────┘ │
└─────────────────────────────────────┘
```

### Flannel

**Type:** Simple overlay
**Approach:** Layer 2 overlay network

**Key Features:**
- Simple to deploy and configure
- Multiple backend options (VXLAN, host-gw, AWS, etc.)
- Small resource footprint
- Good for small to medium clusters
- Uses etcd for coordination

**Best For:**
- Simple deployments
- Learning Kubernetes networking
- Small clusters
- Quick setup

**Architecture:**
```
┌─────────────────────────────────────┐
│          Flannel                    │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌──────┐ │
│  │flanneld │  │ VXLAN   │  │etcd  │ │
│  └─────────┘  └─────────┘  └──────┘ │
└─────────────────────────────────────┘
```

### Weave Net

**Type:** Overlay network
**Approach:** Mesh networking

**Key Features:**
- Automatic peer discovery
- No external database required
- Fast data path
- Network policies
- Encryption support
- Works across cloud providers

**Best For:**
- Multi-cloud deployments
- Simplicity (no external dependencies)
- Automatic mesh networking

### Antrea

**Type:** Open vSwitch + eBPF (optional)
**Approach:** Layer 3/4 networking

**Key Features:**
- Uses Open vSwitch (OVS)
- Deep integration with Kubernetes
- Network policies
- Support for eBPF acceleration
- Cloud-native design

**Best For:**
- Enterprises familiar with OVS
- Cloud-native deployments
- Network policy requirements

### Multus

**Type:** Meta-plugin
**Approach:** Multi-network support

**Key Features:**
- Enables multiple network interfaces per pod
- Delegates to other CNI plugins
- Network-attachment-definition K8s CRD
- Supports all other CNI plugins
- Use cases: SR-IOV, DPDK, macvlan

**Best For:**
- Multi-network requirements
- Specialized networking needs
- Legacy application integration

### Comparison Table

| CNI | Performance | Complexity | Features | Best For |
|-----|-------------|------------|----------|----------|
| Cilium | Very High | High | L3-L7 + Security | Enterprise, Cloud-native |
| Calico | High | Medium | L3 + Policies | Data centers, BGP |
| Flannel | Medium | Low | Basic overlay | Small clusters, Learning |
| Weave Net | Medium | Low | Mesh + Encryption | Multi-cloud, Simple |
| Antrea | High | Medium | L3-L4 + Policies | OVS users, Enterprise |
| Multus | Varies | High | Multi-network | Specialized needs |

### Choosing the Right CNI

**For Small Clusters/Learning:**
- Flannel (simple, easy to understand)

**For Enterprise Production:**
- Cilium (performance, security, observability)
- Calico (mature, BGP-based)

**For Multi-Cloud:**
- Weave Net
- Cilium

**For Specialized Needs:**
- Multus (multi-network)
- Cilium (L7 policies)

### Emerging CNIs

- **Cilium**: Growing rapidly, becoming the default choice
- **Cilium Service Mesh**: Integrated service mesh capabilities
- **AWS VPC CNI**: Native AWS integration
- **Azure CNI**: Native Azure integration
- **Google Cloud CNI**: Native GCP integration

---

## 13. Key Takeaways (Architect Notes)

### CNI Fundamentals

- CNI is a **specification**, not an implementation
- Kubernetes delegates pod networking to **CNI plugins**
- Plugins are **executables** invoked by kubelet
- Configuration is **JSON-based** and declarative

### How CNI Integrates with Kubernetes

- **kubelet** invokes plugins at pod create/delete
- Plugins read config from `/etc/cni/net.d/`
- Plugins return IP address to kubelet
- kubelet stores pod IP in Kubernetes API server

### Plugin Types

- **Primary plugins**: Calico, Flannel, Cilium, etc.
- **IPAM plugins**: host-local, dhcp, calico-ipam
- **Meta-plugins**: Multus, multi-cni

### Execution Model

- **ADD**: Create network configuration
- **DEL**: Remove network configuration
- **CHECK**: Verify configuration (optional)
- **VERSION**: Report version (optional)

### Why CNI Matters

- Enables **flexible networking implementations**
- Decouples **runtime** from **networking**
- Allows **pluggable network solutions**
- Standardizes **container networking interface**

---

## 14. Common Interview / Exam Questions

**Q1: What is CNI?**
A: Container Network Interface - a specification for configuring network interfaces for containers.

**Q2: How does Kubernetes use CNI?**
A: Kubernetes kubelet invokes CNI plugins to set up and tear down pod networking.

**Q3: What commands does a CNI plugin must implement?**
A: ADD and DEL. CHECK and VERSION are optional.

**Q4: Where are CNI configurations stored?**
A: `/etc/cni/net.d/`

**Q5: Where are CNI plugins installed?**
A: `/opt/cni/bin/`

**Q6: What is the execution order for multiple plugins?**
A: ADD: execute in order. DEL: execute in reverse order.

**Q7: What is an IPAM plugin?**
A: IP Address Management plugin - handles IP address allocation.

**Q8: Name three popular CNI implementations.**
A: Calico, Flannel, Cilium (or Weave Net, Antrea, etc.)

**Q9: What is the difference between a CNI specification and implementation?**
A: Specification defines the interface; implementation (plugin) provides actual networking.

**Q10: How does kubelet know which CNI plugin to use?**
A: Reads configuration files from `/etc/cni/net.d/`

---

## 15. Next Topics to Learn

- **Cilium Deep Dive**: eBPF-based networking
- **Network Policies**: Kubernetes network security
- **Services & kube-proxy**: Service discovery and load balancing
- **Ingress Controllers**: External access to services
- **Service Mesh**: Istio, Linkerd, Cilium Service Mesh
- **Network Performance**: Tuning and optimization
- **Multi-Network**: Using Multus for advanced scenarios

---

## 16. Practical Commands

### Check CNI Configuration

```bash
# View CNI configuration
cat /etc/cni/net.d/*.conf

# List installed CNI plugins
ls -la /opt/cni/bin/
```

### Check CNI Plugin Logs

```bash
# If using systemd journal
journalctl -u kubelet | grep CNI

# Check kubelet logs
kubectl logs -n kube-system kubelet-<node-name>
```

### Inspect CNI Configuration in Pod

```bash
# Check which CNI is being used
kubectl exec -it <pod-name> -- ip addr

# View routing table
kubectl exec -it <pod-name> -- ip route

# View network namespace on node
ssh <node>
ip netns list
```

### Test CNI Plugin Manually

```bash
# Manually invoke a CNI plugin
CNI_COMMAND=ADD \
CNI_CONTAINERID=test123 \
CNI_NETNS=/run/netns/testns \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < config.json
```

---

**End of Introduction to Container Network Interface (CNI)**