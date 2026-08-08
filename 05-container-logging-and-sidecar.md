# 05 - Container Logging and Sidecar Logging Agent

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: (logging labs)  
> Lab: Understanding Container Logging / Logging Agent and Sidecar Pattern  
> Topics: logging, kubectl-logs, kubectl-exec, kubectl-cp, multi-container, sidecar, fluentd, configmap, volumes, s3

## Goal

Capture container logs from stdout/stderr with `kubectl logs`. Access file-based logs with `exec` and `cp`. Then use a Fluentd sidecar and a shared volume to stream logs to S3.

## Notes

- Kubernetes stores what containers write to **standard output** and **standard error** as logs.
- For logs only in files: use `kubectl exec`, `kubectl cp`, or a logging sidecar (next step).
- Multi-container Pods: `kubectl logs <pod> <container>`. Omit the container name if the Pod has one container.
- `kubectl expose` without `--port` uses the container port (here 80).
- **Sidecar pattern:** a helper container extends the main container. For logging, the sidecar is a logging agent.
- Shared **volume** mounts give the sidecar access to log files.
- **ConfigMaps** hold config (such as `fluent.conf`) so images stay reusable.
- First S3 objects can take a couple of minutes. Refresh if the `logs/` folder is missing.

---

## Step 1 - Understanding container logging

### Introduction

Kubernetes captures stdout and stderr. File-based logs need more work (`exec`, `cp`, or a sidecar).

### Instructions

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace logs
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=logs
```

2. Create a multi-container Pod (server + client):

```bash
cat << 'EOF' > pod-logs.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: logs
  name: pod-logs
spec:
  containers:
  - name: server
    image: busybox:1.30.1
    ports:
    - containerPort: 8888
    # Listen on port 8888
    command: ["/bin/sh", "-c"]
    # -v for verbose mode
    args: ["nc -p 8888 -v -lke echo Received request"]
    readinessProbe:
      tcpSocket:
        port: 8888
  - name: client
    image: busybox:1.30.1
    # Send requests to server every 5 seconds
    command: ["/bin/sh", "-c"]
    args: ["while true; do sleep 5; nc localhost 8888; done"]
EOF
kubectl create -f pod-logs.yaml
```

The server uses `nc -v` so it writes to stdout. The client connects to the server every 5 seconds.

3. Get server logs:

```bash
kubectl logs pod-logs server
```

After the listening message, you see a connection line per client request. More examples: `kubectl logs --help`.

4. Follow the latest client log line with timestamps:

```bash
kubectl logs -f --tail=1 --timestamps pod-logs client
```

You should see a new request about every five seconds.

5. Stop streaming: Ctrl+C.

6. Create an Apache web server Pod and expose it with a LoadBalancer:

```bash
cat << 'EOF' > pod-webserver.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: logs
  name: webserver-logs
spec:
  containers:
  - name: server
    image: httpd:2.4.38-alpine
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
EOF
kubectl create -f pod-webserver.yaml
kubectl expose pod webserver-logs --type=LoadBalancer
```

7. Wait for EXTERNAL-IP:

```bash
watch kubectl get services
```

8. Copy the EXTERNAL-IP DNS address. Press Ctrl+C to stop watching.

9. Open the DNS address in a browser.

Note: If it does not load, wait 1-2 minutes for load balancer health checks. Refresh every minute until it loads.

10. Refresh a few times, then open `/oops` to get a Not Found error.

11. Show webserver logs:

```bash
kubectl logs webserver-logs
```

Browser requests (for example `GET /favicon.ico`, `GET /oops`) mix with readiness probe requests. This httpd image sends access and error logs to stdout/stderr, not files.

12. For file-style access (legacy apps), read the last 10 lines of a file in the container:

```bash
kubectl exec webserver-logs -- tail -10 conf/httpd.conf
```

Same idea works for real log files.

13. Copy a file from the container to the bastion:

```bash
kubectl cp webserver-logs:conf/httpd.conf local-copy-of-httpd.conf
```

Source is `pod:path`. You can also copy local → Pod. See `kubectl cp --help`.

### Validation (Step 1)

- **Service Exposing Deployment Via Load Balancer:** Service reachable via LoadBalancer (lab check name may say Deployment).

### Summary (Step 1)

Stdout/stderr become Kubernetes logs. For file logs, use `exec` or `cp`.

---

## Step 2 - Logging agent sidecar (Fluentd → S3)

### Introduction

A logging sidecar streams primary-container log files to a central store. Both containers mount the log path. This lab uses Fluentd with an S3 plugin and an S3 bucket.

### Instructions

1. Store the lab logs S3 bucket name:

```bash
s3_bucket=$(aws s3api list-buckets --query "Buckets[].Name" --output table | grep logs | tr -d \|)
```

2. Verify:

```bash
echo $s3_bucket
```

Expect a name starting with `cloudacademylabs-k8slogs-`.

3. Create a ConfigMap with Fluentd config (bucket name from `$s3_bucket`):

```bash
cat << EOF > fluentd-sidecar-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    # First log source (tailing a file at /var/log/1.log)
    <source>
      @type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    # Second log source (tailing a file at /var/log/2.log)
    <source>
      @type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    # S3 output configuration (Store files every minute in the bucket's logs/ folder)
    <match **>
      @type s3

      s3_bucket $s3_bucket
      s3_region us-west-2
      path logs/
      buffer_path /var/log/
      store_as text
      time_slice_format %Y%m%d%H%M
      time_slice_wait 1m

      <instance_profile_credentials>
      </instance_profile_credentials>
    </match>
EOF
kubectl create -f fluentd-sidecar-config.yaml
```

Two sources under `/var/log` (tags `count.format1` and `count.format2`). The `match` block sends everything to the S3 `logs/` prefix.

4. Create the Pod with primary `count` and Fluentd sidecar `count-agent`:

```bash
cat << 'EOF' > pod-counter.yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - >
      i=0;
      while true;
      do
        # Write two log files along with the date and a counter
        # every second
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    # Mount the log directory /var/log using a volume
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: lrakai/fluentd-s3:latest
    env:
    - name: FLUENTD_ARGS
      value: -c /fluentd/etc/fluent.conf
    # Mount the log directory /var/log using a volume
    # and the config file
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /fluentd/etc
  # Use host network to allow sidecar access to IAM instance profile credentials
  hostNetwork: true
  # Declare volumes for log directory and ConfigMap
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
EOF
kubectl create -f pod-counter.yaml
```

Both containers mount `varlog` at `/var/log`. The sidecar also mounts the ConfigMap. `hostNetwork: true` lets the sidecar use the instance profile for S3.

5. Open the lab cloud environment (Open button in the lab UI).

6. Sign in with the **lab session** IAM user from the lab UI (Account ID, user `student`, password shown in the lab). Do not reuse old passwords from notes.

7. Open the S3 console. Open the bucket named like `cloudacademylabs-k8slogs-...`.

Expect a `logs/` folder with files about every minute.

Warning: First upload can take a couple of minutes. Refresh if `logs/` is missing.

8. Open one log file in `logs/` (allow pop-ups if the browser blocks them). Confirm the sidecar streamed logs to S3.

### Validation (Step 2)

- **Pod Logs in S3 Bucket:** Fluentd sidecar uploaded logs to S3.

### Summary (Step 2)

Sidecar + shared volume + ConfigMap streams file logs to S3 (or any Fluentd-supported backend).

---

## Summary (both steps)

1. Prefer stdout/stderr + `kubectl logs`.
2. Use `exec` / `cp` for ad hoc file logs.
3. Use a logging sidecar when you need continuous shipping to a central store.
