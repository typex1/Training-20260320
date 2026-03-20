[← Back to Demos](index.html)

# Demo 1: Pod-to-Pod Communication

**Objective:** Understand how pods communicate on the same node (via veth pairs and a Linux bridge) and across nodes (via the Amazon VPC CNI plugin which assigns VPC IP addresses directly to pods).

## How Pod-to-Pod Works

**① veth Pairs**
Each pod gets a virtual ethernet (veth) pair. One end is the pod's **eth0** interface, the other connects to the node's Linux bridge (`cbr0`).

**② Same-Node Traffic**
Pods on the same node communicate through the **Linux bridge**. Packets go from Pod A's veth → bridge → Pod B's veth. No encapsulation, pure L2 switching.

**③ Cross-Node Traffic**
The **VPC CNI** assigns real VPC IPs to pods. Cross-node traffic routes directly through the VPC — no overlay, no encapsulation. The VPC route table knows which ENI owns each pod IP.

**④ VPC CNI Advantage**
Pods are first-class VPC citizens. They can communicate with any VPC resource (RDS, ElastiCache) directly using their VPC IP — no NAT or proxying needed.

## Step 1: Create a namespace

```
kubectl create namespace netdemo
```

## Step 2: Deploy two pods

```bash
cat <<EOF > pod-to-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: netdemo
  labels:
    app: nettest
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: netdemo
  labels:
    app: nettest
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
EOF
```

Apply with: `kubectl apply -f pod-to-pod.yaml`

## Step 3: Check pod IPs and node placement

```
kubectl get pods -n netdemo -o wide
```

Note the **IP** and **NODE** columns. If both pods land on the same node, communication uses veth pairs. If on different nodes, the VPC CNI routes traffic across the VPC.

## Step 4: Test connectivity

```bash
# Ping pod-b from pod-a (replace POD_B_IP)
kubectl exec -n netdemo pod-a -- ping -c 3 <POD_B_IP>

# Inspect network interfaces inside a pod
kubectl exec -n netdemo pod-a -- ip addr
kubectl exec -n netdemo pod-a -- ip route
```

## Step 5: Inspect veth pairs on the node (optional, requires node access)

```bash
# SSH to the node, then:
ip link show type veth
bridge link show
```

### 🔎 What to Observe

* Each pod gets a unique VPC IP address (not a cluster-internal overlay IP) — this is the VPC CNI in action.
* Pods on the **same node** communicate through veth pairs connected to a Linux bridge.
* Pods on **different nodes** communicate directly via VPC routing — no encapsulation needed.
* The `ip route` output inside the pod shows the default gateway pointing to the node's bridge interface.

## Cleanup

```
kubectl delete namespace netdemo
```
