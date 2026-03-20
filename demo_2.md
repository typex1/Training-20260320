[← Back to Demos](index.html)

# Demo 2: ClusterIP Service

**Objective:** Deploy a ClusterIP service — the default Kubernetes service type. It exposes the service on an internal cluster IP, making it reachable *only from within the cluster*.

## How ClusterIP Works

**① Client Sends Request**
A pod inside the cluster sends a request to the service's **virtual IP** (e.g. 10.100.45.12). This IP exists only within the cluster — it's not routable externally.

**② kube-proxy Intercepts**
**kube-proxy** on each node programs iptables/IPVS rules that intercept traffic to the ClusterIP and load-balance it across the service's endpoints (pod IPs).

**③ Traffic Reaches Pods**
The request is forwarded to one of the **backend pods**. kube-proxy distributes traffic randomly or round-robin depending on the mode (iptables vs IPVS).

**④ External Access Blocked**
External clients **cannot reach** a ClusterIP service. The virtual IP only exists in iptables rules on cluster nodes — it's invisible outside the cluster.

## Step 1: Create namespace and deployment

```bash
cat <<EOF > clusterip-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: svc-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: svc-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

## Step 2: Create the ClusterIP service

```bash
cat <<EOF >> clusterip-demo.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: svc-demo
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF
```

## Step 3: Verify the service and endpoints

```
kubectl apply -f clusterip-demo.yaml
kubectl get svc -n svc-demo
kubectl get endpoints -n svc-demo
```

## Step 4: Test from inside the cluster

```bash
# Launch a test pod and curl the service by DNS name
kubectl run test-client --rm -it --image=busybox -n svc-demo -- sh

# Inside the pod:
wget -qO- http://backend.svc-demo.svc.cluster.local
wget -qO- http://backend  # short name works within same namespace
```

### 🔎 What to Observe

* The service gets a **cluster-internal IP** (e.g., 10.100.x.x) that is NOT routable outside the cluster.
* The `endpoints` list shows the pod IPs backing the service — kube-proxy sets up iptables/IPVS rules to load-balance across them.
* DNS resolution works: `backend.svc-demo.svc.cluster.local` resolves to the ClusterIP.
* Requests from outside the cluster to this IP will **not work**.

## Cleanup

```
kubectl delete namespace svc-demo
```
