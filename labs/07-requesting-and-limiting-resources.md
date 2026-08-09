# 07 - Requesting and Limiting Resources for Pods

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Mastering Kubernetes Pod Configuration: Defining Resource Requirements  
> Lab: Requesting and Limiting Resources for Pods  
> Topics: resources, requests, limits, cpu, memory, scheduler, kubectl-top, metrics-server

## Goal

Declare CPU and memory requests (minimum) and limits (maximum) on Pods. See how they affect scheduling and actual usage with Metrics Server and `kubectl top`.

## Notes

- Built-in compute resources: **CPU** (cores) and **memory** (bytes).
- **Request:** minimum the scheduler needs free (as reserved by other requests) on a node.
- **Limit:** maximum the container may use.
- Optional, but the scheduler places Pods better when requests are set. A Pod is scheduled only if the node can satisfy the **sum of its containers' requests**.
- The scheduler uses **requested** capacity, not live utilization. A Pod with no request can burn a node’s CPU and still look “free” for scheduling.
- Exceed rules:
  - Over **memory limit** → container terminated (and restarted if possible)
  - Over **memory request** → may be **evicted** when the node is out of memory
  - Over **CPU limit** → may be throttled; not terminated for CPU alone
- You cannot change container resources with a normal update. Use `kubectl apply --force` (delete + recreate) or recreate the Pod.
- Metrics Server enables `kubectl top` for nodes and pods.

## Steps

### Step 1 - Request and limit resources

1. Create a Pod manifest that tries to use 2 CPU cores (full node size):

```bash
cat << 'EOF' > load.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load
spec:
  containers:
  - name: cpu-load
    image: cloudacademydevops/stress
    args:
    - -cpus
    - "2"
EOF
```

2. Create the Pod:

```bash
kubectl apply -f load.yaml
```

This simulates hard-to-find performance degradation in a busy cluster.

3. Show `top` help:

```bash
kubectl top --help
```

4. List Pod resource use:

```bash
kubectl top pods
```

Expect `load` near ~1930m (almost 2 cores). If missing, retry every 15 seconds for about a minute.

5. Confirm a node is near 100% CPU:

```bash
kubectl top nodes
```

Each node has 2 cores, so one node can hit 100% CPU%.

6. Create a similar Pod with requests and limits:

```bash
cat << 'EOF' > load-limited.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-limited
spec:
  containers:
  - name: cpu-load-limited
    image: cloudacademydevops/stress
    args:
    - -cpus
    - "2"
    resources:
      limits:
        cpu: "0.5" # half a core
        memory: "20Mi" # 20 mebibytes
      requests:
        cpu: "0.35" # 35% of a core
        memory: "10Mi" # 10 mebibytes
EOF
```

Scheduled only if a node has at least 0.35 CPU and 10Mi free **by request accounting**. The unrestricted `load` Pod uses ~2 CPUs but requested nothing, so it does not reduce free request capacity.

7. Show how nodes account for non-terminated Pods:

```bash
kubectl describe nodes | grep --after-context=5 "Non-terminated Pods"
```

The node with `load` still shows no request/limit allocated for that Pod.

8. Create the limited Pod:

```bash
kubectl apply -f load-limited.yaml
```

9. See which node runs each Pod:

```bash
kubectl get pods -o wide
```

`load-limited` may land on the other node by chance. It is not guaranteed while `load` has no CPU request.

10. Wait about a minute, then check Pod metrics:

```bash
kubectl top pods
```

`load-limited` should sit near its limit (~499m). Requests drive scheduling; limits cap usage.

If both Pods share a node, `load-limited` still stays near half a core, and `load` may drop by about half a core.

11. Give `load` a CPU request of 1.7 (leave headroom for system Pods):

```bash
cat << 'EOF' > load.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load
spec:
  containers:
  - name: cpu-load
    image: cloudacademydevops/stress
    args:
    - -cpus
    - "2"
    resources:
      requests:
        cpu: "1.7"
EOF
```

12. Force-apply (resource updates need recreate):

```bash
kubectl apply --force -f load.yaml
```

13. Delete `load-limited`:

```bash
kubectl delete pod load-limited
```

14. Recreate `load-limited`:

```bash
kubectl apply -f load-limited.yaml
```

15. Confirm the Pods are on different nodes:

```bash
kubectl get pods -o wide
```

Combined CPU request is 2.1. Each node has only 2 CPUs, so they cannot share a node.

## Summary

Requests guide scheduling. Limits reduce contention. Set both so the scheduler and the runtime behave predictably.
