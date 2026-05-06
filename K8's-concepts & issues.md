## Kubernetes general issues

The 2 common real-time Kubernetes challenges 

### 1. Resource Sharing Problem (Cluster Resource Contention)

## Problem
- Multiple teams share the same Kubernetes cluster
- Each team deploys services in different namespaces
- Without limits, one namespace/pod can consume all CPU & RAM
- This leads to:
  - Pods crashing
  - OOMKilled errors
  - CrashLoopBackOff
  - Production outages

Example:
- Cluster: 100 CPU / 100GB RAM
- Payments namespace leaks memory
- Consumes 40GB RAM
- Other namespaces left with insufficient memory
- Their pods crash


# Solution Part 1: ResourceQuota (Namespace Level)

Use **ResourceQuota** to limit resources per namespace.

This ensures:
- One namespace cannot consume entire cluster
- Resource isolation between teams
- Reduced blast radius (Cluster → Namespace)

Example:

Payments Namespace:
- CPU: 15
- Memory: 15GB

Web Namespace:
- CPU: 20
- Memory: 20GB

Shipment Namespace:
- CPU: 25
- Memory: 25GB

Important Notes:
- Dev team provides requirements
- Based on performance benchmarking
- If quota exceeds cluster capacity → scale cluster


# Solution Part 2: Resource Limits (Pod Level)

Even with ResourceQuota, one pod can still consume all namespace resources.

Use **Resource Limits** to restrict per pod.

ResourceQuota → Namespace limit  
ResourceLimit → Pod limit  

Example:
Namespace limit = 15GB

Pods:
- pod1 = 3GB
- pod2 = 3GB
- pod3 = 3GB
- pod4 = 3GB
- pod5 = 3GB

Now if pod1 leaks memory:
- Only pod1 crashes
- Other pods remain safe

Blast radius reduced:
Cluster → Namespace → Pod

# Outcome

- Identify faulty pod easily
- Prevent cluster-wide failure
- Prevent namespace failure
- Improve production stability
- Better resource management

### 2. OOMKilled Pod Troubleshooting

## Problem
Pod crashes with:

- OOMKilled
- CrashLoopBackOff

Even after setting:
- ResourceQuota
- ResourceLimits

This means:
Application itself has memory leak

### Troubleshooting Steps

Step 1: Identify failing pod

```bash
kubectl get pods
```
Step 2: Check reason

```
kubectl describe pod <pod-name>
```
Look for: OOMKilled

Step 3: Login into pod

```bash
kubectl exec -it <pod-name> -- /bin/bash
```
Step 4: Take Thread Dump (Java apps)

```bash
kill -3 <pid>
```
Step 5: Take Heap Dump

```bash
jstack <pid>
```
Step 6: Share with Developers

**Developers will**:

Analyze memory leak
Fix code
Release new version
Then new version has to be redeployed

## How does the kubernetes scheduler quickly assign worker nodes for the pods? explain the internal working of k8's cluster?

Internal Working of Kubernetes Cluster  
The entire flow from the moment you run `kubectl apply` to the pod running on a worker node.

Components Involved

```bash
Master Node (Control Plane)
├── kube-apiserver       ← front door of the cluster
├── etcd                 ← database (stores everything)
├── kube-scheduler       ← decides which node for the pod
└── kube-controller-manager  ← watches and maintains desired state

Worker Node
├── kubelet              ← agent, runs pods
├── kube-proxy           ← networking
└── container runtime    ← actually runs containers (containerd)
```

The Full Flow — Step by Step
## Step 1 — You run kubectl apply -f pod.yaml
kubectl sends an HTTP POST request to the kube-apiserver on the master node.

```bash
kubectl  →  HTTPS request  →  kube-apiserver
```
## Step 2 — kube-apiserver validates and saves to etcd
kube-apiserver:

Authenticates the request (are you allowed?)
Validates the yaml (is it correct?)
Saves the pod object to etcd with status Pending and no node assigned yet

```bash
etcd now has:
{
  name: "my-pod",
  status: "Pending",
  nodeName: null      ← no node assigned yet
}
```

## Step 3 — kube-scheduler notices the unassigned pod
kube-scheduler is constantly watching etcd via kube-apiserver for pods that have nodeName: null.
It sees my-pod is pending with no node — kicks into action.

```bash
kube-scheduler:  "Hey! There's a pod with no node assigned, let me find the best node"
```

## Step 4 — Scheduler evaluates all nodes (two phases)
This is where the intelligence happens. Scheduler does this in 2 phases:
Phase 1 — Filtering (eliminate unfit nodes)

```bash
All Nodes in Cluster:
├── worker-node-1  →  not enough memory ❌
├── worker-node-2  →  has taint, pod can't tolerate ❌
├── worker-node-3  →  passes all checks ✅
└── worker-node-4  →  passes all checks ✅

Remaining candidates: worker-node-3, worker-node-4
```

## Filters check:

Does node have enough CPU and memory?  
Does node match nodeAffinity/nodeSelector?  
Can pod tolerate node's taints?  
Is node healthy and ready?  
Are there any port conflicts?  

## Phase 2 — Scoring (rank the fit nodes)

```bash
Remaining nodes get scored 0-100:

worker-node-3  →  score: 70  (has some pods already)
worker-node-4  →  score: 90  (mostly free, closer region) ✅ wins!
```

## Scoring considers:

Least number of existing pods (spread evenly)  
Available CPU/memory (more free = higher score)  
Data locality (is the image already pulled on that node?)  
Affinity preferences (weight scores added)  

## Step 5 — Scheduler assigns the pod to winning node
Scheduler updates the pod object in etcd via kube-apiserver:

```bash
etcd now has:
{
  name: "my-pod",
  status: "Pending",
  nodeName: "worker-node-4"   ← node assigned now!
}
```
Scheduler's job is done here. It does not actually start the pod — that's kubelet's job.

## Step 6 — kubelet on worker-node-4 notices the assignment
Every worker node has a kubelet that constantly watches etcd via kube-apiserver for pods assigned to its node.

```bash
kubelet on worker-node-4:
"Hey! A pod named my-pod is assigned to me, let me run it"
```

kubelet:

Pulls the container image (if not already cached)  
Tells the container runtime (containerd) to start the container  
Mounts volumes, sets up networking  
Starts the pod  

## Step 7 — kubelet updates pod status back to etcd
Once pod is running, kubelet reports back:

```bash
etcd now has:
{
  name: "my-pod",
  status: "Running",
  nodeName: "worker-node-4",
  podIP: "10.244.1.15"
}
```

## Step 8 — kube-controller-manager keeps watching
kube-controller-manager continuously watches etcd and ensures desired state = actual state.

```bash
Desired state (your yaml):  1 pod running
Actual state (etcd):        1 pod running  ✅  all good

worker-node-4 crashes
Actual state:               0 pods running ❌  mismatch!
        ↓
kube-controller-manager notices
        ↓
Creates a new pod object in etcd
        ↓
Scheduler picks a new node
        ↓
Pod runs on worker-node-3
```
```bash
You
 │
 │  kubectl apply -f pod.yaml
 ▼
kube-apiserver  ──→  validates  ──→  saves to etcd (Pending, nodeName: null)
                                          │
                                          │ scheduler watching...
                                          ▼
                                    kube-scheduler
                                    ├── Phase 1: Filter nodes
                                    ├── Phase 2: Score nodes
                                    └── assigns nodeName: worker-node-4
                                          │
                                          │ kubelet watching...
                                          ▼
                                    kubelet (worker-node-4)
                                    ├── pulls image
                                    ├── starts container
                                    └── reports status: Running
                                          │
                                          ▼
                                    etcd updated (Running, IP assigned)
                                          │
                                          │ controller-manager watching...
                                          ▼
                                    kube-controller-manager
                                    └── ensures desired state always matches
```
The beauty of Kubernetes is that no component directly calls another. Instead:

Every component watches etcd for changes  
When they see something relevant — they react  
This is called the reconciliation loop or watch pattern  

```bash
etcd changes  →  components notice  →  they act  →  etcd changes again  →  repeat
```
This is why Kubernetes is so resilient — each component independently does its job by just watching and reacting to state changes in etcd.

## You have 10 nodes with 500GB of disk attached to each node. Let us say the node disk space reached 85% usage; then the pod we have deployed should get evicted and re-deployed on a node with better disk health. Explain if this can even be done?

Kubernetes has a built-in feature called Node Pressure Eviction where the kubelet monitors node resources and evicts pods when thresholds are hit.  
When disk hits a threshold — kubelet automatically evicts pods.

What Happens After Eviction?

```bash
Node disk hits 85%
      ↓
kubelet evicts the pod  (NoExecute taint automatically added to node)
      ↓
kube-controller-manager notices pod is missing
      ↓
Creates a new pod object in etcd
      ↓
kube-scheduler picks a healthy node (filters out 85%+ disk nodes)
      ↓
Pod re-deploys on a node with better disk health
```

How the Kubernetes Secrets Store CSI Driver works, specifically in conjunction with AWS Secrets & Configuration Provider (ASCP), and how secrets are securely fetched and mounted into a pod at runtime?

Kubernetes itself doesn’t know how to fetch secrets from external systems like AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault.

That’s where these pieces come in:

  - Secrets Store CSI Driver → a Kubernetes CSI plugin that lets pods mount external secrets as files
  - ASCP (AWS Secrets & Configuration Provider) → AWS’s provider plugin for that driver
  - SecretProviderClass → Kubernetes custom resource that tells the driver what secret to fetch and how

Think of it like this:

Pod asks for a volume  
→ CSI driver intercepts  
→ CSI driver calls provider (ASCP)  
→ ASCP fetches secret from AWS Secrets Manager / SSM Parameter Store  
→ Secret gets mounted inside pod filesystem  

1. What is Secrets Store CSI Driver?

CSI = Container Storage Interface

Originally CSI was designed for storage volumes, but Kubernetes extended it so external secret systems can appear like mounted volumes.

The Secrets Store CSI Driver runs in your cluster (usually as a DaemonSet).

Its job:  

Receive mount request from pod  
Read SecretProviderClass  
Call external provider plugin  
Mount returned secrets into pod as files  

It does not store the secrets in etcd unless you explicitly sync them as Kubernetes Secrets.

Architecture
```bash
Application Pod
   |
   | volume mount request
   v
Secrets Store CSI Driver
   |
   | gRPC
   v
AWS Provider (ASCP)
   |
   | AWS SDK call
   v
AWS Secrets Manager / Parameter Store
```
2. What is ASCP?  

ASCP = AWS Secrets and Configuration Provider

It’s the AWS-specific provider for the Secrets Store CSI Driver.

It knows how to talk to:

AWS Secrets Manager
AWS Systems Manager Parameter Store

It authenticates using the pod’s IAM role (usually via IRSA on EKS).

So when the CSI driver says:  

"Fetch /prod/db/password"  

ASCP makes the AWS API call and returns the value.  

3. How secrets get mounted to a pod?    

This happens during pod startup.

Step 1: Pod spec requests CSI volume

Example:

```bash
volumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets
```
This tells Kubernetes:  
Use the Secrets Store CSI driver for this volume  

Step 2: CSI driver looks up SecretProviderClass  

It finds:

```bash
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
```
Step 3: CSI driver calls ASCP

It asks:

Fetch secrets defined here

Step 4: ASCP authenticates to AWS  

Usually using:

Amazon EKS IRSA (IAM Roles for Service Accounts)

Flow:

```bash
Pod
 → ServiceAccount
 → IAM Role
 → Temporary AWS credentials
 → AWS Secrets Manager API
```
Step 5: Secret mounted into pod

Files appear inside container:

/mnt/secrets-store/db-password
/mnt/secrets-store/api-key

Your app reads them like normal files.
