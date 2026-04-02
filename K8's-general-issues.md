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
