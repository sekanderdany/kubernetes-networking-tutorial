# Network Policy Overview

This document provides a **comprehensive overview of Kubernetes Network Policies**, explaining why they are needed, how they work, how to define them, their benefits and limitations, and includes a hands-on demo for applying default deny and allow policies for pod isolation.

---

## 1. Why Network Policies Are Needed

### The Default Behavior Problem

Kubernetes makes it very simple to communicate with Pods across the cluster. By default:

- **All pods can communicate with all other pods** — no restrictions
- **All egress traffic is allowed** — pods can reach any external endpoint
- **All ingress traffic is allowed** — any pod can be reached by any other pod

While this is great for development and testing, it is **not suitable for production-grade clusters** where you need to:

- Include **security-hardening practices**
- **Control the flow** of the network
- **Ensure compliance** with organizational and regulatory standards

```
Default Kubernetes Network (No Policies):
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                             │
│                                                                  │
│  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐          │
│  │ Pod A  │←──→│ Pod B  │←──→│ Pod C  │←──→│ Pod D  │          │
│  │(frontend)│  │(api)   │    │(db)    │    │(util)  │          │
│  └────────┘    └────────┘    └────────┘    └────────┘          │
│       ↑             ↑             ↑             ↑               │
│       └─────────────┴─────────────┴─────────────┘               │
│               ALL TRAFFIC ALLOWED (Wide Open)                   │
│               ❌ No isolation                                    │
│               ❌ No security boundaries                          │
│               ❌ No compliance enforcement                       │
└─────────────────────────────────────────────────────────────────┘
```

In most cases, **we cannot have a wide-open network** in production.

---

## 2. What Are Network Policies?

### Definition

**Network Policies** are a native Kubernetes resource that defines rules for controlling traffic flow between pods within a cluster. These policies enable **fine-grained control** over network communication, allowing administrators to enforce **security**, **segmentation**, and **isolation** of applications within the cluster.

### The Primary Purpose

The primary purpose of Kubernetes Network Policies is to **enhance the security and isolation** of applications within the Kubernetes cluster.

### The Road Analogy

Think of it like setting up **boundaries and traffic signs on a road**:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Road Network Analogy                         │
│                                                                  │
│   ┌─────────┐          ┌─────────┐          ┌─────────┐        │
│   │  🏢 HQ  │  ← ✅ →  │Warehouse│  ← ✅ →  │  Store  │        │
│   └─────────┘          └─────────┘          └─────────┘        │
│       ↑                                                      │
│       │ ✅ Allowed                                           │
│       ↓                                                      │
│   ┌─────────┐                                               │
│   │Office B │                                               │
│   └─────────┘                                               │
│                                                                  │
│   ┌─────────┐          ┌─────────┐                              │
│   │Public   │  ← ❌ →  │Server   │  (Blocked by policy)       │
│   │Network  │          │Room     │                              │
│   └─────────┘          └─────────┘                              │
│                                                                  │
│   Traffic signs (Network Policies) guide and control             │
│   the flow of traffic between destinations (Pods)                │
└─────────────────────────────────────────────────────────────────┘
```

Just as traffic signs guide cars and trucks on the road, **network policies in Kubernetes guide and control the flow of traffic between pods**.

### How Network Policies Direct Traffic

Network policies direct the flow in the Kubernetes cluster. Their main job is to make sure that **only the right resources can talk to each other**.

### CNI Plugin Dependency

> ⚠️ **Important**: Network policies are **implemented by the CNI plugins**. Attempting to use a network policy without a compatible CNI plugin **would not work**.

```
┌──────────────────────────────────────────────────────────────┐
│                  Network Policy Implementation                │
│                                                               │
│  ┌──────────────────┐         ┌──────────────────────┐       │
│  │ Kubernetes       │         │ CNI Plugin            │       │
│  │ NetworkPolicy    │ ──────→ │ (Calico, Cilium, etc.)│       │
│  │ (Declarative)    │         │ (Enforcement)         │       │
│  └──────────────────┘         └───────────┬──────────┘       │
│                                           │                   │
│                                           ↓                   │
│                               ┌──────────────────────┐       │
│                               │ Data Plane           │       │
│                               │ (iptables / eBPF)    │       │
│                               │ (Actual filtering)   │       │
│                               └──────────────────────┘       │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. How Network Policies Work

### Traffic Control Mechanism

Network policies control the flow to resources through the use of **ingress** and **egress** rules.

### The Three Identifiers

The entities that a pod can communicate with are identified through a combination of the following **three identifiers**:

| Identifier | Selector Used | Description |
|------------|---------------|-------------|
| **Other Pods** | `podSelector` | Target specific pods by labels |
| **Namespaces** | `namespaceSelector` | Target all pods in specific namespaces |
| **IP Blocks** | `ipBlock` | Target specific CIDR ranges |

### Selection Methods

```
┌──────────────────────────────────────────────────────────────┐
│                Network Policy Selectors                       │
│                                                               │
│  1. Pod & Namespace Based:                                    │
│     ┌──────────────────┐    ┌──────────────────────┐         │
│     │ namespaceSelector│    │ podSelector           │         │
│     │ (matchLabels /   │    │ (matchLabels /        │         │
│     │  matchExpressions)│    │  matchExpressions)    │         │
│     └──────────────────┘    └──────────────────────┘         │
│                                                               │
│  2. IP Block Based:                                          │
│     ┌──────────────────┐    ┌──────────────────────┐         │
│     │ cidr             │    │ except                │         │
│     │ (10.0.0.0/8)    │    │ (10.0.1.0/24)        │         │
│     └──────────────────┘    └──────────────────────┘         │
│                                                               │
│  3. Port Based (Layer 4):                                    │
     ┌──────────────────┐    ┌──────────────────────┐         │
│     │ port            │    │ endPort               │         │
│     │ (80)            │    │ (443 - port range)    │         │
│     └──────────────────┘    └──────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

### The Two Isolation Types

There are two ways in which a pod can be isolated:

| Type | Direction | Description |
|------|-----------|-------------|
| **Ingress** | Incoming | Defines what **incoming** network flows are allowed **from** |
| **Egress** | Outgoing | Defines what **outgoing** network flows are allowed **to** |

```
┌──────────────────────────────────────────────────────────────┐
│                     Pod Traffic Flow                          │
│                                                               │
│         INGRESS                          EGRESS              │
│      (Incoming Traffic)              (Outgoing Traffic)       │
│                                                               │
│           ┌─────────┐                              │         │
│  FROM ──→ │         │ ──→ TO                      │         │
│           │   POD   │                              ↓         │
│  FROM ←── │         │ ←── TO                     │         │
│           └─────────┘                              ↓         │
│                                                               │
│  Ingress Rules:                  Egress Rules:               │
│  - Who can talk TO this pod     - Who this pod can talk TO   │
│  - Control incoming traffic     - Control outgoing traffic   │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Defining a NetworkPolicy

### Anatomy of a NetworkPolicy

Network policies are defined by the `NetworkPolicy` kind and are defined at the **namespace level**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
```

### NetworkPolicy Structure Breakdown

```
┌──────────────────────────────────────────────────────────────┐
│              NetworkPolicy Structure                          │
│                                                               │
│  apiVersion: networking.k8s.io/v1                            │
│  kind: NetworkPolicy                                          │
│  metadata:                                                    │
│    name: <policy-name>                                        │
│    namespace: <namespace>        ← Namespace-level resource  │
│  spec:                                                        │
│    podSelector:                    ← Which pods to apply to   │
│      matchLabels:                                             │
│        key: value                                             │
│                                                               │
│      ⚠️ No matchLabels = ALL pods in namespace               │
│                                                               │
│    policyTypes:                    ← One or both:             │
│    - Ingress                       ←   Ingress, Egress        │
│    - Egress                                                   │
│                                                               │
│    ingress:                        ← Ingress rules            │
│    - from:                         ←   from block             │
│      - podSelector:                                           │
│          matchLabels:                                         │
│      - namespaceSelector:                                     │
│          matchLabels:                                         │
│      - ipBlock:                                               │
│          cidr:                                                │
│      ports:                        ←   ports block            │
│                                                               │
│    egress:                         ← Egress rules             │
│    - to:                           ←   to block               │
│      - podSelector:                                           │
│      - namespaceSelector:                                     │
│      - ipBlock:                                               │
│      ports:                                                   │
└──────────────────────────────────────────────────────────────┘
```

### Key Fields Explained

#### 1. podSelector

- Specifies **which pods** to apply the policy to
- Uses `matchLabels` or `matchExpressions`
- **Excluding matchLabels** applies the policy to **all pods** in the namespace

```yaml
# Target specific pods
podSelector:
  matchLabels:
    app: nginx

# Target ALL pods in namespace (empty selector)
podSelector: {}
```

#### 2. policyTypes

- Defines **one or both** of the isolation types: `Ingress` and/or `Egress`
- If both are specified, both ingress and egress rules apply

```yaml
policyTypes:
  - Ingress
  - Egress
```

#### 3. Ingress Rules (from block)

- Defines a list of entities allowed to **send traffic to** the selected pods
- Uses `podSelector`, `namespaceSelector`, and/or `ipBlock`

```yaml
ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    - namespaceSelector:
        matchLabels:
          env: production
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
          - 10.0.1.0/24
```

#### 4. Egress Rules (to block)

- Defines a list of entities that the selected pods are allowed to **send traffic to**
- Same selectors as ingress

```yaml
egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
```

### Namespace and Pod Selectors

For namespace and pod entities, selection is done through using the `namespaceSelector` and `podSelector` fields:

```yaml
# Match specific labels
- podSelector:
    matchLabels:
      app: nginx

# Match using expressions (can target multiple namespaces)
- namespaceSelector:
    matchExpressions:
    - key: environment
      operator: In
      values: ["production", "staging"]
```

> 💡 **Note**: You can use `matchExpressions` to target **multiple namespaces** with a single rule.

### IP Block Selection

For IP-based network policies, policies are defined based on IP blocks:

```yaml
- ipBlock:
    cidr: 10.0.0.0/8          # Allow this CIDR range
    except:                     # But exclude these
      - 10.0.1.0/24
      - 10.0.2.0/24
```

### Port Configuration

Network Policies operate at **Layer 4**. Because of this, it is possible to configure by **protocol** and even a **range of ports** within your `to` and `from` blocks:

```yaml
ports:
  - port: 80               # Target port
    protocol: TCP           # Protocol (defaults to TCP if unspecified)
    endPort: 443            # Port range: 80 to 443 (Kubernetes 1.25+)
```

| Field | Description | Required |
|-------|-------------|----------|
| `port` | The given port on a Pod for ingress or egress | Yes |
| `protocol` | Protocol of the port (TCP, UDP, SCTP). Defaults to **TCP** | No |
| `endPort` | Used to allow a port range starting with `port` and ending with this value (Kubernetes 1.25+) | No |

> 💡 **Note**: Defining ports is another way to further enhance control over the network.

---

## 5. Default Network Policies

### Default Behavior

If **no policy exists**, then **all ingress and egress is allowed** to and from all Pods:

```
No Network Policy:
┌──────────────────────────────────────────────┐
│                                               │
│   Pod A ←──── All Traffic Allowed ────→ Pod B │
│                                               │
│   ✅ Ingress: ALLOW ALL                      │
│   ✅ Egress:  ALLOW ALL                      │
│                                               │
└──────────────────────────────────────────────┘
```

### Default Deny Policies

You can create a **default policy** for a namespace, meaning that you do **not** target any specific pods. By not targeting any pods, this applies to **all pods** in the namespace.

#### Default Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}          # Matches ALL pods (empty selector)
  policyTypes:
  - Ingress                # Deny all ingress (no 'from' rules defined)
```

```
After default-deny-ingress:
┌──────────────────────────────────────────────┐
│                                               │
│   Pod A ←─── ❌ INGRESS DENIED ────── Pod B  │
│                                               │
│   ❌ Ingress: DENY ALL                       │
│   ✅ Egress:  ALLOW ALL (unchanged)          │
│                                               │
└──────────────────────────────────────────────┘
```

#### Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}          # Matches ALL pods (empty selector)
  policyTypes:
  - Egress                 # Deny all egress (no 'to' rules defined)
```

#### Default Deny All Ingress and Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}          # Matches ALL pods (empty selector)
  policyTypes:
  - Ingress                # Deny all ingress
  - Egress                 # Deny all egress
```

### Default Allow Policies

To **allow all** traffic, you specify the `ingress` or `egress` block **without targets**:

#### Allow All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                     # Empty rule = allow all ingress
```

#### Allow All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - {}                     # Empty rule = allow all egress
```

> 💡 **Note**: Any combination of these policy types can be applied together in one single policy depending on how you wish to manage them.

---

## 6. Benefits of Network Policies

### 1. Enhanced and Granular Security

Network policies provide **granular security** to the network:

```
┌──────────────────────────────────────────────────────────────┐
│              Granular Security with Network Policies          │
│                                                               │
│  ┌─────────┐    ✅ Allowed     ┌─────────┐                  │
│  │Frontend │ ────────────────→ │ API     │                  │
│  │ Pod     │    port 80        │ Pod     │                  │
│  └─────────┘                   └─────────┘                  │
│       │                             │                         │
│       │ ❌ Blocked                   │ ✅ Allowed              │
│       ↓                             ↓                         │
│  ┌─────────┐                 ┌─────────┐                    │
│  │Utility  │                 │Database │                    │
│  │ Pod     │                 │ Pod     │                    │
│  └─────────┘                 └─────────┘                    │
│                                                               │
│  Only necessary communication paths are open                  │
└──────────────────────────────────────────────────────────────┘
```

### 2. Workload Isolation

They provide a means of **isolation** for certain workloads. Some of your applications may require higher security than others. This allows them to run **under the same Kubernetes cluster** while maintaining isolation.

### 3. Compliance

Network policies provide a way to be **compliant** for data and security privacy:

| Regulation | How Network Policies Help |
|------------|---------------------------|
| **GDPR** | Isolate data-processing workloads from unauthorized access |
| **HIPAA** | Restrict access to healthcare data to authorized services only |
| **PCI-DSS** | Segment cardholder data environments from other networks |

Network policies also help **reduce the attack surface** and can help restrict communication to **only what is necessary** (principle of least privilege).

### 4. Consistency in Implementation

Network policies provide a means of **consistency** in implementation. If an application is **misconfigured**, network policies can **guard against it affecting other applications**.

```
┌──────────────────────────────────────────────────────────────┐
│                Misconfiguration Protection                    │
│                                                               │
│  ┌─────────┐                                                 │
│  │ App A   │  ← Misconfigured (trying to reach DB directly) │
│  │ (buggy) │                                                 │
│  └────┬────┘                                                 │
│       │                                                      │
│       │ ❌ BLOCKED by Network Policy                         │
│       ↓                                                      │
│  ┌─────────┐                                                 │
│  │Database │  ← Protected from unauthorized access           │
│  │ Pod     │                                                 │
│  └─────────┘                                                 │
│                                                               │
│  ✅ App B is NOT affected by App A's misconfiguration        │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Network Policy Recap

Network policies can be used to control **ingress and egress flow** on Pods:

| Targeting Method | Example |
|-----------------|---------|
| **Specific Pods** | `podSelector: matchLabels: {app: nginx}` |
| **Specific Namespaces** | `namespaceSelector: matchLabels: {env: prod}` |
| **Specific IP CIDR blocks** | `ipBlock: {cidr: 10.0.0.0/8}` |
| **Specific port** | `ports: [{port: 80, protocol: TCP}]` |
| **Port range** (Kubernetes 1.25+) | `ports: [{port: 80, endPort: 443}]` |

---

## 8. Limitations of Network Policies

It is important to understand the **limits** of Kubernetes Network Policies:

### What Network Policies Cannot Do

| Limitation | Description |
|------------|-------------|
| **Cannot force traffic through a common gateway** | Unlike a service mesh, NetworkPolicies cannot route traffic through a sidecar proxy |
| **Cannot handle TLS** | No TLS termination, inspection, or management |
| **Cannot limit traffic at a node level** | Policies apply to pods, not nodes |
| **No security event logging** | Currently no built-in logging for deny/allow events |
| **Cannot prevent loopback traffic** | Cannot block traffic on the loopback device or from the host |
| **Cannot target Service IPs** | Only targets labels of namespaces or pods, not Service ClusterIPs |
| **Cannot target Layer 7** | Cannot filter HTTP paths, methods, headers (HTTP/HTTPS) |

```
┌──────────────────────────────────────────────────────────────┐
│           What NetworkPolicies CANNOT Do                      │
│                                                               │
│  ❌ Force traffic through gateway (service mesh territory)   │
│  ❌ Handle TLS (termination, inspection)                     │
│  ❌ Limit traffic at node level                              │
│  ❌ Log security events (deny/allow)                         │
│  ❌ Prevent loopback / host traffic                          │
│  ❌ Target Service IPs (ClusterIP)                           │
│  ❌ Target Layer 7 (HTTP paths, methods, headers)            │
│                                                               │
│  Layer Coverage:                                              │
│  ┌────────────────────────────────────┐                      │
│  │ Layer 7 (HTTP, gRPC)    ❌         │                      │
│  │ Layer 4 (TCP/UDP Ports) ✅         │                      │
│  │ Layer 3 (IP Addresses)  ✅         │                      │
│  └────────────────────────────────────┘                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. CNI-Specific Enhanced Network Policies

Some CNIs provide **enhanced network policies** that aid in tasks that Kubernetes NetworkPolicies cannot.

### Extended Policy Comparison

| Capability | K8s NetworkPolicy | Calico | Cilium | Istio |
|------------|-------------------|--------|--------|-------|
| **L3/L4 Policies** | ✅ | ✅ | ✅ | ✅ |
| **L7 Policies (HTTP)** | ❌ | ✅ | ✅ | ✅ |
| **Global Policies** | ❌ | ✅ | ✅ | ✅ |
| **Gateway Routing** | ❌ | ❌ | ✅ | ✅ |
| **TLS Management** | ❌ | ❌ | Partial | ✅ |
| **DNS-based Rules** | ❌ | ✅ | ✅ | ✅ |

### Calico Extended Policies

- Layer 7 policy support
- Global network policies (cluster-wide)
- More flexible selectors

### Cilium Extended Policies

- Layer 7 policy enforcement via eBPF
- HTTP-aware policies (path, method, headers)
- Identity-based policies
- Global policies across namespaces

### Istio Extended Policies

- Service mesh integration
- Traffic routing through gateways
- mTLS management
- Authorization policies at L7

> 💡 **Note**: CNI-specific enhanced policies will be covered in more detail in later sections.

---

## 10. Hands-On Demo: Network Policies in Action

### Demo Overview

In this demo, we will:

1. Start with default (no policy) behavior — all traffic allowed
2. Apply a **default deny egress** policy
3. Apply a **default deny ingress** policy
4. Create **allow rules** for specific pod-to-pod communication

### Initial Setup

```
┌──────────────────────────────────────────────────────────────┐
│                  Default Namespace Setup                       │
│                                                               │
│  ┌──────────────────┐                                         │
│  │ management-pod-1 │  ← CentOS (used for testing)           │
│  │ app=centos       │                                         │
│  └──────────────────┘                                         │
│                                                               │
│  ┌──────────────────┐                                         │
│  │ pod-2            │  ← NGINX (web server)                   │
│  │ app=nginx        │                                         │
│  └──────────────────┘                                         │
│                                                               │
│  ┌──────────────────┐                                         │
│  │ pod-3            │  ← NGINX (web server)                   │
│  │ app=nginx        │                                         │
│  └──────────────────┘                                         │
│                                                               │
│  All pods are part of the same application                    │
│  management-pod-1 is used to test connections                 │
└──────────────────────────────────────────────────────────────┘
```

### Step 1: Verify Default Behavior (All Traffic Allowed)

```bash
# Get pods in default namespace
kubectl get pods -o wide

# Get a pod IP from kube-system namespace
kubectl get pods -n kube-system -o wide

# Test egress from management-pod-1 to google.com
kubectl exec management-pod-1 -- curl -s https://google.com

# Test egress from management-pod-1 to a kube-system pod
kubectl exec management-pod-1 -- ping -c 3 <kube-system-pod-ip>
```

**Expected Output:**
```
# All connections succeed — by default, pods are wide open
# All egress and ingress is allowed (no isolation)
```

### Step 2: Apply Default Deny Egress

```yaml
# default-deny-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

```bash
# Apply the policy
kubectl apply -f default-deny-egress.yaml

# Verify the policy
kubectl describe networkpolicy default-deny-egress
```

**Verify Egress Is Blocked:**
```bash
# Test egress to google.com (should timeout)
kubectl exec management-pod-1 -- curl -s --connect-timeout 5 https://google.com

# Test egress to kube-system pod (should timeout)
kubectl exec management-pod-1 -- ping -c 3 <kube-system-pod-ip>
```

**Expected Output:**
```
# Both commands timeout — egress is now fully blocked
# No data can be transferred outside the default namespace
```

```
After default-deny-egress:
┌──────────────────────────────────────────────┐
│                                               │
│   management-pod-1 ── ❌ ──→ google.com       │
│   management-pod-1 ── ❌ ──→ kube-system pod  │
│                                               │
│   ❌ Egress: DENY ALL                        │
│   ✅ Ingress: ALLOW ALL (not affected)        │
│                                               │
└──────────────────────────────────────────────┘
```

### Step 3: Apply Default Deny Ingress

```yaml
# default-deny-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
# Apply the policy
kubectl apply -f default-deny-ingress.yaml

# Verify the policy
kubectl describe networkpolicy default-deny-ingress
```

**Verify Ingress Is Blocked:**

Since we've denied all egress to all pods in the default namespace, we'll launch a test pod in the **kube-system** namespace to test ingress:

```bash
# Get the pod IP of the NGINX pod
NGINX_IP=$(kubectl get pod pod-2 -o jsonpath='{.status.podIP}')
echo "NGINX Pod IP: ${NGINX_IP}"

# Launch a test pod in kube-system namespace
kubectl run test-nginx --image=busybox --restart=Never -n kube-system -- sleep 3600

# Test ingress from kube-system to default namespace NGINX pod
kubectl exec test-nginx -n kube-system -- wget -qO- --timeout=5 http://${NGINX_IP}
```

**Expected Output:**
```
# Connection timeout — ingress is now fully blocked
```

```
After default-deny-ingress:
┌──────────────────────────────────────────────┐
│                                               │
│   kube-system pod ── ❌ ──→ pod-2 (nginx)    │
│                                               │
│   ❌ Egress:  DENY ALL                       │
│   ❌ Ingress: DENY ALL                       │
│                                               │
│   Pods in default namespace are fully isolated│
└──────────────────────────────────────────────┘
```

### Step 4: Allow Egress to Specific Pods

Now we need `management-pod-1` (CentOS) to be able to talk to the NGINX pods. We need to do **two things**:

1. **Allow egress** from CentOS pods to NGINX pods on port 80
2. **Allow ingress** on NGINX pods from CentOS pods on port 80

```yaml
# allow-egress-to-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: centos
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: nginx
    ports:
    - protocol: TCP
      port: 80
```

```bash
# Apply the egress allow policy
kubectl apply -f allow-egress-to-nginx.yaml
```

### Step 5: Allow Ingress on NGINX Pods

```yaml
# allow-ingress-from-centos.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-centos
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
          app: centos
    ports:
    - protocol: TCP
      port: 80
```

```bash
# Apply the ingress allow policy
kubectl apply -f allow-ingress-from-centos.yaml
```

### Step 6: Verify Communication

```bash
# Get NGINX pod IP
NGINX_IP=$(kubectl get pod pod-2 -o jsonpath='{.status.podIP}')

# Test connection from CentOS pod to NGINX pod
kubectl exec management-pod-1 -- curl -s http://${NGINX_IP}
```

**Expected Output:**
```html
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

✅ **Connection successful!** The CentOS pod can now reach the NGINX pod on port 80.

### Step 7: Verify Port Restriction

```bash
# Test connection on a different port (e.g., 8080)
kubectl exec management-pod-1 -- curl -s --connect-timeout 5 http://${NGINX_IP}:8080
```

**Expected Output:**
```
# Connection timeout — the network policy denied the connection on port 8080
```

```
Port Restriction Verification:
┌──────────────────────────────────────────────┐
│                                               │
│   management-pod-1 ── ✅ ──→ pod-2 :80      │
│   management-pod-1 ── ❌ ──→ pod-2 :8080    │
│                                               │
│   Only port 80 is allowed (as defined)        │
│   Port 8080 is blocked by the policy          │
│                                               │
└──────────────────────────────────────────────┘
```

### Demo Recap

```
┌──────────────────────────────────────────────────────────────┐
│                 Demo Network Policy Flow                      │
│                                                               │
│  Step 1: ✅ All traffic allowed (default)                    │
│  Step 2: ❌ Applied default-deny-egress                      │
│  Step 3: ❌ Applied default-deny-ingress                     │
│  Step 4: ✅ Allowed egress from CentOS → NGINX (port 80)    │
│  Step 5: ✅ Allowed ingress from CentOS → NGINX (port 80)   │
│  Step 6: ✅ Verified connection works (port 80)              │
│  Step 7: ❌ Verified port 8080 is blocked                    │
│                                                               │
│  Final State:                                                 │
│  ┌──────────┐   port 80 ✅    ┌──────────┐                  │
│  │ CentOS   │ ─────────────→ │ NGINX    │                    │
│  │ (mgmt)   │                 │ pods     │                    │
│  └──────────┘                 └──────────┘                    │
│       │                            ↑                          │
│       ❌ All other egress          ❌ All other ingress        │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Key Takeaways

✅ **Default Behavior**: All ingress and egress traffic is allowed unless a Network Policy is applied

✅ **Namespace-Level**: Network Policies are defined at the namespace level

✅ **Ingress & Egress**: Control both incoming and outgoing traffic

✅ **Three Selectors**: Target pods by `podSelector`, `namespaceSelector`, or `ipBlock`

✅ **Port-Level Control**: Restrict traffic to specific ports or port ranges (Kubernetes 1.25+)

✅ **Default Deny**: Use empty selectors to deny all traffic to/from all pods in a namespace

✅ **CNI Dependent**: Network Policies are implemented by CNI plugins — they won't work without one

✅ **Layer 4 Only**: Standard Network Policies operate at Layer 3/4 only, not Layer 7

✅ **CNI Extensions**: For Layer 7 policies, use CNI-specific extensions (Calico, Cilium, Istio)

✅ **Compliance**: Network policies help meet regulatory requirements (GDPR, HIPAA, PCI-DSS)

---

## 12. Exam & Real-World Tips

- **Empty pod selector** (`podSelector: {}`) targets **all pods** in the namespace
- **No `from`/`to` rules** with the policy type specified = **deny all** for that direction
- **Empty `from`/`to` rules** (`- {}`) = **allow all** for that direction
- Network Policies are **additive** — multiple policies can apply to the same pod
- If **no policies** exist for a pod, **all traffic is allowed**
- Network Policies **cannot target Service IPs** — only pod labels and namespaces
- Always test with **both directions** (egress on source + ingress on destination)
- Use `kubectl describe networkpolicy <name>` to verify policy rules

---

**Next Topics to Learn:**
- CNI-Specific Network Policies (Calico, Cilium)
- Layer 7 Policy Enforcement
- Service Mesh and Authorization Policies
- Network Policy Logging and Monitoring
- Advanced Network Policy Patterns

---

_End of Network Policy Overview_