# Challenge 02 - Certified Kubernetes Application Developer (CKAD) Challenge

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: Lab Challenge, no official solution  
> Checks: 4  
> Result: 4 of 4 passed  
> Reviewed against: `labs/`, `exams/`, `exams/CHEATSHEET.md`  
> Topics: serviceaccount, deployments, kubectl-create, kubectl-set, command-vs-args, resources, requests, limits, qos, guaranteed, evictions, node-pressure, limitrange, pvc, persistentvolumeclaim, volumes, emptyDir, volumeName, storageclass, multi-container, adapter, kubectl-edit, jsonpath

## Format

Four validation checks, no solution guide. Nothing here is an official answer. The review blocks
compare my commands against my own labs and exams, plus three facts I verified while writing this up.
A wrong note in those files becomes a wrong note here.

Solutions are collapsed. Open one only after you have tried the task.

## Score card

| Task | Result | What it drills |
|------|--------|----------------|
| 1 - ServiceAccount on a Deployment | Correct | attaching an SA, and `command` against `args` |
| 2 - Survive eviction with Guaranteed QoS | Correct, proved on the wrong object | requests must equal limits, and where the API fills them in |
| 3 - Persist a multi-container Pod's data | Passed, but nothing is persisted | a claim binds by StorageClass, `volumeName` pins it to a PV |
| 4 - Name the multi-container pattern | Correct | adapter against sidecar against ambassador |

**All four checks passed.** Task 3 is the one to reread anyway, because it passed with a claim that
cannot bind. The grader read the manifests and agreed with them. Nothing was persisted. That fault
came from applying my own exam 07 note, "pin a claim with `volumeName`", to a task that needed dynamic
provisioning.

Verification is again the thin part. Two of the four checks I ran read the wrong object, so neither
would have caught its own task going wrong.

## Best commands I know now

```bash
# 1 - three commands, no YAML. create deployment writes command, not args
kubectl -n dwx7eq create sa inspector
kubectl -n dwx7eq create deployment calins --image=busybox:1.31.1 --replicas=1 -- sleep 24h
kubectl -n dwx7eq set serviceaccount deployment calins inspector

# 2 - Guaranteed needs request == limit for cpu and memory, on every container
kubectl -n bk0c2d set resources deployment mission-critical \
  --limits=cpu=200m,memory=200Mi --requests=cpu=200m,memory=200Mi
kubectl -n bk0c2d get pods -o custom-columns='NAME:.metadata.name,QOS:.status.qosClass'

# 3 - a claim with no volumeName, then swap the volume in place
kubectl get sc                                 # is one marked (default)?
kubectl -n zuc0co apply -f pvc.yaml            # 2Gi, ReadWriteOnce, nothing else
kubectl -n zuc0co edit deployment legacy       # emptyDir -> persistentVolumeClaim, keep the name
kubectl -n zuc0co get pvc,pods                 # PVC Bound, new Pod Running

# 4 - the helper reformats the app's output for an outside reader, so it is the adapter
echo adapter > /home/ubuntu/mcpod
```

---

## Task 1 - Service Account

Namespace `dwx7eq` · Skill: attach a ServiceAccount to a Deployment, and tell `command` from `args`

**Requirement**

- ServiceAccount named `inspector` in Namespace `dwx7eq`
- Deployment named `calins` in the same Namespace
- Image `busybox:1.31.1` for the only container
- Pass the arguments `sleep` and `24h` to the container
- 1 replica
- The Deployment's Pod uses the `inspector` ServiceAccount

<details><summary><b>My solution</b></summary>

```bash
kubectl -n dwx7eq create sa inspector
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calins
  namespace: dwx7eq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: calins
  template:
    metadata:
      labels:
        app: calins
    spec:
      serviceAccountName: inspector
      containers:
      - name: busybox
        image: busybox:1.31.1
        command:
        - sleep
        - 24h
EOF

kubectl -n dwx7eq get deployment calins -o jsonpath="{.spec.template.spec.serviceAccountName}{'\n'}"
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: correct, and it uses a better field name than my own lab note does.**

What is right:

- `kubectl create sa inspector` is the cheat sheet entry from exam 02 task 2. `sa` is the short name.
- `serviceAccountName` is the current field. [labs/11-using-serviceaccounts.md](../labs/11-using-serviceaccounts.md) writes `serviceAccount`, which is the older alias. Both work and the API mirrors one into the other, but `serviceAccountName` is the one the docs and `kubectl set serviceaccount` use.
- The selector and the template labels agree, which is what a hand-written Deployment gets wrong most often.
- The verification reads exactly one field with JSONPath, and `{'\n'}` ends the line so the next prompt is readable. That habit came from exam 01.

**The YAML was avoidable. Three commands do the same job:**

```bash
kubectl -n dwx7eq create sa inspector
kubectl -n dwx7eq create deployment calins --image=busybox:1.31.1 --replicas=1 -- sleep 24h
kubectl -n dwx7eq set serviceaccount deployment calins inspector
```

Two details make this safe, and I checked both on kubectl v1.36.3 rather than trusting memory:

- `kubectl create deployment ... -- sleep 24h` writes the two words into `command`, not `args`. That is the same field the hand-written manifest used, so the shortcut produces the same object.
- `kubectl set serviceaccount` writes `serviceAccountName`. It triggers a rollout, which is fine here because the Deployment was created a second earlier.

`kubectl run --serviceaccount` is gone from v1.36.3. The tip in lab 11 is stale, and `kubectl set serviceaccount` is its replacement. Note the flag was never available on `kubectl create deployment` either.

**`command` against `args`, since the task said "arguments".**

| Field | Replaces | Docker name |
|-------|----------|-------------|
| `command` | the image entrypoint | ENTRYPOINT |
| `args` | the arguments passed to it | CMD |

`busybox:1.31.1` sets `CMD ["sh"]` and no entrypoint, so `command: [sleep, 24h]` and
`args: [sleep, 24h]` both end up running `sleep 24h`. The container stays up either way. The only
risk is a grader that reads `.args` literally, and nothing in my notes suggests these checks do that.

Worth remembering because the two generators disagree: text after `--` becomes **args** for
`kubectl run` and **command** for `kubectl create deployment`. `kubectl run --command` switches run
over to `command`.

Source in my notes: [exams/02-configuration.md](../exams/02-configuration.md) task 2, [labs/11-using-serviceaccounts.md](../labs/11-using-serviceaccounts.md), [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) ServiceAccounts and Pods

Docs: [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) · [Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)

</details>

**What to fix next time**

Nothing is wrong. Learn the three-command path so a task like this costs 20 seconds instead of a
heredoc, and check the Pod as well as the template: `kubectl -n dwx7eq get pods -o jsonpath='{.items[*].spec.serviceAccountName}'`.

---

## Task 2 - Evictions

Namespace `bk0c2d` · Skill: Guaranteed QoS, which means request equals limit on every container

**Requirement**

- The `mission-critical` Deployment in Namespace `bk0c2d` gets evicted when the cluster is short of memory
- Change it so it is evicted only when higher priority Pods need the room, which is Guaranteed Quality of Service
- The container needs, and will never exceed, 200 milliCPU and 200 mebibytes of memory

<details><summary><b>My solution</b></summary>

```bash
kubectl -n bk0c2d get deployment mission-critical -o yaml > deployment-mission-critical.yaml

vim deployment-mission-critical.yaml

# Add resource limits:
        resources:
          limits:
            cpu: "200m"
            memory: "200Mi"

kubectl apply -f deployment-mission-critical.yaml
kubectl -n bk0c2d get deployment mission-critical -o jsonpath="{.spec.template.spec.containers[*].resources}{'\n'}"
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: the change is correct. Limits alone do give Guaranteed QoS. The command used to prove it shows the opposite.**

**Why limits alone are enough.** The API fills in a missing request from the limit. The exact
sentence from the docs: "If you specify a limit for a resource, but do not specify any request, and no
admission-time mechanism has applied a default request for that resource, then Kubernetes copies the
limit you specified and uses it as the requested value for the resource." So each container ends up
with `requests == limits` for both CPU and memory, which is the definition of Guaranteed.

**Why the verification misleads.** That copy happens on a **Pod**, never on a pod template. The
Kubernetes source says so in a comment next to the code: "we only want this defaulting semantic to
take place on a v1.Pod and not a v1.PodTemplate". So the command that was run prints:

```json
{"limits":{"cpu":"200m","memory":"200Mi"}}
```

No requests. Read at face value that is Burstable, not Guaranteed. The Pod is where the answer lives:

```bash
kubectl -n bk0c2d get pods -o custom-columns='NAME:.metadata.name,QOS:.status.qosClass'
kubectl -n bk0c2d get pod <pod> -o jsonpath='{.status.qosClass}{"\n"}'      # Guaranteed
```

`.status.qosClass` is the graded fact, and it is computed by the cluster, so it cannot be fooled by a
manifest that looks right.

**The one case where limits alone fail.** Re-read the docs sentence: "and no admission-time mechanism
has applied a default request". A LimitRange in the Namespace with a `defaultRequest` fills the
request with **its** number before defaulting runs. If that number is not 200m and 200Mi, the Pod
becomes Burstable and the task fails, with a manifest that still looks correct.

```bash
kubectl -n bk0c2d get limitrange                 # one line, settles it
```

Writing both blocks removes the dependency, so write both. It is also fewer keystrokes than the
editor:

```bash
kubectl -n bk0c2d set resources deployment mission-critical \
  --limits=cpu=200m,memory=200Mi --requests=cpu=200m,memory=200Mi
```

`kubectl set resources` is already in my cheat sheet. It works here because a Deployment has a pod
template. Exam 02 task 4 is the opposite case: a bare **Pod** has no template, so `kubectl set` cannot
touch it and YAML is the only way.

**Why Guaranteed answers the wording.** The kubelet evicts BestEffort first, then Burstable, then
Guaranteed. Guaranteed Pods are "guaranteed not to be killed until they exceed their limits or there
are no lower-priority Pods that can be preempted from the Node", which is the task sentence almost
word for word. [labs/07-requesting-and-limiting-resources.md](../labs/07-requesting-and-limiting-resources.md)
already carries the mechanism: going over the **memory request** makes a Pod an eviction candidate
under node pressure. Set request equal to limit and the container cannot pass its request without
also hitting its limit, so it never becomes a candidate.

No lab or exam in this repository names the three QoS classes. That was a real gap, and it is now a
section in the cheat sheet.

One more line from lab 07 that does **not** apply here: "resource updates need recreate,
`kubectl apply --force`". That is true for a bare Pod, where `resources` is immutable. On a
Deployment the rollout replaces the Pods, so a plain apply or `kubectl set resources` is enough.

Source in my notes: [labs/07-requesting-and-limiting-resources.md](../labs/07-requesting-and-limiting-resources.md), [exams/02-configuration.md](../exams/02-configuration.md) task 4, [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Resources and QoS classes

Docs: [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) · [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

</details>

**What to fix next time**

Write both `requests` and `limits`, in one `kubectl set resources` line. Then read
`.status.qosClass` on the **Pod**. A pod template never shows a defaulted request, so checking the
Deployment can only ever look like a failure.

---

## Task 3 - Persisting Data

Namespace `zuc0co` · Skill: swap an `emptyDir` for a PersistentVolumeClaim without breaking the mounts

**Requirement**

- A legacy application runs as a Deployment in Namespace `zuc0co`
- Its Pod has two containers, and the data is lost whenever the Pod is deleted
- Change the Deployment to use a PersistentVolumeClaim
- 2GiB of storage, access mode `ReadWriteOnce`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n zuc0co get pods
kubectl -n zuc0co exec -it legacy-86bd9df457-gdtpq -- cat /metrics/raw.txt

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: legacy-lvc
  namespace: zuc0co
spec:
  accessModes:
  - ReadWriteOnce
  volumeName: legacy-lvc
  resources:
    requests:
      storage: 2Gi
EOF

kubectl -n zuc0co get pvc legacy-lvc -o yaml

kubectl -n zuc0co get deployments legacy -o yaml > legacy-deployment.yaml
vim legacy-deployment.yaml

# Replace volumes with the following
      volumes:
      - name: metrics
        persistentVolumeClaim:
          claimName: legacy-lvc

kubectl apply -f legacy-deployment.yaml
kubectl -n zuc0co get deployment legacy -o jsonpath="{.spec.template.spec.volumes}{'\n'}"
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: the check passed, and the data is still not persisted. `volumeName: legacy-lvc` leaves the claim unable to bind.**

What is right, and worth keeping:

- Reading `/metrics/raw.txt` inside the Pod first. That is how you learn which mount path matters before changing anything, and it is what made task 4 easy.
- 2Gi and `ReadWriteOnce` are the numbers the task gave.
- The volume swap keeps the entry named `metrics`. Both containers mount `metrics`, so renaming it would have broken both `volumeMounts` and the Pod would fail validation. This is the exact shape of [labs/09-persistent-data-with-pods.md](../labs/09-persistent-data-with-pods.md).

**The fault.** `spec.volumeName` on a claim names the **PersistentVolume** to bind to. It is not a
name for the claim, and no PV called `legacy-lvc` exists, because that is the claim's own name. I read
the binding controller to be sure of what happens next: when `volumeName` is set and the named PV is
not found, the controller sets the claim to `Pending` and retries, and the dynamic provisioning branch
runs **only** when `volumeName` is empty. So:

- the claim never binds, and no EBS volume is ever created
- the new Pod stops at `Pending`, with `pod has unbound immediate PersistentVolumeClaims`
- a RollingUpdate does not delete the old Pod until the new one is Ready, so the old `emptyDir` Pod
  keeps serving and the application still looks healthy

That last point is what makes this quiet. Nothing errors, `kubectl apply` succeeds, the application
keeps answering, and the data is still not persisted.

**Why it passed anyway, and why that matters.** The grader read the claim's spec and the Deployment's
volume block. Both are what the task asked for. It did not read the claim's STATUS and it did not
delete the Pod to see whether the file came back. So a green check here says the manifests are right,
not that the storage works. The real exam grades from the API objects the same way, which cuts both
ways: an object that looks right can score, and an object that works can still miss a field the check
reads. Unless a PersistentVolume named `legacy-lvc` happened to exist in that cluster, this claim was
sitting at `Pending` when the check went green, and one command would have shown it.

**Where the habit came from, and why it misfired.** Item 5 of "What to take into the next exam" in
[exams/07-state-persistence.md](../exams/07-state-persistence.md) says to pin a claim to one PV with
`spec.volumeName`. That is correct for **static** provisioning, where the exam had me create a
hostPath PV first and one of several PVs had to be chosen. This task is **dynamic** provisioning, the
lab 09 case, where the cluster creates the volume from the default StorageClass and the claim names no
PV at all. Item 8 of the same list is the rule that should have won: adding fields the task never
asked for is unpaid risk.

**The check that was missing.** One column answers it:

```bash
kubectl -n zuc0co get pvc                  # STATUS must read Bound, not Pending
kubectl -n zuc0co describe pvc legacy-lvc  # the events say why, if it is Pending
kubectl get sc                             # is a class marked (default)?
```

If no StorageClass is marked `(default)`, a claim with no `storageClassName` also stays Pending, and
then a static PV really is needed. Lab 09 ran on `gp2`, so that cluster had one.

**Two faster and safer moves:**

```bash
kubectl -n zuc0co edit deployment legacy    # no get, no file, no apply
```

Do not reach for `kubectl patch` on this one. `spec.volumes` merges on the key `name`, so a patch adds
`persistentVolumeClaim` **next to** the existing `emptyDir` and the API rejects the Pod with
"may not specify more than 1 volume type". Patching needs the old field nulled out:

```bash
kubectl -n zuc0co patch deployment legacy --patch \
  '{"spec":{"template":{"spec":{"volumes":[{"name":"metrics","emptyDir":null,"persistentVolumeClaim":{"claimName":"legacy-pvc"}}]}}}}'
```

Same merge trap as challenge 01 task 2 and exam 06 task 4, in a third shape. `kubectl edit` sidesteps
all of it.

**One risk that survives the fix.** `ReadWriteOnce` is one **node**. With the default RollingUpdate,
the replacement Pod can be scheduled on another node, fail to attach the volume, and stall while the
old Pod is kept alive waiting for it. `strategy: Recreate` avoids the standoff for a single-replica
stateful Deployment. Lab 13 uses a StatefulSet with `volumeClaimTemplates` for the same reason: one
claim per Pod, created in order.

Small thing: `legacy-lvc` looks like a typo for `pvc`. It costs nothing, but the name gets retyped
inside the volume block, so a name that reads right is a name you get right.

Source in my notes: [labs/09-persistent-data-with-pods.md](../labs/09-persistent-data-with-pods.md), [exams/07-state-persistence.md](../exams/07-state-persistence.md) tasks 1 and 2, [labs/13-deploy-stateful-application-mysql.md](../labs/13-deploy-stateful-application-mysql.md), [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Volumes and PersistentVolumes

Docs: [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) · [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) · [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)

</details>

**What to fix next time**

Leave `volumeName` out unless a PersistentVolume with that exact name already exists. After creating
any claim, run `kubectl get pvc` and read the STATUS column before touching the workload.

---

## Task 4 - Multi-Container Pattern

Namespace `zuc0co` · Skill: name the pattern from the direction the data flows

**Requirement**

- Write the name of the multi-container Pod design pattern used by the Pod in task 3
- Save it to `/home/ubuntu/mcpod`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n zuc0co get pods
kubectl -n zuc0co get pods legacy-86bd9df457-gdtpq -o yaml
```

The two containers, from that output. Trimmed to the parts that decide the answer.

```yaml
spec:
  containers:
  - name: app
    image: alpine:3.9.2
    command: [/bin/sh, -c]
    args:
    - while true; do date > /metrics/raw.txt; top -n 1 -b >> /metrics/raw.txt; sleep 5; done
    volumeMounts:
    - mountPath: /metrics
      name: metrics
  - name: adapter
    image: httpd:2.4.38-alpine
    command: [/bin/sh, -c]
    args:
    - while true; do date=$(head -1 /metrics/raw.txt); memory=...; cpu=...;
      echo "{\"date\":\"$date\",\"memory\":\"$memory\",\"cpu\":\"$cpu\"}" >> /metrics/adapted.json;
      sleep 5; done
    volumeMounts:
    - mountPath: /metrics
      name: metrics
  volumes:
  - emptyDir: {}
    name: metrics
```

```bash
echo "adapter" > /home/ubuntu/mcpod
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: correct, and reached the right way.**

The `app` container writes raw output to `/metrics/raw.txt`. The second container reads that file and
writes `/metrics/adapted.json` in a fixed JSON shape that a metric aggregation system can consume.
Reshaping one container's output so an outside reader can use it is the **adapter** pattern.

Reading the Pod YAML instead of guessing is the method that scales. The tell is the direction of the
data, not the container's name. A container called `adapter` is a hint and nothing more, and the same
name over a container that proxied outgoing calls would make the answer ambassador.

**The three patterns.** My notes name only the sidecar, so this is the gap this task exposed:

| Pattern | What the helper container does | Example |
|---------|-------------------------------|---------|
| Sidecar | adds a function the main container lacks | ships log files, as in lab 05 with Fluentd |
| Adapter | reformats the main container's output for an outside consumer | raw metrics to a scraper's format, this task |
| Ambassador | proxies the main container's outbound connections | a local proxy that finds the right database |

All three are helper containers in one Pod sharing a volume or `localhost`. When a task asks for the
**pattern**, answer with the specific one, not with "sidecar".

Two small things that were right: `>` and not `>>`, so a re-run does not append a second line; and one
word with no trailing text, which is what a file check compares. The quotes around `adapter` are
optional.

Faster than a full Pod dump, when the container names are all that is needed:

```bash
kubectl -n zuc0co get deployment legacy -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
cat /home/ubuntu/mcpod                        # read back what the check reads
```

Source in my notes: [labs/05-container-logging-and-sidecar.md](../labs/05-container-logging-and-sidecar.md), [exams/03-multi-container-pods.md](../exams/03-multi-container-pods.md), [labs/12-ephemeral-volume-types.md](../labs/12-ephemeral-volume-types.md)

Docs: [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) · [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) · [Composite Container Patterns](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)

</details>

**What to fix next time**

Nothing. Keep this one as it is.

---

## Pattern check

**Doing well**

1. Reading the object before changing it. `cat` inside the Pod on task 3, the full Pod YAML on task 4. That is what made task 4 a five-second answer instead of a guess between three names.
2. JSONPath on the one field, with `{'\n'}` to end the line, after three of the four tasks. That habit is from the fix list of challenge 01 and it stuck.
3. Current field names. `serviceAccountName`, not the deprecated `serviceAccount` that my own lab 11 note still uses.
4. Getting the QoS rule right when no lab or exam in this repository covers QoS classes at all.

**To fix**

1. **Check the object the rule lives on.** QoS is computed on the Pod, so a pod template can never show it. This is the same shape as challenge 01: the change was right, the proof looked at the wrong place.
2. **`volumeName` is a pointer to an existing PersistentVolume.** After creating any claim, read the STATUS column. `Pending` is silent, a RollingUpdate hides it by keeping the old Pod alive, and a spec-reading grader passes it anyway.
3. **Two of my own notes collide, so rank them.** "Pin a claim with `volumeName`" needs a reason. "Add nothing the task did not ask for" is the default.
4. **The `vim` round trip.** `get -o yaml > file`, edit, apply is four steps. `kubectl set resources` and `kubectl edit` are one, and both were already in my cheat sheet.

**One habit that would have caught both problems, which a passing score did not**

After every change, read the field back from the object that owns it: `.status.qosClass` on the Pod,
the STATUS column on the PVC, `.spec.template.spec.serviceAccountName` on the Deployment.
