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

## Phase 1: Create and Deploy the Pods

### Step 1: Create Pod Manifests

#### Pod 1 – Multi‑container Pod

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

#### Pod 2 – Single‑container Pod

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

### Step 2: Deploy the Pods and Verify

```bash
kubectl apply -f pod1.yaml
kubectl apply -f pod2.yaml
```

Verify:

```bash
k get pods -o wide
```

Expected output:

```
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
pod1   3/3     Running   0          22s   172.17.3.2   node02   <none>           <none>
pod2   1/1     Running   0          16s   172.17.1.2   node01   <none>           <none>
```

---

## Phase 2: Host-Level Namespace Inspection

### Step 3: SSH Into the Node (Where Pods Are Running)

```bash
kubectl get pods -o wide
```

Identify the node IP and SSH:

```bash
ssh node01
```

> ⚠️ From this point, we are working at the **Linux host level**, not Kubernetes abstractions.

---

### Step 4: List Linux Network Namespaces

```bash
lsns -t pid
```

```
node01 ~ ➜  lsns -t pid
        NS TYPE NPROCS   PID USER  COMMAND
4026540666 pid       1  2270 65535 /pause
4026540669 pid       1  2277 65535 /pause
4026540671 pid       1  2429 root  /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=node01
4026540743 pid      18     1 root  /sbin/init --log-level=err
4026543251 pid       9  2838 root  /usr/local/bin/runsvdir -P /etc/service/enabled
4026543253 pid       1  3104 root  /opt/bin/flanneld --ip-masq --kube-subnet-mgr
4026547847 pid       1 55180 65535 /pause
4026548355 pid      17 55484 root  nginx: master process nginx -g daemon off;
```

To identify which network namespace these processes belong to, run:
```bash
ip netns identify 55484
```

```
cni-70c4ce0e-d58e-483f-e4aa-429e281a7fe8
```

Key observations:

- You will see **multiple network namespaces**
- Each Pod has **exactly one network namespace**
- Containers inside the same Pod **share one namespace**

---

### Step 5: Map Container Processes to Namespaces

```bash
lsns -t pid
```

Look for:

- `nginx` processes
- `sleep` processes

```
node01 ~ ➜  lsns -t pid
        NS TYPE NPROCS   PID USER  COMMAND
4026540666 pid       1  2270 65535 /pause
4026540669 pid       1  2277 65535 /pause
4026540671 pid       1  2429 root  /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=node01
4026540743 pid      20     1 root  /sbin/init --log-level=err
4026543251 pid       9  2838 root  /usr/local/bin/runsvdir -P /etc/service/enabled
4026543253 pid       1  3104 root  /opt/bin/flanneld --ip-masq --kube-subnet-mgr
4026547847 pid       1 55180 65535 /pause
4026548355 pid      17 55484 root  nginx: master process nginx -g daemon off;

node02 ~ ➜  lsns -t pid
        NS TYPE NPROCS   PID USER  COMMAND
4026540255 pid      18     1 root  /sbin/init --log-level=err
4026543244 pid       1  2331 65535 /pause
4026543247 pid       1  2338 65535 /pause
4026543249 pid       1  2489 root  /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=node02
4026543360 pid       9  2875 root  /usr/local/bin/runsvdir -P /etc/service/enabled
4026543362 pid       1  3145 root  /opt/bin/flanneld --ip-masq --kube-subnet-mgr
4026545674 pid       1 55076 65535 /pause
4026547841 pid      17 55403 root  nginx: master process nginx -g daemon off;
4026547843 pid       1 55561 root  sleep 86400
4026547849 pid       1 55662 root  sleep 86400
```

You will notice:
- 3 container processes mapped to **2 unique network namespaces**
  - 1 namespace for `pod1`
  - 1 namespace for `pod2`

```
node02 ~ ➜  for i in 55403 55561 55662; do ip netns identify $i; done
cni-1287ee12-6929-c929-66c4-343aa343626c
cni-1287ee12-6929-c929-66c4-343aa343626c
cni-1287ee12-6929-c929-66c4-343aa343626c

node01 ~ ➜  ip netns identify 55484
cni-70c4ce0e-d58e-483f-e4aa-429e281a7fe8
```

This proves:
> **Multiple containers in a Pod → one shared network namespace**

---

## Phase 3: Pod Network Details

### Step 6: Inspect Interfaces in the Pod Network Namespace

```
node01 ~ ➜  ip netns exec cni-70c4ce0e-d58e-483f-e4aa-429e281a7fe8 ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP group default qlen 1000
    link/ether c6:d9:c6:b3:de:82 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.1.2/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c4d9:c6ff:feb3:de82/64 scope link 
       valid_lft forever preferred_lft forever


node02 ~ ➜  ip netns exec cni-1287ee12-6929-c929-66c4-343aa343626c ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP group default qlen 1000
    link/ether 06:09:a6:c1:fd:9f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.3.2/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::409:a6ff:fec1:fd9f/64 scope link 
       valid_lft forever preferred_lft forever

```

Now if we see the container ip address we will see the same ip address as above:

```
controlplane ~ ✖ k exec -it pod1 -c sleep1 -- ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1410 qdisc noqueue qlen 1000
    link/ether 06:09:a6:c1:fd:9f brd ff:ff:ff:ff:ff:ff
    inet 172.17.3.2/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::409:a6ff:fec1:fd9f/64 scope link 
       valid_lft forever preferred_lft forever

controlplane ~ ➜  k exec -it pod1 -c sleep2 -- ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1410 qdisc noqueue qlen 1000
    link/ether 06:09:a6:c1:fd:9f brd ff:ff:ff:ff:ff:ff
    inet 172.17.3.2/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::409:a6ff:fec1:fd9f/64 scope link 
       valid_lft forever preferred_lft forever
```

---

## Phase 4: Connectivity Tests

### Step 7: Container‑to‑Container Communication (Same Pod)

Now we can actually get the nginx welcome page from both containers in pod1 using localhost:

```
controlplane ~ ✖ kubectl exec -it pod1 -c sleep2 -- wget -qO- http://localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

controlplane ~ ➜  kubectl exec -it pod1 -c sleep1 -- wget -qO- http://localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Step 8: Pod‑to‑Pod Communication (Using Pod IPs From the Lab)

Now lets see how we can reach pod2 from pod1 using pod2's IP address:

get the ip address from both pod using below command:
```bash
controlplane ~ ➜  kubectl get pods -o=jsonpath='{range .items[*]}{"podName: "}{.metadata.name}{"\t"}{"podIP: "}{.status.podIP}{"\n"}{end}'
```
Output:
```
podName: pod1   podIP: 172.17.3.2
podName: pod2   podIP: 172.17.1.2
```

lets try to access pod2 from pod1 using pod2's IP address:

```
controlplane ~ ➜  k exec pod1 -c nginx -- curl 172.17.1.2:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
100   615  100   615    0     0   1115      0 --:--:-- --:--:-- --:--:--  1114
```

---

## Phase 5: CNI Interfaces and Namespace Entry

### Step 9: Inspect Network Interfaces Created by CNI

On the node:

```bash
ip addr
```

Look for interfaces starting with:

```
veth*
```

Important notes:

- These **veth pairs** are created by the CNI plugin
- One end lives in the **node namespace**
- The other end lives in the **Pod network namespace**

---

### Step 10: Enter a Pod Network Namespace

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

You will see:

- A single interface (usually `eth0`)
- An IP address → **Pod IP**

This IP is:
- Shared by **all containers in the Pod**
- Assigned by the **CNI plugin**

---

### Step 11: Validate Shared Network Namespace (Quick Check)

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

### Step 12: Pod‑to‑Pod Communication (Alternate Example)

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

## What This Lab Proves

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

## Final Sets of Commands

Use these consolidated commands at the end of the lab:

```bash
vi pod1.yaml
vi pod2.yaml
kubectl apply -f pod1.yaml
kubectl apply -f pod2.yaml
k get pods -o wide
ssh node01
lsns -t pid
ip netns identify 55484
ip addr
ip netns exec cni-70c4ce0e-d58e-483f-e4aa-429e281a7fe8 ip add
# exit from node01
k get pods
k exec -i pod1 -c sleep1 -- ip add
k exec -i pod1 -c sleep2 -- ip add
kubectl exec pod1 -c sleep1 -- wget -qO- http://localhost:80
kubectl get pods -o=jsonpath='{range .items[*]}{"podName: "}{.metadata.name}{"\t"}{"podIP: "}{.status.podIP}{"\n"}{end}'
kubectl exec pod1 -c nginx -- curl 172.17.1.2:80
```

---

_End of Kubernetes Networking Model – Hands‑on Lab_
