# Kubernetes Cluster Implementation on AWS EC2

**Project:** Manual Kubernetes Cluster Setup and Minor Version Upgrade  
**Platform:** AWS EC2  
**Operating System:** Ubuntu 26.04.1 LTS  
**Container Runtime:** containerd 2.2.2  
**Cluster Bootstrap:** kubeadm  
**Initial Kubernetes Version:** v1.35.10  
**Final Verified Kubernetes Version:** v1.36.6  
**CNI:** Calico v3.33.0  
**Cluster Architecture:** 1 Control Plane + 2 Worker Nodes

---

# 1. Project Objective

The objective was to deploy a Kubernetes cluster manually on AWS EC2 instances using `kubeadm`, configure containerd as the container runtime, deploy Calico as the Container Network Interface (CNI), join worker nodes, and perform a sequential Kubernetes minor-version upgrade.

The implementation was divided into these activities:

1. AWS infrastructure preparation.
2. EC2 instance provisioning.
3. Security group and network configuration.
4. Ubuntu operating system preparation.
5. Containerd installation and configuration.
6. Kubernetes package installation.
7. Control-plane initialization using kubeadm.
8. Calico CNI installation.
9. Worker node joining.
10. Cluster health verification.
11. Kubernetes v1.35.10 to v1.36.6 upgrade.
12. Final cluster verification.

# 2. Architecture

## 2.1 Cluster architecture

The cluster consists of three EC2 instances.

| Component | Hostname | Private IP | Public IP | Instance type |
|---|---|---|---|---|
| Control Plane | k8s-master | 10.0.4.222 | 16.176.169.12 | m7i-flex.large |
| Worker 1 | k8s-worker-1 | 10.0.8.253 | 54.79.3.106 | c7i-flex.large |
| Worker 2 | k8s-worker-2 | 10.0.15.115 | 32.236.164.142 | c7i-flex.large |

**Instance specifications:**

- Control Plane: 2 vCPU, 8 GiB RAM.
- Worker 1: 2 vCPU, 4 GiB RAM.
- Worker 2: 2 vCPU, 4 GiB RAM.

All three instances were deployed in the same Availability Zone.

## 2.2 Software architecture

| Component | Version / Configuration |
|---|---|
| OS | Ubuntu 26.04.1 LTS |
| Kubernetes | v1.35.10 initially |
| kubeadm | v1.35.10 initially |
| kubelet | v1.35.10 initially |
| kubectl | v1.35.10 initially |
| Container runtime | containerd 2.2.2 |
| CNI | Calico v3.33.0 |
| Pod CIDR | 192.168.0.0/16 |
| Container runtime socket | /run/containerd/containerd.sock |
| Upgrade target | v1.36.6 |

## 2.3 Network architecture

The cluster uses private IP addresses for communication between Kubernetes nodes.

Public IP addresses are used for SSH access.

The Kubernetes API server is not exposed publicly.

```text
                     Internet
                         |
                    Internet Gateway
                         |
                  Public Subnet
                    10.0.0.0/20
                         |
          +--------------+--------------+
          |              |              |
          |              |              |
      k8s-master     k8s-worker-1   k8s-worker-2
      10.0.4.222     10.0.8.253     10.0.15.115
          |              |              |
          +--------------+--------------+
                 Private communication
                 Kubernetes Cluster
```

---

# 3. AWS Infrastructure Preparation

## 3.1 VPC configuration

Created a dedicated VPC for the Kubernetes cluster.

| Parameter | Value |
|---|---|
| VPC ID | vpc-0e092342e8c2a2bc4 |
| Subnet ID | subnet-084d5946ce03b14a1 |
| VPC CIDR | 10.0.0.0/20 |
| Availability Zone | ap-southeast-2a |
| AZ ID | apse2-az1 |
| Internet Gateway | igw-098211ba4b2fd04e1 |
| Route Table | rtb-09553d9a843278065 |

### Internet Gateway and route table

An Internet Gateway was attached to the VPC.

The route table contained:

| Destination | Target |
|---|---|
| 10.0.0.0/20 | local |
| 0.0.0.0/0 | Internet Gateway |

The public subnet was associated with the route table.

Auto-assign public IPv4 was enabled for the subnet.

VPC DNS resolution and DNS hostnames were enabled.

**Important:** Although the instances were in a public subnet, Kubernetes communication was configured around private IP addresses.

A NAT Gateway was not created.

## 3.2 Security group configuration

Two security groups were used:

- k8s-master-sg
- Worker security group

Master security group ID:

`sg-0991197c19de361ca`

### Inbound rules

| Rule | Source | Purpose |
|---|---|---|
| SSH TCP 22 | My IP | Administrative SSH access |
| All traffic | Master SG and Worker SG references | Internal cluster communication |

The master and worker security groups were configured to allow communication between the cluster nodes through security group references.

This also covered Calico VXLAN traffic between nodes.

### Outbound rules

Default outbound access was retained:

| Type | Destination |
|---|---|
| All traffic | 0.0.0.0/0 |

### Security considerations

- Kubernetes API port 6443 was not opened to the public internet.
- SSH access was restricted to the administrator's IP.
- Cluster communication used private IP addresses.
- No public Kubernetes API endpoint was configured.

---

# 4. EC2 Instance Preparation

Three Ubuntu 26.04.1 LTS instances were launched.

The hostnames were configured as:

```bash
sudo hostnamectl set-hostname k8s-master
```

On worker 1:

```bash
sudo hostnamectl set-hostname k8s-worker-1
```

On worker 2:

```bash
sudo hostnamectl set-hostname k8s-worker-2
```

## 4.1 Configure host resolution

The following entries were added to `/etc/hosts` on all three instances:

```text
10.0.4.222   k8s-master
10.0.8.253   k8s-worker-1
10.0.15.115  k8s-worker-2
```

Verify:

```bash
getent hosts k8s-master k8s-worker-1 k8s-worker-2
```

Expected result: all three hostnames resolve to their respective private IP addresses.

## 4.2 Verify OS and networking

```bash
hostnamectl
```

```bash
ip addr
```

```bash
ip route
```

```bash
getent hosts k8s-master k8s-worker-1 k8s-worker-2
```

The same hostname resolution was verified on every node.

---

# 5. Kubernetes Node Prerequisites

The following configuration was applied to all three nodes.

## 5.1 Load required kernel modules

Create the configuration file:

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Load the modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify:

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

**Purpose:**

- `overlay`: Supports overlay filesystems used by container storage.
- `br_netfilter`: Allows bridged network traffic to be processed by netfilter.

## 5.2 Configure kernel networking parameters

Create:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

All three values were verified as `1`.

**Purpose:**

- Bridge traffic can be processed by iptables.
- IPv6 bridge traffic can be processed by ip6tables.
- IPv4 forwarding allows packets to move between network interfaces.

These settings are important for Kubernetes pod networking.

## 5.3 Disable swap

Kubernetes kubelet requires swap to be disabled under the default configuration.

Run:

```bash
sudo swapoff -a
```

Verify:

```bash
swapon --show
```

Expected output: empty.

The swap configuration was also checked to ensure swap would not be re-enabled during reboot.

---

# 6. Container Runtime Installation and Configuration

containerd was selected as the container runtime.

Docker Engine was not used as the Kubernetes runtime because Docker Engine does not directly implement the Container Runtime Interface (CRI). Using Docker Engine would require an additional adapter such as cri-dockerd.

## 6.1 Install containerd

Install containerd through the configured Ubuntu package repositories.

Verify:

```bash
containerd --version
```

Installed version:

```text
containerd 2.2.2
```

Check service status:

```bash
sudo systemctl status containerd --no-pager
```

Enable and start the service:

```bash
sudo systemctl enable --now containerd
```

## 6.2 Generate the containerd configuration

Generate the default configuration:

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
```

## 6.3 Configure systemd cgroup driver

Edit:

```bash
sudo nano /etc/containerd/config.toml
```

Locate the runc runtime options section:

```toml
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
```

Set:

```toml
SystemdCgroup = true
```

**Why systemd cgroups?**

Kubernetes and containerd should use compatible cgroup management.

With systemd as the cgroup driver, systemd manages the resource control groups for containers and pods.

## 6.4 Restart containerd

```bash
sudo systemctl restart containerd
```

Verify:

```bash
sudo systemctl status containerd --no-pager
```

Check the CRI plugin:

```bash
sudo ctr plugins ls | grep cri
```

The CRI plugin was verified as healthy.

This configuration was completed on all three nodes.

---

# 7. Install Kubernetes Components

Kubernetes packages were installed from the official `pkgs.k8s.io` repository.

A separate package repository exists for each Kubernetes minor version.

The initial repository was v1.35.

## 7.1 Configure the Kubernetes APT repository

The repository configuration used:

```text
https://pkgs.k8s.io/core:/stable:/v1.35/deb/
```

The repository file was:

```text
/etc/apt/sources.list.d/kubernetes.list
```

The signing key was configured under:

```text
/etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Update the package index:

```bash
sudo apt update
```

## 7.2 Install Kubernetes packages

The following packages were installed on all three nodes:

- kubeadm
- kubelet
- kubectl

Initial version:

```text
v1.35.10
```

Verify:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

## 7.3 Hold Kubernetes packages

Kubernetes packages were held to prevent unintended upgrades through normal system package updates.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify:

```bash
apt-mark showhold
```

Expected:

```text
kubeadm
kubectl
kubelet
```

This was performed on all three nodes.

---

# 8. Initialize the Kubernetes Control Plane

The control-plane node was initialized using kubeadm.

Node:

```text
Hostname: k8s-master
Private IP: 10.0.4.222
```

## 8.1 Run kubeadm init

Execute on the master:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.0.4.222 \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=v1.35.10 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

### Explanation of the parameters

| Parameter | Purpose |
|---|---|
| `--apiserver-advertise-address` | Specifies the private IP used by the API server |
| `--pod-network-cidr` | Defines the pod network range |
| `--kubernetes-version` | Specifies the initial Kubernetes version |
| `--cri-socket` | Specifies the containerd runtime socket |

The Pod CIDR was selected as:

```text
192.168.0.0/16
```

This range was also used in the Calico configuration.

## 8.2 Configure kubectl access

After successful initialization, configure the Kubernetes administrator kubeconfig for the Ubuntu user:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

At this stage, the control-plane node may show `NotReady` because the CNI has not yet been installed.

This is expected.

---

# 9. Install and Configure Calico CNI

Calico v3.33.0 was selected as the cluster networking solution.

Calico provides pod networking and network policy functionality.

The cluster was configured to use VXLAN encapsulation with BGP disabled.

## 9.1 Deploy Tigera Operator

The Tigera Operator was deployed using the Calico v3.33.0 installation manifests.

The operator manages Calico components and their configuration.

Check:

```bash
kubectl get pods -n tigera-operator
```

## 9.2 Troubleshoot the Tigera Operator issue

During the initial installation, the Tigera Operator experienced CrashLoopBackOff.

The issue was related to the Kubernetes v1.35 admission API and the MutatingAdmissionPolicy feature.

The API server was configured with these additional arguments in:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
- --feature-gates=MutatingAdmissionPolicy=true
- --runtime-config=admissionregistration.k8s.io/v1beta1=true
```

The API server static pod was automatically recreated by kubelet after the manifest change.

Verification:

```bash
kubectl get pods -n kube-system
```

The API group was also checked:

```bash
kubectl get --raw /apis/admissionregistration.k8s.io
```

The required API version became available.

**Important operational lesson:**

Do not keep backup copies of static pod manifests inside:

```text
/etc/kubernetes/manifests/
```

Kubelet processes manifest files from this directory. An additional backup YAML file can be interpreted as another static pod definition.

The backup was moved outside the manifest directory:

```text
/etc/kubernetes/backup/kube-apiserver.yaml.backup
```

## 9.3 Configure Calico Installation

The Calico Installation custom resource was configured as follows:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - blockSize: 26
        cidr: 192.168.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
```

Apply the configuration:

```bash
kubectl apply -f custom-resources.yaml
```

### Explanation

| Configuration | Purpose |
|---|---|
| `bgp: Disabled` | Disables BGP routing |
| `cidr: 192.168.0.0/16` | Defines the pod network |
| `blockSize: 26` | Allocates pod IP blocks |
| `encapsulation: VXLAN` | Uses VXLAN overlay networking |
| `natOutgoing: Enabled` | Enables outgoing NAT |
| `nodeSelector: all()` | Applies the pool to all nodes |

## 9.4 Verify Calico

```bash
kubectl get pods -n calico-system
```

```bash
kubectl get pods -n tigera-operator
```

```bash
kubectl get nodes
```

Calico components became healthy and the master node transitioned to `Ready`.

---

# 10. Join Worker Nodes

After the control plane and CNI were ready, both worker nodes were joined to the cluster.

## 10.1 Generate the join command

Run on the master:

```bash
kubeadm token create --print-join-command
```

The command provides the API server endpoint, token, and discovery CA certificate hash.

The generated token is temporary and should not be stored in public documentation.

## 10.2 Join worker 1

SSH into worker 1 and execute the generated join command with the containerd socket specified:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## 10.3 Join worker 2

Repeat the same procedure on worker 2.

Each worker was joined using the same cluster join credentials.

## 10.4 Verify the nodes

On the master:

```bash
kubectl get nodes -o wide
```

Expected architecture:

| Node | Role | Initial Version |
|---|---|---|
| k8s-master | control-plane | v1.35.10 |
| k8s-worker-1 | worker | v1.35.10 |
| k8s-worker-2 | worker | v1.35.10 |

All three nodes eventually reached `Ready`.

---

# 11. Understand the Kubernetes Components

After joining the workers, the cluster contained several system pods.

Check:

```bash
kubectl get pods -A -o wide
```

The main components were:

| Component | Responsibility |
|---|---|
| kube-apiserver | Exposes the Kubernetes API |
| etcd | Stores cluster state |
| kube-scheduler | Assigns pods to nodes |
| kube-controller-manager | Runs Kubernetes controllers |
| kube-proxy | Implements Service networking |
| CoreDNS | Provides cluster DNS |
| calico-node | Runs Calico networking on nodes |
| calico-kube-controllers | Manages Calico-related cluster resources |
| calico-typha | Distributes Calico datastore updates |
| tigera-operator | Manages Calico installation |
| csi-node-driver | Provides node-level storage integration |

Some components are deployed as static pods, some as Deployments, and others as DaemonSets.

Useful commands:

```bash
kubectl get pods -n kube-system
```

```bash
kubectl get pods -n calico-system
```

```bash
kubectl get pods -n tigera-operator
```

```bash
kubectl get daemonsets -A
```

```bash
kubectl get deployments -A
```

```bash
kubectl get services -A
```

---

# 12. Kubernetes Upgrade: v1.35.10 to v1.36.6

## 12.1 Upgrade objective

The cluster was upgraded sequentially from Kubernetes v1.35.10 to v1.36.6.

The upgrade was performed in this order:

1. Control-plane kubeadm upgrade.
2. Control-plane Kubernetes component upgrade.
3. Control-plane kubelet and kubectl upgrade.
4. Worker 1 upgrade.
5. Worker 2 upgrade.
6. Final cluster verification.

The upgrade was performed one node at a time.

## 12.2 Important upgrade concepts

| Command / Concept | Purpose |
|---|---|
| `kubeadm upgrade plan` | Displays available cluster upgrade paths |
| `kubeadm upgrade apply` | Upgrades the first control-plane node |
| `kubeadm upgrade node` | Updates the local node configuration |
| `kubectl drain` | Marks a node unschedulable and evicts eligible workloads |
| `kubectl uncordon` | Makes the node schedulable again |
| `apt-mark unhold` | Allows a held package to be upgraded |
| `apt-mark hold` | Prevents automatic package upgrades |

**Important:** The Kubernetes version shown by `kubectl get nodes` is the kubelet version reported by each node. Upgrading kubeadm and the control-plane components does not automatically upgrade the kubelet on every node.

## 12.3 Upgrade the master repository

On the master, change the repository from v1.35 to v1.36:

```bash
sudo sed -i 's/v1.35/v1.36/g' \
  /etc/apt/sources.list.d/kubernetes.list
```

Update:

```bash
sudo apt update
```

Check available versions:

```bash
apt-cache madison kubeadm
```

The selected package version was:

```text
1.36.6-1.1
```

## 12.4 Upgrade master kubeadm

```bash
sudo apt-mark unhold kubeadm

sudo apt-get update

sudo apt-get install -y kubeadm=1.36.6-1.1

sudo apt-mark hold kubeadm
```

Verify:

```bash
kubeadm version
```

Expected:

```text
GitVersion: v1.36.6
```

## 12.5 Check the upgrade plan

```bash
sudo kubeadm upgrade plan
```

The plan identified:

- Current cluster version: v1.35.10.
- Target version: v1.36.6.
- Control-plane components to be upgraded.
- kube-proxy upgrade.
- CoreDNS upgrade.
- etcd upgrade.
- Kubelet configuration and version requirements.

The plan indicated that the cluster could be upgraded to v1.36.6.

## 12.6 Upgrade the control plane

Run on the master:

```bash
sudo kubeadm upgrade apply v1.36.6
```

The command completed successfully.

The control-plane components were upgraded, including:

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd
- kube-proxy
- CoreDNS

The command reported:

```text
[upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.36.6".
```

At this stage, `kubectl get nodes` still showed v1.35.10 because the kubelet package had not yet been upgraded.

This was expected.

## 12.7 Drain the master

Before upgrading the master kubelet, drain the node:

```bash
kubectl drain k8s-master --ignore-daemonsets
```

The master was cordoned and eligible workloads were evicted.

DaemonSet-managed pods were intentionally ignored.

The command completed with:

```text
node/k8s-master drained
```

## 12.8 Upgrade master kubelet and kubectl

Check package versions:

```bash
apt-cache madison kubelet kubectl
```

Both packages had version `1.36.6-1.1` available.

Install:

```bash
sudo apt-mark unhold kubelet kubectl

sudo apt-get update

sudo apt-get install -y kubelet=1.36.6-1.1 kubectl=1.36.6-1.1

sudo apt-mark hold kubelet kubectl
```

Restart:

```bash
sudo systemctl daemon-reload

sudo systemctl restart kubelet
```

Verify:

```bash
kubelet --version
kubectl version --client
systemctl status kubelet --no-pager
```

Both client and kubelet versions showed v1.36.6, and the kubelet service was active.

## 12.9 Uncordon the master

```bash
kubectl uncordon k8s-master
```

Verify:

```bash
kubectl get nodes
```

The master returned to `Ready` on v1.36.6.

---

# 13. Upgrade Worker 1

Worker 1 was upgraded after the control plane.

## 13.1 Drain worker 1

Run this command from the master, where the administrative kubeconfig is configured:

```bash
kubectl drain k8s-worker-1 --ignore-daemonsets
```

The worker was cordoned and eligible workloads were evicted.

The command completed with:

```text
node/k8s-worker-1 drained
```

**Operational note:** Running this command directly on a worker without a configured kubeconfig resulted in a connection attempt to `localhost:8080`. The command was subsequently run successfully from the master.

## 13.2 Update the worker repository

SSH into worker 1:

```bash
sudo sed -i 's/v1.35/v1.36/g' \
  /etc/apt/sources.list.d/kubernetes.list

sudo apt update
```

Check versions:

```bash
apt-cache madison kubeadm kubelet kubectl
```

## 13.3 Upgrade worker kubeadm

```bash
sudo apt-mark unhold kubeadm

sudo apt-get install -y kubeadm=1.36.6-1.1

sudo apt-mark hold kubeadm
```

Verify:

```bash
kubeadm version
```

Expected: v1.36.6.

## 13.4 Update worker configuration

```bash
sudo kubeadm upgrade node
```

The command:

- Retrieved cluster configuration from the kubeadm-config ConfigMap.
- Ran preflight checks.
- Skipped control-plane upgrade phases because this was a worker.
- Updated the local kubelet configuration.
- Backed up the existing kubelet configuration.
- Wrote the updated configuration to `/var/lib/kubelet/config.yaml`.

The command completed successfully.

A warning about the recommended `bindAddress` value in KubeProxyConfiguration was displayed. It did not prevent the node configuration upgrade.

## 13.5 Upgrade worker kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl

sudo apt-get install -y kubelet=1.36.6-1.1 kubectl=1.36.6-1.1

sudo apt-mark hold kubelet kubectl
```

Restart:

```bash
sudo systemctl daemon-reload

sudo systemctl restart kubelet
```

Verify:

```bash
kubelet --version
kubectl version --client
systemctl status kubelet --no-pager
```

Both packages reported v1.36.6.

## 13.6 Uncordon worker 1

Return to the master:

```bash
kubectl uncordon k8s-worker-1
```

Verify:

```bash
kubectl get nodes
```

Worker 1 returned to `Ready` with version v1.36.6.

---

# 14. Upgrade Worker 2

The same procedure was performed on worker 2.

## 14.1 Drain worker 2

On the master:

```bash
kubectl drain k8s-worker-2 --ignore-daemonsets
```

The command successfully cordoned and drained worker 2.

## 14.2 Update the repository

On worker 2:

```bash
sudo sed -i 's/v1.35/v1.36/g' \
  /etc/apt/sources.list.d/kubernetes.list

sudo apt update
```

## 14.3 Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm

sudo apt-get install -y kubeadm=1.36.6-1.1

sudo apt-mark hold kubeadm
```

Verify:

```bash
kubeadm version
```

Expected: v1.36.6.

## 14.4 Update node configuration

```bash
sudo kubeadm upgrade node
```

The worker configuration was updated successfully.

The command skipped control-plane and addon phases because this was a worker node.

## 14.5 Upgrade kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl

sudo apt-get install -y kubelet=1.36.6-1.1 kubectl=1.36.6-1.1

sudo apt-mark hold kubelet kubectl
```

Restart:

```bash
sudo systemctl daemon-reload

sudo systemctl restart kubelet
```

Verify:

```bash
kubelet --version
kubectl version --client
systemctl status kubelet --no-pager
```

Both packages reported v1.36.6.

## 14.6 Uncordon worker 2

On the master:

```bash
kubectl uncordon k8s-worker-2
```

The command returned:

```text
node/k8s-worker-2 uncordoned
```

---

# 15. Final Verification

After upgrading all three nodes, the cluster was checked from the master.

Command:

```bash
kubectl get nodes -o wide
```

Final observed output:

```text
NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP
k8s-master     Ready    control-plane   3h     v1.36.6   10.0.4.222
k8s-worker-1   Ready    <none>          133m   v1.36.6   10.0.8.253
k8s-worker-2   Ready    <none>          129m   v1.36.6   10.0.15.115
```

All nodes were:

- `Ready`
- Running Kubernetes v1.36.6
- Using Ubuntu 26.04.1 LTS
- Using containerd 2.2.2

The external IP column showed `<none>` in Kubernetes node information. This was not a failure: the EC2 public IP addresses were used for SSH access, while Kubernetes node communication used private addresses.

## Additional verification commands

Check cluster information:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Check all pods:

```bash
kubectl get pods -A -o wide
```

Check system workloads:

```bash
kubectl get pods -n kube-system
```

Check Calico:

```bash
kubectl get pods -n calico-system
```

Check Tigera Operator:

```bash
kubectl get pods -n tigera-operator
```

Check node scheduling status:

```bash
kubectl get nodes
```

Check held packages on each node:

```bash
apt-mark showhold
```

---

# 16. Important Troubleshooting Notes and Lessons Learned

## 16.1 Node shows NotReady after kubeadm init

**Cause:** The cluster network plugin had not yet been installed.

**Resolution:** Install and configure Calico. Once the CNI components became healthy, the node transitioned to Ready.

## 16.2 Tigera Operator CrashLoopBackOff

**Cause:** The operator required an admission API and feature-gate configuration that was not enabled in the initial API server configuration.

**Resolution:** Added the following API server flags:

```yaml
- --feature-gates=MutatingAdmissionPolicy=true
- --runtime-config=admissionregistration.k8s.io/v1beta1=true
```

Verified that the API group was available and the operator started successfully.

## 16.3 Duplicate static pod manifest

**Cause:** A backup YAML file was temporarily placed inside `/etc/kubernetes/manifests/`.

Kubelet processes YAML manifest files in that directory.

**Resolution:** Moved the backup outside the static pod manifest directory.

## 16.4 kubectl attempts localhost:8080

**Cause:** The worker node did not have an administrative kubeconfig configured.

**Resolution:** Ran cluster administration commands from the master, which had the admin kubeconfig.

## 16.5 Node version remains old after kubeadm upgrade apply

**Cause:** The kubeadm control-plane upgrade does not automatically update the kubelet package on every node.

**Resolution:** Upgrade kubelet and kubectl individually on each node, restart kubelet, and verify the node version.

## 16.6 Node remains unschedulable after upgrade

**Cause:** `kubectl drain` cordons the node, preventing new pods from being scheduled.

**Resolution:**

```bash
kubectl uncordon <node-name>
```

Always verify the node status after uncordoning.

---

# 17. Important Operational Commands Reference

| Task | Command |
|---|---|
| List nodes | `kubectl get nodes -o wide` |
| List all pods | `kubectl get pods -A -o wide` |
| Check system pods | `kubectl get pods -n kube-system` |
| Drain a node | `kubectl drain <node> --ignore-daemonsets` |
| Uncordon a node | `kubectl uncordon <node>` |
| Check cluster information | `kubectl cluster-info` |
| Generate join command | `kubeadm token create --print-join-command` |
| Check kubeadm version | `kubeadm version` |
| Check kubelet version | `kubelet --version` |
| Check kubectl version | `kubectl version --client` |
| Check kubelet service | `systemctl status kubelet --no-pager` |
| Restart kubelet | `sudo systemctl restart kubelet` |
| Check containerd | `sudo systemctl status containerd` |
| Check held packages | `apt-mark showhold` |
| Check available package versions | `apt-cache madison kubeadm kubelet kubectl` |
| Check Calico pods | `kubectl get pods -n calico-system` |

---

# 18. Current Project Status

| Activity | Status |
|---|---|
| AWS VPC and subnet configuration | Completed |
| Internet Gateway and route table | Completed |
| Security groups | Completed |
| Three EC2 instances | Completed |
| Hostname and private DNS resolution | Completed |
| Kernel modules and sysctl configuration | Completed |
| Swap disabled | Completed |
| containerd installation | Completed |
| containerd systemd cgroup configuration | Completed |
| Kubernetes v1.35.10 installation | Completed |
| kubeadm control-plane initialization | Completed |
| Calico v3.33.0 installation | Completed |
| Worker 1 joined | Completed |
| Worker 2 joined | Completed |
| Initial cluster health verification | Completed |
| Master upgrade to v1.36.6 | Completed |
| Worker 1 upgrade to v1.36.6 | Completed |
| Worker 2 upgrade to v1.36.6 | Completed |
| Final node health verification | Completed |

**Final cluster version: Kubernetes v1.36.6**

**Final node state: 3/3 Ready**

---

# 19. Future Activity: Kubernetes v1.37 Upgrade

The next planned activity is to upgrade the cluster from v1.36.6 to the latest available v1.37 patch release.

The general sequence will remain:

1. Verify release notes and compatibility.
2. Verify cgroup v2 compatibility on all nodes.
3. Change the Kubernetes package repository to v1.37 on all nodes.
4. Upgrade kubeadm on the control plane.
5. Run `kubeadm upgrade plan`.
6. Run `kubeadm upgrade apply`.
7. Drain and upgrade the control-plane kubelet and kubectl.
8. Uncordon the master.
9. Upgrade worker 1.
10. Upgrade worker 2.
11. Verify all nodes are Ready.

Do not assume the v1.36.6 package version will be used for v1.37. Check the available v1.37 patch version before installing.

---

# 20. References

- Kubernetes kubeadm installation: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Kubernetes cluster upgrade: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- Kubernetes Linux worker node upgrade: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/
- Kubernetes cluster upgrade overview: https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/
- Calico documentation: https://docs.tigera.io/calico/latest/

**Documentation note:** This README records the implementation and observed results of the lab. AWS resource identifiers, private IPs, software versions, and successful upgrade outputs reflect the actual environment. Temporary join tokens and other credentials have intentionally been excluded.