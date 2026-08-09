# 06 - Probes and Application Monitoring

> Course: Cloud Native Champions: CKAD Bootcamp  
> Lab: Using Probes to Better Understand Pod Health / Monitoring Kubernetes Applications  
> Topics: readiness-probe, liveness-probe, startup-probe, httpGet, tcpSocket, exec-probe, metrics-server, kubectl-top, events, resources

## Goal

Use readiness and liveness probes to detect not-ready and broken Pods. Then monitor the cluster with events, `describe`, `logs`, and Metrics Server via `kubectl top`.

## Notes

- **Running** does not mean ready or healthy.
- **Readiness probe:** Pod not ready for traffic (startup, cache warm-up). Failing Pods are removed from Service endpoints.
- **Liveness probe:** Pod stuck (for example deadlock). Failure restarts the container.
- **Startup probe:** Runs before readiness/liveness for slow starts. Stops after first success.
- A container may define at most one of each probe type. Config shape is the same; failure handling differs.
- Probe actions: `exec` (exit 0 = success), `httpGet` (2xx/3xx = success), `tcpSocket` (connect = success).
- Timing fields: `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold`, `initialDelaySeconds`.
- After success, a probe needs consecutive failures (`failureThreshold`, default often 3) before it is considered failed.
- `kubectl top` needs Metrics Server (or another metrics API). Prometheus is a fuller pipeline but out of scope here.
- Set CPU/memory **requests** and **limits** on containers. See `kubectl explain pod.spec.containers.resources`.

---

## Step 1 - Using probes for Pod health

### Instructions

1. Explain readiness probes:

```bash
kubectl explain pod.spec.containers.readinessProbe
```

Read DESCRIPTION and FIELDS. Actions: `exec`, `httpGet`, `tcpSocket`.

2. Create a Pod with an HTTP readiness probe (server starts after 30s sleep):

```bash
cat << 'EOF' > pod-readiness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness
  name: readiness-http
spec:
  containers:
  - name: readiness
    image: httpd:2.4.38-alpine
    ports:
    - containerPort: 80
    # Sleep for 30 seconds before starting the server
    command: ["/bin/sh","-c"]
    args: ["sleep 30 && httpd-foreground"]
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
EOF
kubectl create -f pod-readiness.yaml
```

Custom headers: `kubectl explain pod.spec.containers.readinessProbe.httpGet`.

3. Describe the Pod and check Events / Conditions:

```bash
kubectl describe pod readiness-http
```

Expect about 10 failed probes while sleeping (`initialDelaySeconds: 3`, then every 3s for ~30s). When ready, **Ready** and **ContainersReady** are `True`. The Containers section shows the probe config.

4. Kill httpd inside the container to fail readiness again:

```bash
kubectl exec readiness-http -- pkill httpd
```

`--` ends kubectl option parsing; the rest runs in the container.

5. Describe again. Ready conditions should be `False`.

Note: httpd may recover after about a minute. If conditions are already `True`, kill httpd again and describe quickly.

6. Create a Pod with a TCP liveness probe:

```bash
cat << 'EOF' > pod-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-tcp
spec:
  containers:
  - name: liveness
    image: busybox:1.30.1
    ports:
    - containerPort: 8888
    # Listen on port 8888 for 30 seconds, then sleep
    command: ["/bin/sh", "-c"]
    args: ["timeout 30 nc -p 8888 -lke echo hi && sleep 600"]
    livenessProbe:
      tcpSocket:
        port: 8888
      initialDelaySeconds: 3
      periodSeconds: 5
EOF
kubectl create -f pod-liveness.yaml
```

`nc` listens for 30s, then `timeout` kills it. The probe checks port 8888 every 5s.

7. Watch until the probe fails and the Pod restarts:

```bash
watch kubectl describe pod liveness-tcp
```

Look for Unhealthy / BackOff events and Restart Count rising (about every minute after failures).

8. Stop watching: Ctrl+C.

9. Delete the Pods:

```bash
kubectl delete -f pod-readiness.yaml
kubectl delete -f pod-liveness.yaml
```

### Summary (Step 1)

Readiness delays traffic until ready. Liveness restarts broken containers. Startup probes separate slow start from post-start checks.

---

## Step 2 - Monitoring Kubernetes applications

### Introduction

Track node CPU/memory, desired vs actual Pods, errors, and more. Built-in tools plus Metrics Server for `kubectl top`. Fuller stacks (Prometheus) are out of scope here.

### Instructions

1. Check Pods across all namespaces:

```bash
kubectl get pods --all-namespaces
```

READY is `ready/total` containers. For Deployments, `get` shows desired vs actual. Also use `describe`, `logs`, and `get -o yaml` when debugging.

2. List events in `default`:

```bash
kubectl get events -n default
```

Events are namespaced. Use `-o wide` for more detail. Per-resource events also appear in `describe`.

3. See how `top` works:

```bash
kubectl top
```

Needs a metrics API implementation (Metrics Server in this lab).

4. Node utilization:

```bash
kubectl top node
```

CPU in cores and %. Example: 2 cores → max `2000m`. Watch memory pressure as you add Pods.

5. Pod utilization in `kube-system`:

```bash
kubectl top pods -n kube-system
```

6. Per-container utilization:

```bash
kubectl top pod -n kube-system --containers
```

`--containers` is useful for multi-container Pods. NAME is the container name.

7. Filter with a label selector:

```bash
kubectl top pod -n kube-system --containers -l k8s-app=kube-dns
```

### Extra practice (if time remains)

- Find which container in `kube-system` uses the most CPU or memory.
- Read Debian version from kube-proxy Pods: `/etc/debian_version` (for example via `kubectl exec`).

### Summary (Step 2)

Use `get`, `describe`, `logs`, `events`, and `kubectl top` (Metrics Server) to monitor and debug.

---

## Summary (both steps)

1. Probes express readiness and liveness beyond Pod phase.
2. Monitoring starts with kubectl and Metrics Server; add Prometheus when you need a full metrics pipeline.
