# 12 - Utilizing Ephemeral Volume Types

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Utilizing Ephemeral Volume Types in Kubernetes  
> Lab: Utilizing Ephemeral Volume Types in Kubernetes  
> Topics: emptyDir, ephemeral-storage, volumes, volumeMounts, sizeLimit, multi-container

## Goal

Use `emptyDir` ephemeral volumes for Pod-lifetime data. Confirm data survives container restarts but not Pod deletion. Set `ephemeral-storage` requests/limits and `emptyDir.sizeLimit`.

## Notes

- **Ephemeral volume:** lifetime matches the Pod. Delete the Pod → volume and data are gone. Data **does** survive container restarts.
- Common uses: share data between containers in a Pod, temporary caches, scratch space.
- Read-only config is usually ConfigMaps/Secrets (also mounted as ephemeral volumes). Not covered further here.
- `spec.volumes` + `spec.containers.volumeMounts` work the same for ephemeral and persistent volumes. Ephemeral shows up as `emptyDir: {}` (defaults) or with options.
- Default `emptyDir` lives on the **node disk** under the kubelet Pod path. It can survive node reboot if the Pod stays on that node.
- `medium: Memory` uses tmpfs (faster; counts against memory limits; lost on node reboot).
- Prefer `kubectl exec` to read volume data. SSH to the node is for learning where files live.
- `ephemeral-storage` requests/limits work like CPU/memory. Exceeding the limit can **evict** the Pod.
- `emptyDir.sizeLimit` caps that volume’s size.

## Steps

### Step 1 - Use emptyDir and ephemeral-storage limits

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace ephemeral
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=ephemeral
```

2. Create a Pod that appends coin tosses to a file on an `emptyDir`:

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: coin-toss
spec:
  containers:
  - name: coin-toss
    image: busybox:1.33.1
    command: ["/bin/sh", "-c"]
    args:
    - >
      while true;
      do
        # Record coint tosses
        if [[ $(($RANDOM % 2)) -eq 0 ]]; then echo Heads; else echo Tails; fi >> /var/log/tosses.txt;
        sleep 1;
      done
    # Mount the log directory /var/log using a volume
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  # Declare log directory volume an emptyDir ephemeral volume
  volumes:
  - name: varlog
    emptyDir: {}
EOF
```

3. List the volume on the node (optional; shows where kubelet stores it):

```bash
pod_node=$(kubectl get pod coin-toss -o jsonpath='{.status.hostIP}')
pod_id=$(kubectl get pod coin-toss -o jsonpath='{.metadata.uid}')
ssh $pod_node -oStrictHostKeyChecking=no sudo ls /var/lib/kubelet/pods/$pod_id/volumes/kubernetes.io~empty-dir/varlog
```

Expect `tosses.txt`. For sensitive data, consider `medium: Memory` instead of disk.

4. Explain `emptyDir` fields:

```bash
kubectl explain pod.spec.volumes.emptyDir
```

5. Count lines in the log via `exec`:

```bash
kubectl exec coin-toss -- wc -l /var/log/tosses.txt
```

6. Change the image so the container restarts; watch the Pod:

```bash
kubectl set image pod coin-toss coin-toss=busybox:1.34.0
kubectl get pods -w
```

Restart usually takes under a minute.

7. Stop watching (Ctrl+C), then count lines again:

```bash
kubectl exec coin-toss -- wc -l /var/log/tosses.txt
```

Count should not reset to zero. Volume lifetime follows the Pod, not the container. Writable-layer-only storage would be lost on restart.

8. Delete the Pod:

```bash
kubectl delete pod coin-toss
```

9. Confirm the node path is gone:

```bash
ssh $pod_node -oStrictHostKeyChecking=no sudo ls /var/lib/kubelet/pods/$pod_id/volumes/kubernetes.io~empty-dir/varlog
```

10. Create a Pod with a tiny `ephemeral-storage` request/limit and `sizeLimit: 1Ki`:

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cache
spec:
  containers:
  - name: cache
    image: redis:6.2.5-alpine
    resources:
      requests:
        ephemeral-storage: "1Ki"
      limits:
        ephemeral-storage: "1Ki"
    volumeMounts:
    - name: ephemeral
      mountPath: "/data"
  volumes:
    - name: ephemeral
      emptyDir:
        sizeLimit: 1Ki
EOF
```

Request: node must have enough ephemeral storage or the Pod stays Pending. Limit: max usage.

11. Describe and see eviction after the limit is exceeded:

```bash
kubectl describe pod cache
```

Expect scheduling, then a message that EmptyDir `"ephemeral"` exceeds `"1Ki"`, then eviction/kill.

## Summary

`emptyDir` ties data to the Pod. Use it for logs, caches, and sharing between containers. Set `ephemeral-storage` requests/limits (and `sizeLimit`) so nodes have enough space and one Pod cannot fill the node.
