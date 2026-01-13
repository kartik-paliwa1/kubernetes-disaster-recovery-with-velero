# kubernetes-disaster-recovery-with-velero

<img width="1536" height="1024" alt="arch" src="https://github.com/user-attachments/assets/cb04897e-b982-416a-a06d-771fa824b90a" />


This document explains **step-by-step** how to set up **two Kubernetes clusters (Cluster-A and Cluster-B)** on AWS EC2 and perform **Disaster Recovery (DR)** using **Velero**.

---

## Step 1: Create VMs for Cluster A and Cluster B

### Cluster A (Primary Cluster)

* Created **3 VMs**

  * 1 Control Plane node
  * 2 Worker nodes

### Cluster B (Disaster Recovery Cluster)

* Created **2 VMs**

  * 1 Control Plane node
  * 1 Worker node

---

## Step 2: Setup of Cluster A

### 1. SSH into Cluster A Control Plane VM

```bash
ssh -i <key>.pem ubuntu@<cluster_a_control_plane_ip>
```

### 2. Update and Upgrade the VM

```bash
sudo apt update
sudo apt upgrade -y
```

### 3. Install Kubernetes Required Packages

(Using official Kubernetes documentation)

* Install `kubeadm`, `kubelet`, `kubectl`

### 4. Install Container Runtime (containerd)

```bash
sudo apt install containerd -y
```

### 5. Verify Kubernetes Tools

Now the system has:

* kubelet
* kubeadm
* kubectl

### 6. Initialize Kubernetes Cluster (Cluster A)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### 7. Save the `kubeadm join` Command

This command will be used later to join worker nodes.

### 8. Install Pod Network (CNI)

```bash
kubectl apply -f <pod-network-yaml>
```

### 9. SSH into Worker Node of Cluster A

```bash
ssh -i <key>.pem ubuntu@<cluster_a_worker_ip>
```

### 10. Repeat Kubernetes Setup on Worker Nodes

* Update & upgrade
* Install containerd
* Install kubeadm, kubelet, kubectl

### 11. Join Worker Node to Cluster A

```bash
sudo kubeadm join <control_plane_ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 12. Verify Nodes from Control Plane

```bash
kubectl get nodes
```

---

## Step 3: Create Cluster B

### 1. Setup Control Plane and Worker Node for Cluster B

* Install Kubernetes packages
* Install containerd
* Install pod network (Calico)
* Follow same process as Cluster A

### 2. Verify Worker Node Connection

```bash
kubectl get nodes
```

---

## Step 4: Initialization of Disaster Recovery (DR) Using Velero

### 1. DR Plan

* **Cluster A** → Primary Cluster
* **Cluster B** → Recovery Cluster

Cluster A must access Cluster B.

---

### 2. SSH into Cluster A

```bash
ssh -i <key>.pem ubuntu@<cluster_a_ip>
```

---

### 3. Backup Existing kubeconfig

```bash
mv ~/.kube ~/.kube.backup
mkdir -p ~/.kube
```

---

### 4. Prepare Cluster A kubeconfig

```bash
sudo cp /etc/kubernetes/admin.conf /home/ubuntu/cluster-a.conf
sudo chown ubuntu:ubuntu /home/ubuntu/cluster-a.conf
chmod 600 /home/ubuntu/cluster-a.conf
```

---

### 5. Rename Context for Cluster A

```bash
kubectl --kubeconfig=/home/ubuntu/cluster-a.conf \
config rename-context kubernetes-admin@kubernetes cluster-a
```

Now context name is **cluster-a**.

---

### 6. SSH into Cluster B

```bash
ssh -i <key>.pem ubuntu@<cluster_b_ip>
```

---

### 7. Prepare Cluster B kubeconfig

```bash
sudo cp /etc/kubernetes/admin.conf /home/ubuntu/cluster-b.conf
sudo chown ubuntu:ubuntu /home/ubuntu/cluster-b.conf
chmod 600 /home/ubuntu/cluster-b.conf
```

---

### 8. Rename Context for Cluster B

```bash
kubectl --kubeconfig=/home/ubuntu/cluster-b.conf \
config rename-context kubernetes-admin@kubernetes cluster-b
```

---

### 9. Copy Cluster-B kubeconfig to Cluster-A

```bash
scp -i <key>.pem ubuntu@<cluster_b_master_private_ip>:/home/ubuntu/cluster-b.conf ~/
```

---

### 10. Merge kubeconfigs (Important Step)

```bash
KUBECONFIG=~/cluster-a.conf:~/cluster-b.conf \
kubectl config view --flatten > ~/.kube/config
```

Secure it:

```bash
chmod 600 ~/.kube/config
```

---

### 11. Verify Contexts

```bash
kubectl config get-contexts
```

Both **cluster-a** and **cluster-b** should appear.

---

### 12. Verify Cluster A Nodes

```bash
kubectl config use-context cluster-a
kubectl get nodes
```

Expected:

* 1 Control Plane
* 2 Worker Nodes

---

### 13. Verify Cluster B Nodes

```bash
kubectl config use-context cluster-b
kubectl get nodes
```

Expected:

* 1 Control Plane
* 1 Worker Node

---

## Step 5: Errors & Fixes

### Error 1: `illegal base64 data`

**Fix:**

```bash
rm -rf ~/.kube
mkdir ~/.kube
```

Then merge kubeconfigs again.

---

### Error 2: `connection to localhost:8080 was refused`

**Reason:**
KUBECONFIG not exported.

**Fix:**

```bash
export KUBECONFIG=~/.kube/config
```

---

### Error 3 (Most Important): Both Contexts Show Same Nodes

**Reason:**
Both kubeconfigs point to the same API server.

**Check:**

```bash
grep server ~/cluster-a.conf
grep server ~/cluster-b.conf
```

**Fix:**

* Edit both kubeconfig files manually using `nano`
* Ensure each file points to its correct API server
* Re-merge kubeconfigs again

---

## Step 6: Disaster Recovery with Velero

### 1. Install Velero on Cluster-A (Primary)

```bash
kubectl config use-context cluster-a
```

### 2. Cluster-B Will Be DR Cluster

---

### 3. Create S3 Bucket (AWS Console)

* Create S3 bucket for backups

---

### 4. Create IAM User for Velero

* Grant **S3 Full Access**
* Grant **EC2 Full Access**

---

### 5. Create Credentials File on Cluster-A

```bash
nano credentials-velero
```

Paste:

```text
aws_access_key_id=
aws_secret_access_key=
```

---

### 6. Install Velero CLI

(Using official documentation)

---

### 7. Install Velero on Cluster-A

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.1 \
  --bucket juju-velero-backup \
  --secret-file ./credentials-velero \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1 \
  --wait
```

---

### 8. Verify Velero Installation

```bash
kubectl get pods -n velero
kubectl get ns
```

Velero pod and namespace should exist.

---

## Step 7: Create First Backup

### 1. Switch to Cluster-A

```bash
kubectl config use-context cluster-a
kubectl get nodes
```

---

### 2. Create `prod` Namespace (If Not Exists)

---

### 3. Deploy Application

```bash
nano prod-nginx.yaml
kubectl apply -f prod-nginx.yaml
```

---

### 4. Verify Application

```bash
kubectl get all -n prod
```

---

### 5. Create Backup

```bash
velero backup create prod-backup \
--include-namespaces prod
```

---

### 6. Verify Backup

```bash
velero backup get
```

---

### 7. Check Backup Details

```bash
velero backup describe prod-backup-v2 --details
```

Should include:

* Deployment
* ReplicaSet
* Pods
* Service

---

## Step 8: Simulate Disaster

### 1. Delete Namespace

```bash
kubectl delete ns prod
```

---

### 2. Verify Deletion

```bash
kubectl get ns
```

---

### 3. Switch to Cluster-B

```bash
kubectl config use-context cluster-b
kubectl get nodes
```

---

### 4. Install Velero on Cluster-B

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.1 \
  --bucket juju-velero-backup \
  --secret-file ./credentials-velero \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1 \
  --wait
```

---

### 5. Verify Velero on Cluster-B

```bash
kubectl get pods -n velero
```

---

### 6. Verify Backup Visibility

```bash
velero backup get
```

---

### 7. Restore Backup

```bash
velero restore create prod-restore \
--from-backup prod-backup
```

---

### 8. Verify Restore

```bash
velero restore get
```

---

### 9. Verify Namespace Restored

```bash
kubectl get ns
```

---

### 10. Verify All Resources

```bash
kubectl get all -n prod
```

---

## ✅ Disaster Recovery Result

* Pods restored
* Deployments restored
* Services restored
* ReplicaSets restored

---

## Final Summary

> We took a namespace-level Velero backup of Cluster-A.
> After deleting the namespace to simulate a disaster, we restored the backup on Cluster-B.
> All Kubernetes objects, including Deployments, Pods, ReplicaSets, and Services, were successfully restored.
> The first restore recreated only the namespace because no workloads existed at that time.
> After deploying workloads and taking another backup, Velero successfully restored the application on the DR cluster.

---
