# Exam 04 - Observability

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Observability  
> Checks: 5  
> Solution guide: https://app.qa.com/resource/ckad-practice-exam-observability-solution-guide/  
> Topics: probes, livenessProbe, readinessProbe, httpGet, troubleshooting, endpoints, services, nodeport, kubectl-logs, label-selector, kubectl-exec, kubectl-cp, kubectl-top, metrics-server, kubectl-edit, kubectl-describe

## Format

Five checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

```bash
# 1 - probes have no flag, so generate and edit
kubectl run nginx -n ca1 --image=nginx --port=80 --dry-run=client -o yaml > pod.yaml
#   add livenessProbe under the container, then apply

# 2 - find the broken Service: empty endpoints means no ready Pods
kubectl -n hosting get endpoints
kubectl -n hosting describe pods --selector app=web2   # Events name the failing probe
kubectl -n hosting edit deployment web2                # fix the probe port

# 3 - kubectl logs takes a label selector and merges the output
kubectl logs -n ca2 -l app=prod | wc -l > /home/ubuntu/combined-row-count-prod.txt

# 4 - pull a text file out of a container
kubectl exec -n ca2 skynet -- cat /skynet/t2-specs.txt > /home/ubuntu/t2-specs.txt

# 5 - highest CPU Pod by name
kubectl top pods -n matrix --sort-by=cpu --no-headers | head -n1 | awk '{print $1}' > /home/ubuntu/max-cpu-podname.txt
```

## Exam strategy notes

- **Probes have no imperative flag.** Generate the Pod with `kubectl run --dry-run=client -o yaml`, then add the probe block.
- **Debug a dead Service in this order:** `kubectl get endpoints` first. Empty endpoints means no ready Pods, which is a Pod problem, not a Service problem. Only then look at selectors and ports.
- **A Service never routes to a Pod that is not Ready.** A failing readiness probe silently removes a Pod from the Service.
- **`kubectl logs` accepts `-l`**, so you rarely need a loop over Pods. Add `--prefix` to see which Pod each line came from.
- **`kubectl top` needs metrics-server** and `--sort-by` takes only `cpu` or `memory`.
- **`kubectl exec -- cat` beats `kubectl cp` for text.** `kubectl cp` needs the `tar` binary inside the container.
- **Prefer `awk '{print $1}'` over `cut`** when splitting kubectl output. Column widths vary, and awk treats any run of spaces as one separator.

---

## Task 1 - Create an nginx Pod with a liveness HTTP GET probe

Namespace `ca1` · Skill: probe block in a manifest

**Requirement**

- Pod named `nginx` in Namespace `ca1`, image `nginx`
- Listens on port 80
- Liveness probe with type `httpGet`, path `/`, port 80
- Initial delay 10 seconds, polling period 5 seconds

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: ca1
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
EOF

kubectl -n ca1 describe pod nginx
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl run nginx -n ca1 --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > pod-livenessprobe.yaml
vim pod-livenessprobe.yaml
```

The edited file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: ca1
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
kubectl apply -f pod-livenessprobe.yaml
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- No `kubectl run` flag creates a probe, so the manifest is unavoidable. Only the wrapper differs: generate then edit, or type it out.
- `initialDelaySeconds` is the wait before the first check. `periodSeconds` is the gap between checks. Defaults are 0 and 10.
- A liveness probe failing restarts the container. A readiness probe failing removes the Pod from Service endpoints. Startup probes hold the other two off while a slow app boots.
- `port: 80` may also be written as a named port, matching a `name:` under `ports`.
- `kubectl describe pod` prints the probe as one line, for example `Liveness: http-get http://:80/ delay=10s timeout=1s period=5s #success=1 #failure=3`. Read that line to confirm your values landed.

Review: [labs/06-probes-and-monitoring.md](../labs/06-probes-and-monitoring.md)

Docs: [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

</details>

**Difference that matters**

Both work. One detail is worth noticing: the official command adds `--restart=Never`, which the task never asked for. That combination is odd, because a liveness probe exists to restart an unhealthy container, and `restartPolicy: Never` stops that from happening. My version leaves the default `Always`, which matches what a liveness probe is for.

---

## Task 2 - Hosting Service not working

Namespace `hosting` · Skill: diagnosing a Service with no endpoints

**Requirement**

- A Service in Namespace `hosting` is not responding to requests
- Work out which Service is broken and fix the underlying cause

<details><summary><b>My solution</b></summary>

```bash
# the livenessProbe must listen on port 80, not 30
kubectl -n hosting edit deployment web2

kubectl -n hosting exec -it web1-c69c6b569-hkbmj -- curl web1:80
kubectl -n hosting exec -it web1-c69c6b569-hkbmj -- curl web2:80
```

</details>

<details><summary><b>Official solution</b></summary>

The diagnosis path:

```bash
kubectl -n hosting get service -o wide          # shows the selectors
kubectl -n hosting get endpoints                # web2 has none
kubectl -n hosting get pods --selector app=web2 # two Pods, so labels are fine
kubectl -n hosting describe pods --selector app=web2
```

The describe output shows the cause:

```text
Readiness:      http-get http://:30/ delay=3s timeout=1s period=3s #success=1 #failure=3
```

The probe checks port 30, the container listens on port 80. The written solution patches it:

```bash
cat << EOF > patch.yaml
spec:
  template:
    spec:
      containers:
      - name: web2
        readinessProbe:
          httpGet:
            port: 80
EOF
kubectl -n hosting patch deployments web2 --patch "$(cat patch.yaml)"
```

The guide says plainly that in the exam you should use `kubectl edit` instead, and that the patch is only written this way so the answer reads as one command.

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl get endpoints` is the fastest signal. A Service with no endpoints has no ready Pods behind it, so the fault is in the Pods.
- Then work backwards: do Pods match the selector, are they Running, are they Ready. Two Pods matching `app=web2` rules out a label mistake straight away.
- `kubectl describe pods --selector <label>` describes every matching Pod in one command. The Events section names the failing probe.
- A readiness probe on the wrong port is the classic cause. The container port and the probe port are separate values and nothing checks that they agree.
- Both Services here are `NodePort`, so the outside test is `curl <NodeIP>:<NodePort>` with node IPs from `kubectl get nodes -o wide`.

Review: [labs/06-probes-and-monitoring.md](../labs/06-probes-and-monitoring.md), [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md)

Docs: [Configure probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) · [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

</details>

**Differences that matter**

- The official guide agrees with my approach over its own. It states that `kubectl edit` is what to use in the exam and that building the patch takes longer.
- My verification is also simpler. `kubectl exec -- curl web2:80` from a Pod already in the cluster uses Service DNS, so there is no need to look up node IPs and NodePort numbers. Testing `web1` first proves the method works before testing the broken one.
- One correction to my own note: the failing probe was the **readiness** probe, not liveness. That distinction is the whole reason the Service had no endpoints. A failing liveness probe would have restarted the containers instead.

---

## Task 3 - Pod log analysis

Namespace `ca2` · Skill: `kubectl logs -l`

**Requirement**

- Namespace `ca2` holds Pods labelled `app=test` or `app=prod`, each logging a static list of numbers
- Combine the logs of all Pods labelled `app=prod`
- Get the total row count and save it to `/home/ubuntu/combined-row-count-prod.txt`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n ca2 get pods --show-labels -l app=prod
kubectl -n ca2 logs test1

for pod in $(kubectl -n ca2 get pods -l app=prod -o name); do
	for number in $(kubectl -n ca2 logs $pod); do
		echo "$number" >> /home/ubuntu/combined-rows-prod.txt
	done
done

wc -l /home/ubuntu/combined-rows-prod.txt | awk '{print $1}' > /home/ubuntu/combined-row-count-prod.txt
cat /home/ubuntu/combined-row-count-prod.txt
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl logs -n ca2 -l app=prod | wc -l > /home/ubuntu/combined-row-count-prod.txt
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl logs` takes `-l/--selector` and merges the output of every matching Pod. No loop needed.
- `--prefix` tags each line with its Pod and container name, which is how you tell the sources apart.
- `--max-log-requests` defaults to 5 and caps concurrent log streams when following by selector. Worth knowing if a task involves many Pods and `-f`.
- `wc -l` on the piped stream avoids writing an intermediate file at all.
- Read the task for how many files it wants. This one asked for the count only.

Review: [labs/05-container-logging-and-sidecar.md](../labs/05-container-logging-and-sidecar.md)

Docs: [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

This is the one to learn from. A single pipe replaces eight lines.

There is also a bug hiding in mine. `for number in $(kubectl logs $pod)` splits on **whitespace**, not on newlines, so a log line containing two words would be written as two rows and inflate the count. It only gave the right answer because each line here is a single number.

My version also created `/home/ubuntu/combined-rows-prod.txt`, which the task never asked for.

---

## Task 4 - Pod diagnostics, extract a file

Namespace `ca2` · Skill: getting a file out of a container

**Requirement**

- Pod `skynet` in Namespace `ca2` contains `/skynet/t2-specs.txt`
- Extract that file to `/home/ubuntu/t2-specs.txt`

<details><summary><b>My solution</b></summary>

```bash
kubectl cp ca2/skynet:skynet/t2-specs.txt /home/ubuntu/t2-specs.txt
cat /home/ubuntu/t2-specs.txt
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl exec -n ca2 skynet -- cat /skynet/t2-specs.txt > /home/ubuntu/t2-specs.txt
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl cp <ns>/<pod>:<path> <local>` copies files both ways, but it shells out to `tar` **inside the container**. No `tar` binary, no copy. Many small images have none.
- `kubectl exec -- cat > file` depends only on `cat` and works for any text file. For binary content it is unsafe, and `kubectl cp` is the right tool.
- `kubectl cp` strips a leading `/` from the source and prints a warning about it. Writing the path without the slash, as I did, avoids the warning and resolves from the container working directory.
- Add `-c <container>` for a multi-container Pod, otherwise both commands pick the first container.

Review: [labs/05-container-logging-and-sidecar.md](../labs/05-container-logging-and-sidecar.md)

Docs: [Get a shell to a running container](https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

Both are correct. `kubectl exec -- cat` is the safer default because it has no dependency inside the container. Keep `kubectl cp` for binaries, whole directories, and for copying files **into** a Pod.

---

## Task 5 - Pod CPU utilisation

Namespace `matrix` · Skill: `kubectl top --sort-by`

**Requirement**

- Find the Pod using the most CPU in Namespace `matrix`
- Write its name to `/home/ubuntu/max-cpu-podname.txt`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n matrix top pod --sort-by cpu --no-headers=true --no-headers=true | head -n 1 | awk '{print $1}' > /home/ubuntu/max-cpu-podname.txt
cat /home/ubuntu/max-cpu-podname.txt
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl top pods -n matrix --sort-by=cpu --no-headers=true | head -n1 | cut -d" " -f1 > /home/ubuntu/max-cpu-podname.txt
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `--sort-by` accepts only `cpu` or `memory`. Confirmed on kubectl v1.36.3. It sorts descending, so the first row is the highest.
- `--no-headers` is a boolean flag, so writing it bare is the same as `=true`.
- `kubectl top` needs metrics-server running. Without it the command errors instead of returning empty.
- Readings are a recent average, not an instant value, so numbers move between runs.

Review: [labs/06-probes-and-monitoring.md](../labs/06-probes-and-monitoring.md), [labs/07-requesting-and-limiting-resources.md](../labs/07-requesting-and-limiting-resources.md)

Docs: [kubectl top pod](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#top) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

- I passed `--no-headers=true` twice. Harmless, kubectl takes the last value, but it is noise.
- `awk '{print $1}'` is the better half of my answer. `cut -d" " -f1` works here only because the Pod name has no leading space. awk treats any run of spaces as one separator, so it survives column padding.

---

## What to take into the next exam

1. Probes have no imperative flag. Generate with `--dry-run=client -o yaml`, then add the block.
2. Broken Service? `kubectl get endpoints` first. Empty means no ready Pods, so the fault is in the Pods.
3. A failing readiness probe drops a Pod from its Service. A failing liveness probe restarts the container.
4. `kubectl describe pods --selector <label>` describes a whole group in one command, and Events name the cause.
5. `kubectl logs -l <label>` merges logs across Pods. Never write a loop for this.
6. Shell loops over command output split on whitespace, not newlines. Avoid them for counting rows.
7. `kubectl exec -- cat > file` to pull text out of a container. `kubectl cp` only when you need binaries or directories, and only when the container has `tar`.
8. `kubectl top pods --sort-by=cpu --no-headers | head -n1 | awk '{print $1}'`.
9. Fields with no imperative flag so far: probes join `terminationGracePeriodSeconds`, resources, `securityContext`, volumes, and `envFrom`.
