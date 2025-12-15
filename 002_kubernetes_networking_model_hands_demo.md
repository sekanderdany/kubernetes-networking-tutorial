# Kubernetes Networking Model – Hands‑on Lab

This lab demonstrates **how Kubernetes networking actually works at Pod and container level**, focusing on:
- Shared network namespaces inside a Pod
- IP‑per‑Pod model
- Pod‑to‑Pod communication
- Role of CNI in creating network namespaces and interfaces

---

## Lab Objectives

By the end of this lab, you will understand:

- How containers in the **same Pod share one network namespace**
- Why containers in a Pod have the **same IP & MAC address**
- How Pods get **unique IPs across the cluster**
- How **Pod‑to‑Pod communication** works without NAT
- How **CNI creates network namespaces and veth interfaces**

---

## Lab Setup Overview

We will create:

- **Pod 1** → 3 containers
  - 1 × NGINX container
  - 2 × utility containers (`sleep`)

- **Pod 2** → 1 container
  - 1 × NGINX container

We will then:
- Inspect Linux network namespaces
- Inspect interfaces created by CNI
- Test container‑to‑container communication (localhost)
- Test Pod‑to‑Pod communication (Pod IP)

---

## Step 1: Create Pod Manifests

### Pod 1 – Multi‑container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: sleep1
    image: busybox
    command: ["sh", "-c", "sleep 86400"]
  - name: sleep2
    image: busybox
    command: ["sh", "-c", "sleep 86400"]
```

---

### Pod 2 – Single‑container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

---

## Step 2: Deploy the Pods

```bash
kubectl apply -f pod1.yaml
kubectl apply -f pod2.yaml
```

Verify:

```bash
kubectl get pods
```

Expected output:

```
pod1   3/3   Running
pod2   1/1   Running
```

---

## Step 3: SSH Into the Node (Where Pods Are Running)

```bash
kubectl get pods -o wide
```

Identify the node IP and SSH:

```bash
ssh <node-ip>
```

> ⚠️ From this point, we are working at the **Linux host level**, not Kubernetes abstractions.

---

## Step 4: List Linux Network Namespaces

```bash
lsns -t net
```

### Key Observations

- You will see **multiple network namespaces**
- Each Pod has **exactly one network namespace**
- Containers inside the same Pod **share one namespace**

---

## Step 5: Identify Container Processes

```bash
lsns -t pid
```

Look for:

- `nginx` processes
- `sleep` processes

You will notice:
- 3 container processes mapped to **2 unique network namespaces**
  - 1 namespace for `pod1`
  - 1 namespace for `pod2`

This proves:
> **Multiple containers in a Pod → one shared network namespace**

---

## Step 6: Inspect Network Interfaces Created by CNI

On the node:

```bash
ip addr
```

Look for interfaces starting with:

```
veth*
```

### Important Notes

- These **veth pairs** are created by the CNI plugin
- One end lives in the **node namespace**
- The other end lives in the **Pod network namespace**

---

## Step 7: Enter Pod Network Namespace

Find the PID of a container in `pod1`:

```bash
ps aux | grep nginx
```

Enter its network namespace:

```bash
nsenter -t <PID> -n
```

Now run:

```bash
ip addr
```

### You Will See

- A single interface (usually `eth0`)
- An IP address → **Pod IP**

This IP is:
- Shared by **all containers in the Pod**
- Assigned by the **CNI plugin**

---

## Step 8: Validate Shared Network Namespace

From **container 1** in pod1:

```bash
kubectl exec pod1 -c sleep1 -- ip addr
```

From **container 2** in pod1:

```bash
kubectl exec pod1 -c sleep2 -- ip addr
```

✅ Output will be **identical**:
- Same IP
- Same MAC address

---

## Step 9: Container‑to‑Container Communication (Same Pod)

NGINX is running on port 80 inside pod1.

From another container in the same Pod:

```bash
kubectl exec pod1 -c sleep1 -- wget -qO- http://localhost:80
```

✅ Works because:
- Containers share the same network namespace
- `localhost` resolves to the Pod network stack

---

## Step 10: Pod‑to‑Pod Communication

Get Pod IPs:

```bash
kubectl get pods -o wide
```

Example:

```
pod1   10.244.0.5
pod2   10.244.0.6
```

From **pod1**, access **pod2**:

```bash
kubectl exec pod1 -c sleep1 -- wget -qO- http://10.244.0.6:80
```

✅ This works because:
- Kubernetes networking model requires **all Pods be reachable**
- No NAT is used
- CNI handles routing across nodes

---

## Step 11: What This Lab Proves

- **IP‑per‑Pod model** is real, not theoretical
- Containers in a Pod:
  - Share IP
  - Share MAC
  - Share ports
- Pods communicate directly using IP addresses
- CNI is responsible for:
  - Network namespace creation
  - veth pair creation
  - IP allocation
  - Routing

---

## Architect‑Level Notes (Very Important)

- Kubernetes itself does **not implement networking**
- It only defines the **rules**
- CNI plugins implement the **how**
- Debugging networking issues almost always means:
  - Inspecting namespaces
  - Inspecting routes
  - Inspecting CNI configuration

---

## Common Interview / Exam Traps

- ❌ Containers get IPs → **Wrong**
- ✅ Pods get IPs → **Correct**

- ❌ Pods use NAT → **Wrong**
- ✅ Flat network model → **Correct**

---

## Next Learning Steps

- kube‑proxy & Services
- Pod‑to‑Service routing
- Network Policies
- Cilium vs Calico (eBPF vs iptables)

---

_End of Kubernetes Networking Model – Hands‑on Lab_

