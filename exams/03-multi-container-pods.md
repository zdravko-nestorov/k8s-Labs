# Exam 03 - Multi-Container Pods

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Multi-Container Pods  
> Checks: 3  
> Solution guide: https://app.qa.com/resource/ckad-practice-exam-multi-container-pods-solution-guide/  
> Topics: multi-container, sidecar, emptyDir, volumes, volumeMounts, readOnly, logging, tail, pod-networking, localhost, dns, configmap, configMapKeyRef, postStart, lifecycle, kubectl-replace, kubectl-logs

## Format

Three checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

```bash
# 1 - a running Pod cannot gain a container, so dump, edit, recreate
kubectl -n mcp get pod random -o yaml > pod.yaml
#   add an emptyDir volume named logs
#   mount it at /var/log in BOTH containers
#   second container: args: [/bin/sh, -c, 'tail -n+1 -f /var/log/random.log']
kubectl -n mcp replace --force -f pod.yaml

# 2 - containers in one Pod share a network namespace, so the host is localhost
#   also valid: 127.0.0.1, the Pod name, the Pod IP,
#   or 10-0-0-11.app1.pod.cluster.local

# 3 - read only is a volumeMount field, not a volume field
#   volumeMounts:
#   - name: vol1
#     mountPath: /data
#     readOnly: true
kubectl apply -f /home/ubuntu/md5er-app.yaml
kubectl logs -n app2 md5er c2 > /home/ubuntu/md5er-output.log
```

## Exam strategy notes

- **You cannot add a container to a running Pod.** `spec.containers` is immutable, so `kubectl edit` alone will not do it. Dump the manifest, edit, recreate.
- **`kubectl replace --force -f file` is delete and create in one command.** Faster than running `kubectl delete` then `kubectl create`.
- **`kubectl get pod -o yaml` output is noisy but usable.** The API server ignores the extra status fields, so edit the `spec` and leave the rest. `kubectl edit` shows a cleaner version you can copy from if you prefer.
- **`emptyDir` is how containers in a Pod share files.** Both containers mount the same volume name at the path they need.
- **Containers in a Pod share the network namespace.** They reach each other on `localhost` and must not use the same port.
- **They do not share a filesystem.** Only mounted volumes are shared.
- **`kubectl logs <pod> -c <container>`** picks one container. Without `-c` on a multi-container Pod, kubectl asks you to choose.

Bookmarks that pay off here: Logging Architecture, Volumes, DNS for Services and Pods. The logging page carries the sidecar streaming example you can copy almost as-is.

---

## Task 1 - Pod with legacy logs

Namespace `mcp` · Skill: streaming sidecar over a shared `emptyDir`

**Requirement**

- A Pod has one container, `random`, writing to `/var/log/random.log`
- Add a second container named `second` using the `busybox` image
- `kubectl -n mcp logs random second` must show the log contents

<details><summary><b>My solution</b></summary>

```bash
kubectl -n mcp get pod random -o yaml > pod.yaml
vim pod.yaml
```

Add a `volumeMounts` block to the existing `random` container:

```yaml
    volumeMounts:
    - name: logs
      mountPath: /var/log
```

Add the second container:

```yaml
  - name: second
    image: busybox
    command: ['/bin/sh', '-c', 'tail -F /var/log/random.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
```

Add the volume:

```yaml
  volumes:
  - name: logs
    emptyDir: {}
```

```bash
kubectl -n mcp replace --force -f pod.yaml
kubectl -n mcp describe pod random
kubectl -n mcp logs random second
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n mcp get pod random -o yaml > original.yaml
cp original.yaml second.yaml
```

Edit the copy until it matches this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random
  namespace: mcp
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - while true; do shuf -i 0-1 -n 1 >> /var/log/random.log; sleep 1; done
    image: busybox
    name: random
    volumeMounts:
    - mountPath: /var/log
      name: logs
  - name: second
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/random.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

```bash
kubectl delete -f second.yaml
kubectl create -f second.yaml
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- This is the **streaming sidecar** pattern. The main container keeps writing to a file, the sidecar reads that file and prints to stdout, which is what `kubectl logs` reads.
- The shared `emptyDir` is what makes the file visible to both. Mount it at `/var/log` in **both** containers. Forgetting the mount on the original container is the usual mistake, and it leaves the sidecar staring at an empty directory.
- `emptyDir` is the right volume type when the only goal is sharing data between containers in one Pod. It lives and dies with the Pod.
- `tail -n+1 -f` means start at line 1 and keep following. The `-n+1` part is what replays the lines already written.
- Keep a copy of the original manifest before you edit. If the edit goes wrong, the Pod is already gone.
- The Kubernetes logging page has this exact example, including the `tail` command. Copying it beats writing it from memory.

Review: [labs/05-container-logging-and-sidecar.md](../labs/05-container-logging-and-sidecar.md), [labs/12-ephemeral-volume-types.md](../labs/12-ephemeral-volume-types.md)

Docs: [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) · [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) · [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)

</details>

**Differences that matter**

- `kubectl replace --force -f pod.yaml` is one command against the official two, `kubectl delete` then `kubectl create`. It deletes and recreates in a single step.
- `tail -F` follows the file and retries if it is rotated, but it still starts from the **last 10 lines**. The official `tail -n+1 -f` prints the whole file from the first line. Both pass a check that only looks for output, but `-n+1` is the safer reading of "display the logs written to the file".

---

## Task 2 - Multi-container Pod networking

Namespace `app1` · Skill: shared network namespace

**Requirement**

- Replace `<REPLACE_HOST_HERE>` in the given manifest so container `c2` can make HTTP requests to container `c1`
- Change nothing else in the manifest
- After deploying, `kubectl logs -n app1 webpod -c c2 > /home/ubuntu/webpod-log.txt` must hold a single string

<details><summary><b>My solution</b></summary>

Used `localhost` as the host, then deployed the manifest unchanged otherwise:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
 name: webpod
 namespace: app1
spec:
 restartPolicy: Never
 volumes:
 - name: vol1
   emptyDir: {}
 containers:
 - name: c1
   image: nginx
   volumeMounts:
   - name: vol1
     mountPath: /usr/share/nginx/html
   lifecycle:
     postStart:
       exec:
         command:
           - "bash"
           - "-c"
           - |
             date | sha256sum | tr -d " *-" > /usr/share/nginx/html/index.html
 - name: c2
   image: appropriate/curl
   command: ["/bin/sh", "-c", "curl -s http://localhost && sleep 3600"]
EOF

kubectl logs -n app1 webpod -c c2 > /home/ubuntu/webpod-log.txt
cat /home/ubuntu/webpod-log.txt
```

</details>

<details><summary><b>Official solution</b></summary>

Also `localhost`. The guide lists every value that works:

```text
localhost
127.0.0.1
webpod                                          # the Pod name
<pod-assigned-ip-address>                       # for example 10.0.0.11
<pod-ip-with-dashes>.<namespace>.pod.cluster.local
                                                # for example 10-0-0-11.app1.pod.cluster.local
```

```bash
kubectl get pod -n app1 webpod
kubectl logs -n app1 webpod -c c2
kubectl logs -n app1 webpod -c c2 > /home/ubuntu/webpod-log.txt
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- Containers in a Pod share one network namespace and one IP. `localhost` from `c2` reaches `c1`. This is also why two containers in a Pod cannot both bind port 80.
- The Pod name works because Kubernetes writes the Pod IP and hostname into `/etc/hosts` inside the Pod.
- The dashed form `10-0-0-11.app1.pod.cluster.local` is Pod DNS. Worth recognising, but slower to type and it changes when the Pod restarts.
- Prefer `localhost`. It is short and survives a Pod IP change.
- `curl -s` hides the progress meter, which is what keeps the output to a single string.
- The `postStart` hook writes `index.html` before nginx serves it. Containers start in parallel, so if `c2` ever curls too early you get an empty file. Recreating the Pod fixes it.

Review: no lab covers the shared network namespace inside a Pod. [labs/00-introduction-to-kubernetes-playground.md](../labs/00-introduction-to-kubernetes-playground.md) covers Service discovery between Pods, which is the different case.

Docs: [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) · [Container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

</details>

**Difference that matters**

None, both use `localhost`. Worth memorising the full list of valid hosts, because a check may block the obvious answer and force one of the others.

---

## Task 3 - Add a container with a read-only volume

Namespace `app2` · Skill: `readOnly` volume mount

**Requirement**

- Update `/home/ubuntu/md5er-app.yaml`, adding a second container named `c2` to the `md5er` Pod
- `c2` uses the same `bash` image as `c1`
- `c2` mounts the existing `vol1` volume **read only**
- `c2` runs a script that MD5 hashes each word in `/data/file.txt`
- Save the output with `kubectl logs -n app2 md5er c2 > /home/ubuntu/md5er-output.log`

<details><summary><b>My solution</b></summary>

```yaml
 - name: c2
   image: bash
   volumeMounts:
     - name: vol1
       mountPath: /data
       readOnly: true
   command:
     - /usr/local/bin/bash
     - -c
     - |
       for word in $(</data/file.txt); do
         echo "$word" | md5sum | awk '{print $1}'
       done
```

```bash
kubectl -n app2 apply -f /home/ubuntu/md5er-app.yaml
kubectl -n app2 describe pod md5er
kubectl -n app2 logs md5er c2 > /home/ubuntu/md5er-output.log
cat /home/ubuntu/md5er-output.log
```

</details>

<details><summary><b>Official solution</b></summary>

```yaml
 - name: c2
   image: bash
   volumeMounts:
   - name: vol1
     mountPath: /data
     readOnly: true
   command:
     - "/usr/local/bin/bash"
     - "-c"
     - |
       for word in $(</data/file.txt)
       do 
       echo $word | md5sum | awk '{print $1}'
       done
```

```bash
kubectl apply -f /home/ubuntu/md5er-app.yaml
kubectl logs -n app2 md5er c2
kubectl logs -n app2 md5er c2 > /home/ubuntu/md5er-output.log
```

The guide also shows where `c1` gets its data:

```bash
kubectl describe cm -n app2 cm1
# data:
# ----
# no one has ever done anything like this
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `readOnly: true` belongs on the **volumeMount**, not on the volume. The same volume can be read-write in one container and read-only in another.
- The mount path is dictated by the script, not by the task text. The script reads `/data/file.txt`, so `mountPath: /data` is forced.
- `c1` reads a ConfigMap key into the `DATA` environment variable with `configMapKeyRef`, then writes it into the shared volume. That is the ConfigMap-to-file pattern in miniature.
- Use a YAML block scalar (`|`) for multi-line shell. It keeps newlines and avoids quoting problems.
- `$(</data/file.txt)` is bash reading a file, which is why the image is `bash` and the command is `/usr/local/bin/bash`.
- `restartPolicy: Never` means the containers run once and stop. `kubectl logs` still works after they finish.

Review: [labs/12-ephemeral-volume-types.md](../labs/12-ephemeral-volume-types.md), [labs/10-configmaps-and-secrets.md](../labs/10-configmaps-and-secrets.md)

Docs: [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) · [Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

</details>

**Difference that matters**

None, the two answers are the same. Mine quotes `"$word"`, which is the safer habit in shell but changes nothing for single words.

---

## What to take into the next exam

1. A running Pod cannot gain a container. Dump it, edit it, then `kubectl replace --force -f file`.
2. Copy the original manifest before editing. Recreating means the original is gone.
3. Containers share files only through a volume. `emptyDir` is the in-Pod choice.
4. Containers share the network namespace, so `localhost` is the host, and ports must not clash.
5. `readOnly: true` sits on the volumeMount.
6. Streaming sidecar for file logs: `tail -n+1 -f <file>`.
7. `kubectl logs <pod> -c <container>` for any Pod with more than one container.
