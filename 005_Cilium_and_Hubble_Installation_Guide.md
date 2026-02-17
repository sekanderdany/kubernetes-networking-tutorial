# Cilium and Hubble Installation Guide

This guide provides step-by-step instructions for installing **Cilium** and **Hubble** on Kubernetes clusters using both the **Cilium CLI** and **Helm** methods.

---

## 1. Prerequisites

### System Requirements

Before installing Cilium, ensure your cluster meets these requirements:

#### Minimum Requirements

| Component | Minimum Version | Recommended Version |
|-----------|----------------|---------------------|
| **Kubernetes** | 1.19+ | 1.24+ |
| **Linux Kernel** | 4.19+ | 5.10+ or 5.15+ |
| **Architecture** | amd64, arm64 | amd64, arm64 |
| **Memory per Node** | 2 GB | 4 GB+ |
| **CPU per Node** | 2 cores | 4 cores+ |

#### eBPF Feature Requirements

Cilium requires the following kernel features:

- **BPF syscall support** (`CONFIG_BPF=y`)
- **BPF JIT compilation** (`CONFIG_BPF_JIT=y`)
- **BPF cgroup support** (`CONFIG_CGROUP_BPF=y`)
- **BPF syscall filtering** (`CONFIG_BPF_SYSCALL=y`)

#### Verify Kernel Compatibility

```bash
# Check kernel version
uname -r

# Verify eBPF support
lsmod | grep bpf

# Check BPF features
cat /boot/config-$(uname -r) | grep BPF
```

#### Kubernetes Cluster Requirements

- Cluster must be up and running
- `kubectl` configured and working
- Sufficient permissions to install DaemonSets, CRDs, RBAC resources
- No other CNI plugins installed (or removed)

#### Check Current CNI

```bash
# Check if any CNI is installed
kubectl get pods -A -l k8s-app=calico-node
kubectl get pods -A -l app=flannel
kubectl get pods -A -l k8s-app=canal

# Check CNI configuration
ls /etc/cni/net.d/

# If another CNI is installed, you must remove it first
```

### Tools Required

| Tool | Purpose | Installation |
|------|---------|--------------|
| `kubectl` | Kubernetes CLI | https://kubernetes.io/docs/tasks/tools/ |
| `helm` | Package manager | https://helm.sh/docs/intro/install/ |
| `curl` | Download binaries | Usually pre-installed |
| `sha256sum` | Verify downloads | Usually pre-installed |

---

## 2. Installation Methods Overview

Cilium can be installed using two primary methods:

### Method 1: Cilium CLI (Recommended for Beginners)

**Pros:**
- Simple, one-line installation
- Automatic dependency management
- Built-in validation and checks
- Easy upgrades
- Integrated with Cilium CLI for management

**Cons:**
- Less customization options compared to Helm
- May not fit all complex scenarios

### Method 2: Helm (Recommended for Production)

**Pros:**
- Full customization via values files
- GitOps friendly (Helm charts in Git)
- Better integration with existing CI/CD pipelines
- More control over deployment

**Cons:**
- More complex setup
- Requires Helm knowledge
- Manual configuration of dependencies

### Choosing the Right Method

```
Use Cilium CLI if:
✅ Learning Cilium for the first time
✅ Quick setup for development/testing
✅ Default configuration is sufficient
✅ Want simple upgrade process

Use Helm if:
✅ Production deployment
✅ Need custom configuration
✅ Using GitOps (ArgoCD, Flux)
✅ Integrating with existing Helm workflows
✅ Need fine-grained control over deployment
```

---

## 3. Installation via Cilium CLI

### Step 1: Install Cilium CLI

The Cilium CLI (`cilium`) is used to install and manage Cilium on your Kubernetes cluster.

#### Download and Install Cilium CLI

```bash
# Get the latest stable version
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
echo "Installing Cilium CLI version: ${CILIUM_CLI_VERSION}"

# Detect system architecture
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
echo "System architecture: ${CLI_ARCH}"

# Download Cilium CLI binary and checksum
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Verify the download integrity
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum

# Extract and install to /usr/local/bin
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin

# Clean up downloaded files
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Verify installation
cilium version
```

**Expected Output:**
```
cilium-cli: v0.16.x
```

#### Verify Installation

```bash
# Check cilium version
cilium version

# Check cilium help
cilium --help

# List available commands
cilium --help | grep -A 20 "Available Commands"
```

### Step 2: Install Cilium CLI

The Cilium CLI can also be installed directly from the repository:

```bash
# Clone the repository
git clone https://github.com/cilium/cilium-cli.git
cd cilium-cli

# Build and install (requires Go)
make install
```

**Note:** This method requires Go installed on your system.

### Step 3: Install Hubble CLI

Hubble CLI (`hubble`) provides command-line access to Hubble observability features.

#### Download and Install Hubble CLI

```bash
# Get the latest stable version
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
echo "Installing Hubble CLI version: ${HUBBLE_VERSION}"

# Detect system architecture
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
echo "System architecture: ${HUBBLE_ARCH}"

# Download Hubble CLI binary and checksum
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}

# Verify the download integrity
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum

# Extract and install to /usr/local/bin
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin

# Clean up downloaded files
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}

# Verify installation
hubble version
```

**Expected Output:**
```
Hubble CLI: v0.12.x
```

### Step 4: Install Cilium on Kubernetes Cluster

Now use the Cilium CLI to install Cilium on your Kubernetes cluster.

#### Basic Installation

```bash
# Install Cilium with default settings
cilium install --version 1.15.0 --wait
```

**What this command does:**
- Downloads Cilium manifests
- Installs Cilium DaemonSet
- Enables Hubble Relay
- Waits for installation to complete

**Expected Output:**
```
🔮 Auto-detected Kubernetes kind: kind
✨ Happy to help you find the perfect Cilium edition for your cluster
ℹ️  Cilium version 1.15.0
🔮 Auto-detected cluster name: cilium-k8s
ℹ️  Using K8s API server URL: https://127.0.0.1:6443
ℹ️  Cilium will be installed in namespace kube-system
✨ Generated Cilium installation manifest
ℹ️  Installing Cilium in cluster...
✅ Cilium was successfully installed! Run 'cilium status' to view installation health.
```

#### Installation with Specific Version

```bash
# Install a specific version of Cilium
cilium install --version 1.15.0 --wait
```

#### Installation with Hubble UI Enabled

```bash
# Install Cilium with Hubble UI
cilium install --version 1.15.0 --set hubble-ui.enabled=true --wait
```

#### Installation with Custom Settings

```bash
# Install with custom configuration
cilium install --version 1.15.0 \
  --set kubeProxyReplacement=true \
  --set hubble.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true \
  --wait
```

### Step 5: Verify Cilium Installation

After installation, verify that Cilium is running correctly.

#### Check Cilium Status

```bash
# Check overall Cilium status
cilium status
```

**Expected Output:**
```
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Hubble Relay:       OK
 \__/¯¯\__/    ClusterMesh:        disabled
    \__/        Datapath Mode:      veth
    CNI (v1):  OK
    IPv4:       10.244.0.0/16
    IPv6:       disabled
```

#### Check Cilium Pods

```bash
# Check Cilium pods are running
kubectl get pods -n kube-system -l k8s-app=cilium

# Check Hubble pods
kubectl get pods -n kube-system -l k8s-app=hubble-relay

# Check Cilium Operator
kubectl get pods -n kube-system -l name=cilium-operator
```

**Expected Output:**
```
NAME           READY   STATUS    RESTARTS   AGE
cilium-xxx     1/1     Running   0          2m
cilium-xxx     1/1     Running   0          2m
cilium-operator-xxx  1/1     Running   0          2m
hubble-relay-xxx     1/1     Running   0          2m
```

#### Check Cilium DaemonSet

```bash
# Check Cilium DaemonSet status
kubectl get ds cilium -n kube-system

# Check pods are ready on all nodes
kubectl get nodes -o wide
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
```

### Step 6: Run Connectivity Test

Cilium provides a built-in connectivity test to verify the installation.

#### Run Cilium Connectivity Test

```bash
# Run the connectivity test
cilium connectivity test
```

**What this test does:**
1. Deploys test pods
2. Tests pod-to-pod communication
3. Tests service connectivity
4. Tests external connectivity
5. Tests network policies
6. Tests ingress/egress

**Expected Output:**
```
ℹ️  Deploying connectivity test...
ℹ️  Waiting for test pods to become ready...
ℹ️  Running tests...
✅ All tests passed successfully!
```

#### Run Specific Test Scenarios

```bash
# Run connectivity test with verbose output
cilium connectivity test --verbose

# Run connectivity test with specific namespace
cilium connectivity test --namespace cilium-test
```

### Step 7: Enable Hubble UI (Optional)

If you didn't enable Hubble UI during installation, you can enable it now.

```bash
# Enable Hubble UI
cilium hubble enable --ui

# Port forward to access Hubble UI
cilium hubble ui
```

**Access Hubble UI:**
- Open browser: `http://localhost:12000`

---

## 4. Installation via Helm

### Step 1: Install Helm

If you don't have Helm installed, install it first.

#### Install Helm on Linux

```bash
# Download Helm binary
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

#### Install Helm via Package Manager

```bash
# On Ubuntu/Debian
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# On macOS
brew install helm
```

### Step 2: Add Cilium Helm Repository

Add the Cilium Helm repository to your Helm configuration.

```bash
# Add Cilium Helm repository
helm repo add cilium https://helm.cilium.io/

# Update repository
helm repo update

# List available versions
helm search repo cilium/cilium --versions | head -10
```

### Step 3: Create Values File (Recommended)

Create a custom values file for your deployment.

#### Basic Values File

Create `cilium-values.yaml`:

```yaml
# Basic Cilium configuration
image:
  repository: cilium/cilium
  tag: v1.15.0
  pullPolicy: IfNotPresent

operator:
  image:
    repository: cilium/operator
    tag: v1.15.0
    pullPolicy: IfNotPresent

# Enable Hubble
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    serviceType: LoadBalancer

# Enable Hubble metrics
metrics:
  enabled:
  - hubble
  - policy
  - drop
```

#### Production Values File

Create `cilium-production.yaml`:

```yaml
# Production Cilium configuration
image:
  repository: cilium/cilium
  tag: v1.15.0
  pullPolicy: IfNotPresent

operator:
  image:
    repository: cilium/operator
    tag: v1.15.0
    pullPolicy: IfNotPresent

# Enable kube-proxy replacement
kubeProxyReplacement: true

# Enable Hubble with all features
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    serviceType: LoadBalancer
  metrics:
    enabled:
    - dns:query
    - drop
    - tcp
    - flow
    - icmp
    - http

# Enable all metrics
prometheus:
  enabled: true
  serviceMonitor:
    enabled: true

# Enable bandwidth manager
bandwidthManager:
  enabled: true

# Enable BPF masquerading
bpfMasquerade:
  enabled: true

# Enable native routing
nativeRoutingCIDR: 10.244.0.0/16

# IPv4 configuration
ipv4:
  enabled: true

# IPv6 configuration (optional)
ipv6:
  enabled: false

# CNI configuration
cni:
  chainingMode: none
  customConf: false

# Enable security awareness
securityContext:
  privileged: true

# Resource limits
resources:
  limits:
    cpu: 4000m
    memory: 2Gi
  requests:
    cpu: 100m
    memory: 256Mi
```

### Step 4: Install Cilium via Helm

Install Cilium using Helm with your custom values file.

#### Basic Helm Installation

```bash
# Install Cilium with default values
helm install cilium cilium/cilium --namespace kube-system
```

#### Install with Custom Values

```bash
# Install Cilium with custom values
helm install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.15.0 \
  --values cilium-values.yaml
```

#### Install with Specific Configuration

```bash
# Install with inline configuration
helm install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.15.0 \
  --set image.tag=v1.15.0 \
  --set hubble.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true \
  --set kubeProxyReplacement=true
```

### Step 5: Verify Helm Installation

Verify that Cilium was installed correctly.

#### Check Helm Release

```bash
# Check Cilium release status
helm status cilium -n kube-system

# List all Helm releases
helm list -n kube-system
```

#### Check Cilium Pods

```bash
# Check all Cilium-related pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium

# Check Hubble pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=hubble
```

#### Check Cilium Status

```bash
# Use Cilium CLI if installed
cilium status

# Or check DaemonSet directly
kubectl get ds cilium -n kube-system
kubectl get deploy cilium-operator -n kube-system
```

### Step 6: Upgrade Cilium via Helm

Upgrading Cilium with Helm is straightforward.

#### Upgrade to New Version

```bash
# Upgrade Cilium to a new version
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version 1.15.0 \
  --values cilium-values.yaml \
  --reuse-values
```

#### Upgrade with New Values

```bash
# Upgrade with updated values file
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values cilium-production.yaml
```

---

## 5. Hubble Configuration

### Enable Hubble

Hubble is Cilium's observability platform.

#### Enable Hubble with Cilium CLI

```bash
# Enable Hubble Relay
cilium hubble enable

# Enable Hubble UI
cilium hubble enable --ui

# Enable Hubble with specific configuration
cilium hubble enable --ui --listen-address=:4244
```

#### Enable Hubble with Helm

Hubble can be enabled during installation via Helm values:

```yaml
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    serviceType: LoadBalancer
```

### Access Hubble UI

#### Port Forwarding

```bash
# Port forward to Hubble UI
cilium hubble ui

# Or use kubectl
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Access Hubble UI at: `http://localhost:12000`

#### LoadBalancer (Cloud)

If using a LoadBalancer service:

```bash
# Get Hubble UI service URL
kubectl get svc -n kube-system hubble-ui
```

Access Hubble UI at the external URL.

### Using Hubble CLI

#### Basic Hubble Commands

```bash
# Check Hubble status
hubble status

# Observe all flows
hubble observe

# Observe flows with filters
hubble observe --namespace default --pod frontend

# View dropped flows
hubble observe --verdict dropped

# Show flow summary
hubble observe --summary
```

#### Flow Filtering

```bash
# Filter by namespace
hubble observe --namespace production

# Filter by pod
hubble observe --pod api-server-xxx

# Filter by labels
hubble observe --labels app=nginx,env=production

# Filter by protocol
hubble observe --protocol http

# Filter by destination
hubble observe --to-pod database-xxx

# Filter by port
hubble observe --port 80

# Filter by verdict
hubble observe --verdict dropped

# Filter by L7 protocol
hubble observe --l7 http

# Combine filters
hubble observe \
  --namespace production \
  --labels app=api \
  --protocol http \
  --to-pod database-xxx
```

#### Flow Analysis

```bash
# Show detailed flow information
hubble observe --verbose

# Show flow statistics
hubble observe --summary

# Export flows to JSON
hubble observe --output flows.json

# Export flows to PCAP
hubble observe --format pcap > capture.pcap

# Follow a specific flow
hubble observe --follow
```

---

## 6. Post-Installation Verification

### Verify Cilium Functionality

#### Check Cilium Status

```bash
# Comprehensive status check
cilium status

# Check cluster connectivity
cilium cluster connectivity
```

#### Check Network Policies

```bash
# List all Cilium network policies
cilium networkpolicy list

# Get specific policy details
cilium networkpolicy get <policy-id>

# Check policy enforcement
cilium policy get --labels app=nginx
```

#### Check Identities

```bash
# List all identities
cilium identity list

# Get identity details
cilium identity get <identity-id>

# Check identity mappings
cilium bpf ip list
```

#### Check Endpoints

```bash
# List all endpoints
cilium endpoint list

# Get endpoint details
cilium endpoint get <endpoint-id>

# Check endpoint status
cilium endpoint status
```

### Test Network Connectivity

#### Deploy Test Application

```bash
# Create test namespace
kubectl create namespace cilium-test

# Deploy test pods
kubectl run test-pod-1 -n cilium-test --image=nginx --restart=Never
kubectl run test-pod-2 -n cilium-test --image=nginx --restart=Never

# Wait for pods to be ready
kubectl wait --for=condition=ready pod/test-pod-1 -n cilium-test --timeout=60s
kubectl wait --for=condition=ready pod/test-pod-2 -n cilium-test --timeout=60s
```

#### Test Pod-to-Pod Communication

```bash
# Get pod IPs
POD1_IP=$(kubectl get pod test-pod-1 -n cilium-test -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod test-pod-2 -n cilium-test -o jsonpath='{.status.podIP}')

# Test connectivity from pod1 to pod2
kubectl exec -n cilium-test test-pod-1 -- curl -s http://${POD2_IP}

# Test connectivity from pod2 to pod1
kubectl exec -n cilium-test test-pod-2 -- curl -s http://${POD1_IP}
```

#### Test DNS Resolution

```bash
# Test DNS resolution
kubectl exec -n cilium-test test-pod-1 -- nslookup kubernetes.default

# Test DNS with FQDN
kubectl exec -n cilium-test test-pod-1 -- nslookup kubernetes.default.svc.cluster.local
```

#### Test External Connectivity

```bash
# Test external connectivity
kubectl exec -n cilium-test test-pod-1 -- curl -s https://www.google.com -o /dev/null -w "%{http_code}"

# Test external DNS
kubectl exec -n cilium-test test-pod-1 -- nslookup google.com
```

### Test Network Policies

#### Create Network Policy

```bash
# Create a network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
  namespace: cilium-test
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: allowed
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
EOF
```

#### Test Policy Enforcement

```bash
# Label pods appropriately
kubectl label pod test-pod-1 -n cilium-test app=nginx
kubectl label pod test-pod-2 -n cilium-test role=allowed

# Test with Hubble
hubble observe --namespace cilium-test --verdict dropped
```

### Test Hubble

#### Verify Hubble is Collecting Flows

```bash
# Check Hubble status
hubble status

# Observe flows
hubble observe

# In another terminal, generate traffic
kubectl exec -n cilium-test test-pod-1 -- curl http://${POD2_IP}

# You should see flows in the Hubble observe output
```

#### Test Hubble UI

```bash
# Start Hubble UI
cilium hubble ui

# Open browser to http://localhost:12000

# Verify you can see:
# - Service map
# - Flow visualization
# - Real-time monitoring
```

---

## 7. Advanced Configuration

### Enable Kube-Proxy Replacement

Cilium can replace kube-proxy for better performance.

#### Enable Kube-Proxy Replacement

```bash
# Enable with Cilium CLI
cilium install --version 1.15.0 --set kubeProxyReplacement=true --wait

# Or upgrade existing installation
cilium upgrade --version 1.15.0 --set kubeProxyReplacement=true
```

#### Enable with Helm

```yaml
# In values file
kubeProxyReplacement: true
```

#### Verify Kube-Proxy Replacement

```bash
# Check if kube-proxy is being replaced
kubectl get pods -A | grep kube-proxy

# Check Cilium status
cilium status | grep "Kube-Proxy"
```

### Enable Bandwidth Manager

Cilium's bandwidth manager provides QoS capabilities.

#### Enable Bandwidth Manager

```bash
# Enable with Cilium CLI
cilium install --version 1.15.0 --set bandwidthManager.enabled=true --wait

# Or with Helm
# In values file:
# bandwidthManager:
#   enabled: true
```

#### Test Bandwidth Limits

```bash
# Create a pod with bandwidth limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bandwidth-test
  namespace: cilium-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        kubernetes.io/egress-bandwidth: 10M
        kubernetes.io/ingress-bandwidth: 10M
EOF
```

### Enable Encryption

Cilium supports WireGuard-based encryption for pod traffic.

#### Enable Encryption

```bash
# Enable with Cilium CLI
cilium install --version 1.15.0 --set encryption.enabled=true --wait

# Or with Helm
# In values file:
# encryption:
#   enabled: true
```

#### Verify Encryption

```bash
# Check encryption status
cilium status | grep Encryption

# Test encrypted communication
kubectl exec -n cilium-test test-pod-1 -- curl http://${POD2_IP}
```

### Enable Prometheus Metrics

Cilium can export metrics to Prometheus.

#### Enable Metrics

```bash
# Enable with Cilium CLI
cilium install --version 1.15.0 \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --wait

# Or with Helm
# In values file:
# prometheus:
#   enabled: true
```

#### Access Metrics

```bash
# Port forward to metrics endpoint
kubectl port-forward -n kube-system svc/cilium-metrics 9962:9962

# Access metrics
curl http://localhost:9962/metrics
```

---

## 8. Uninstallation

### Uninstall Cilium (Cilium CLI)

```bash
# Uninstall Cilium using Cilium CLI
cilium uninstall

# Verify removal
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl get ds -n kube-system
```

### Uninstall Cilium (Helm)

```bash
# Uninstall Cilium using Helm
helm uninstall cilium -n kube-system

# Delete CRDs (optional)
kubectl delete crd ciliumendpoints.cilium.io
kubectl delete crd ciliumidentities.cilium.io
kubectl delete crd ciliumnetworkpolicies.cilium.io

# Verify removal
kubectl get pods -n kube-system
```

### Remove Cilium CLI

```bash
# Remove Cilium CLI
sudo rm /usr/local/bin/cilium

# Remove Hubble CLI
sudo rm /usr/local/bin/hubble

# Verify removal
which cilium
which hubble
```

---

## 9. Troubleshooting

### Common Issues and Solutions

#### Issue 1: Cilium Pods Not Starting

**Symptoms:**
- Cilium pods in `CrashLoopBackOff` or `Error` state

**Solutions:**
```bash
# Check pod logs
kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# Check for kernel compatibility
uname -r

# Check node resources
kubectl describe nodes

# Check if other CNI is installed
ls /etc/cni/net.d/
```

#### Issue 2: Connectivity Test Fails

**Symptoms:**
- `cilium connectivity test` fails

**Solutions:**
```bash
# Run test with verbose output
cilium connectivity test --verbose

# Check Cilium status
cilium status

# Check for dropped flows
hubble observe --verdict dropped

# Check network policies
cilium networkpolicy list
```

#### Issue 3: Hubble UI Not Accessible

**Symptoms:**
- Cannot access Hubble UI

**Solutions:**
```bash
# Check Hubble Relay status
kubectl get pods -n kube-system -l k8s-app=hubble-relay

# Check Hubble UI pod
kubectl get pods -n kube-system -l k8s-app=hubble-ui

# Check Hubble status
hubble status

# Restart Hubble components
kubectl rollout restart -n kube-system deployment/hubble-relay
kubectl rollout restart -n kube-system deployment/hubble-ui
```

#### Issue 4: Pods Cannot Communicate

**Symptoms:**
- Pods cannot reach each other

**Solutions:**
```bash
# Check Cilium status
cilium status

# Check for network policies
cilium networkpolicy list

# Observe dropped flows
hubble observe --verdict dropped

# Check pod IPs
kubectl get pods -o wide

# Test from pod
kubectl exec -it <pod-name> -- ping <destination-ip>
```

### Debugging Commands

#### Cilium Debugging

```bash
# Check Cilium logs
kubectl logs -n kube-system -l k8s-app=cilium --follow

# Check Cilium Operator logs
kubectl logs -n kube-system -l name=cilium-operator --follow

# Check Cilium configuration
cilium config view

# Dump eBPF maps
cilium bpf policy list
cilium bpf ip list

# Check cluster mesh status
cilium clustermesh status
```

#### Hubble Debugging

```bash
# Check Hubble Relay logs
kubectl logs -n kube-system -l k8s-app=hubble-relay --follow

# Check Hubble UI logs
kubectl logs -n kube-system -l k8s-app=hubble-ui --follow

# Check Hubble status
hubble status --verbose

# Test Hubble Relay
hubble observe --test
```

#### Network Debugging

```bash
# Check network interfaces
kubectl exec -it <pod-name> -- ip addr

# Check routing table
kubectl exec -it <pod-name> -- ip route

# Test DNS resolution
kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Test external connectivity
kubectl exec -it <pod-name> -- curl -I https://www.google.com
```

### Log Collection

#### Collect Cilium Logs

```bash
# Collect Cilium logs from all nodes
kubectl logs -n kube-system -l k8s-app=cilium --tail=1000 > cilium-logs.txt

# Collect Hubble logs
kubectl logs -n kube-system -l k8s-app=hubble-relay --tail=1000 > hubble-relay-logs.txt
kubectl logs -n kube-system -l k8s-app=hubble-ui --tail=1000 > hubble-ui-logs.txt

# Collect Cilium status
cilium status > cilium-status.txt

# Collect network information
kubectl get pods -A -o wide > all-pods.txt
kubectl get svc -A > all-services.txt
kubectl get networkpolicy -A > all-policies.txt
```

---

## 10. Best Practices

### Production Deployment

1. **Use Helm for Production**
   - Store values in Git
   - Use GitOps for deployment
   - Document all configuration changes

2. **Enable Resource Limits**
   - Set CPU and memory limits
   - Monitor resource usage
   - Scale based on needs

3. **Enable Monitoring**
   - Integrate with Prometheus
   - Set up Grafana dashboards
   - Configure alerts

4. **Enable Logging**
   - Configure log aggregation
   - Set up log retention policies
   - Monitor error logs

5. **Backup Configuration**
   - Version control values files
   - Document installation steps
   - Maintain runbooks

### Security

1. **Enable Network Policies**
   - Use default deny policies
   - Follow principle of least privilege
   - Regularly audit policies

2. **Enable Encryption**
   - Use WireGuard for pod traffic
   - Secure Hubble UI access
   - Use RBAC for API access

3. **Update Regularly**
   - Keep Cilium updated
   - Monitor security advisories
   - Test upgrades in staging first

### Performance

1. **Enable Kube-Proxy Replacement**
   - Better performance
   - Lower latency
   - Reduced resource usage

2. **Enable Bandwidth Manager**
   - QoS for critical workloads
   - Prevent network congestion
   - Improve predictability

3. **Optimize eBPF Maps**
   - Monitor map usage
   - Tune map sizes
   - Avoid map exhaustion

---

## 11. Summary of Commands

### Quick Reference

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}

# Install Cilium
cilium install --version 1.15.0 --wait

# Check status
cilium status

# Run connectivity test
cilium connectivity test

# Observe flows
hubble observe
```

---

## 12. Next Steps

After successfully installing Cilium and Hubble:

1. **Learn Cilium CLI**: Master advanced commands
2. **Configure Network Policies**: Implement zero-trust networking
3. **Set up Monitoring**: Integrate with Prometheus/Grafana
4. **Explore Hubble**: Deep dive into observability
5. **Test Advanced Features**: Service mesh, encryption, etc.

---

**End of Cilium and Hubble Installation Guide**