# Exam 01 - Core Concepts

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Core Concepts  
> Checks: 6  
> Solution guide: https://app.qa.com/resource/ckad-practice-exam-core-concepts-solution-guide/  
> Topics: pods, namespaces, labels, restartPolicy, jsonpath, dry-run, kubectl-run, kubectl-label, kubectl-edit, kubectl-explain, terminationGracePeriodSeconds, env

## Format

Six checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

The shortest correct command for each check. Read this before an attempt, then close it.

```bash
# 1 - Pod basic in cre, restart OnFailure, TCP port 80
kubectl run -n cre basic --image=nginx:stable-alpine-perl --restart=OnFailure --port=80

# 2 - Namespace workers, Pod worker with three labels, restart Never
kubectl create ns workers
kubectl run -n workers worker --image=busybox --restart=Never \
  --labels="company=acme,speed=fast,type=async" \
  -- /bin/sh -c "echo working... && sleep 3600"

# 3 - change a label on a running Pod, no restart
kubectl label pod -n ca200 compiler language=python --overwrite

# 4 - Pod IP with JSONPath, save the command to a file
kubectl get pod -n ca300 ip-podzoid -o jsonpath='{.status.podIP}'
echo "kubectl get pod -n ca300 ip-podzoid -o jsonpath='{.status.podIP}'" > /home/ubuntu/podip.sh

# 5 - generate a manifest only, do not create the Pod
kubectl run -n core-system borg1 --image=busybox --restart=Always \
  --labels="platform=prod" --env system=borg \
  --dry-run=client -o yaml \
  -- /bin/sh -c "echo borg.running... && sleep 3600" > /home/ubuntu/pod.yaml

# 6 - field with no flag, so generate then edit
kubectl run web-zeroshutdown -n sys2 --image=nginx --restart=Never \
  --dry-run=client -o yaml > pod.yaml
# add terminationGracePeriodSeconds: 0 directly under spec, then
kubectl apply -f pod.yaml
```

## Exam strategy notes

Habits that apply to every task, not just this exam.

- **Imperative first.** `kubectl run` with flags beats hand-written YAML. Reach for YAML only when a field has no flag.
- **Tab completion is on.** The lab host runs `source <(kubectl completion bash)`. Use it to discover flags instead of reading help.
- **`kubectl run --help`** has examples you can combine into the command you need.
- **Namespace:** `-n <ns>` on each command is explicit and safe. `kubectl config set-context --current --namespace=<ns>` saves typing but is easy to forget between tasks.
- **Read the verb.** "Generate a manifest" is not "create the Pod". Extra work costs time and earns nothing.
- **Vim matters.** Practise `i`, Escape, `:wq` before the exam. In the real exam you can draft in the notepad and paste into vim.

---

## Task 1 - Create and configure a basic Pod

Namespace `cre` · Skill: `kubectl run` flags

**Requirement**

- Pod named `basic` in the `cre` Namespace
- Image `nginx:stable-alpine-perl`, one container
- Restart only `OnFailure`
- Port 80 open to TCP

<details><summary><b>My solution</b></summary>

```bash
kubectl config set-context --current --namespace=cre

kubectl apply --namespace cre -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: basic
spec:
  containers:
  - name: basic
    image: nginx:stable-alpine-perl
    ports:
    - containerPort: 80
  restartPolicy: OnFailure
EOF
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl run -n cre --image=nginx:stable-alpine-perl --restart=OnFailure --port=80 basic
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- Every field here has a `kubectl run` flag, so no manifest is needed.
- `--port` means TCP unless you say otherwise.
- Three ways to get a starting manifest when you do need one: copy an example from the docs, use `kubectl run --dry-run=client -o yaml` (the `--image` flag is required), or dump an existing Pod and delete fields. The third is slow, Pod output runs to hundreds of lines.
- `kubectl explain pod.spec` tells you which fields exist and where they sit.
- In the real exam, draft the manifest in the notepad, then `vim manifest.yaml`, press `i`, paste, press Escape, type `:wq`.

Review: [labs/00-introduction-to-kubernetes-playground.md](../labs/00-introduction-to-kubernetes-playground.md), [labs/01-pod-definition-basics.md](../labs/01-pod-definition-basics.md)

Docs: [Configure Pods and Containers](https://kubernetes.io/docs/tasks/configure-pod-container/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

One line replaces 14 lines of YAML. `--restart=OnFailure` sets `spec.restartPolicy`, `--port=80` sets the container port.

---

## Task 2 - Create a Namespace and launch a Pod with labels

Namespace `workers` · Skill: `--labels`, container command

**Requirement**

Create Namespace `workers`, and in it a Pod:

- Pod named `worker`
- Image `busybox`, one container
- Restart `Never`
- Command `/bin/sh -c "echo working... && sleep 3600"`
- Labels `company=acme`, `speed=fast`, `type=async`

<details><summary><b>My solution</b></summary>

```bash
kubectl create namespace workers

kubectl config set-context --current --namespace=workers

kubectl apply --namespace=workers -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: worker
  labels:
    company: acme
    speed: fast
    type: async
spec:
  containers:
  - name: worker
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "echo working... && sleep 3600"]
  restartPolicy: Never
EOF
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl create ns workers
kubectl run -n workers worker --image=busybox --labels="company=acme,speed=fast,type=async" -- /bin/sh -c "echo working... && sleep 3600"
```

This command is missing `--restart=Never`. See the difference notes below.

</details>

<details><summary><b>Technique and what to review</b></summary>

- `--labels` takes a comma-separated list, so three labels cost one flag.
- Everything after `--` becomes the container `args`, which replaces the image CMD. `busybox` has no ENTRYPOINT, so plain args run as the full command. For an image with an ENTRYPOINT you need `--command` to replace it instead.
- `kubectl create ns` is shorter than `kubectl create namespace`.

Review: [labs/01-pod-definition-basics.md](../labs/01-pod-definition-basics.md), [labs/02-pod-labels-selectors-annotations.md](../labs/02-pod-labels-selectors-annotations.md)

Docs: [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) · [Configure Pods and Containers](https://kubernetes.io/docs/tasks/configure-pod-container/)

</details>

**Difference that matters**

The official one-liner leaves out `--restart=Never`, and `kubectl run` defaults to `Always`. The task asks for `Never`, so the command should be:

```bash
kubectl run -n workers worker --image=busybox --restart=Never \
  --labels="company=acme,speed=fast,type=async" \
  -- /bin/sh -c "echo working... && sleep 3600"
```

---

## Task 3 - Update the label on a running Pod

Namespace `ca200` · Skill: `kubectl label --overwrite`

**Requirement**

Namespace `ca200` has a running Pod named `compiler`. Without restarting it, change its `language` label from `java` to `python`.

<details><summary><b>My solution</b></summary>

```bash
kubectl config set-context --current --namespace=ca200

kubectl get pod compiler -o yaml

kubectl label pod compiler language=python --overwrite=true

kubectl get pod compiler -o yaml
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
# Edit and save the pod - this updates it without a restart
kubectl edit pod -n ca200 compiler
```

`kubectl edit` opens the manifest in vim. Change the label, then press Escape, type `:x!`, press Enter to save and apply.

</details>

<details><summary><b>Technique and what to review</b></summary>

- `--overwrite` is required when the label key already exists. Without it kubectl refuses the change.
- Both approaches touch metadata only, so the Pod keeps running. Changing `spec.containers[*].image` would restart it, changing labels does not.
- `kubectl edit` is the fallback for any field that has no dedicated command.

Review: [labs/02-pod-labels-selectors-annotations.md](../labs/02-pod-labels-selectors-annotations.md)

Docs: [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

Mine is the faster answer. `kubectl label` is one command with no editor, so there is no vim step to get wrong under time pressure.

---

## Task 4 - Get the Pod IP address using JSONPath

Namespace `ca300` · Skill: `-o jsonpath`

**Requirement**

Find the IP of Pod `ip-podzoid` in Namespace `ca300` using JSONPath. Save the **command**, not the result, to `/home/ubuntu/podip.sh`.

<details><summary><b>My solution</b></summary>

```bash
kubectl config set-context --current --namespace=ca300

kubectl get pod ip-podzoid -o jsonpath='{$.status.podIPs[].ip}'

echo "kubectl --namespace=ca300 get pod ip-podzoid -o jsonpath='{$.status.podIPs[].ip}'" >> /home/ubuntu/podip.sh

cat /home/ubuntu/podip.sh
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
# Step 1 - examine the json structure, the pod IP sits in .status.podIP
kubectl get pods -n ca300 ip-podzoid -o json
# Step 2 - build the jsonpath expression
kubectl get pods -n ca300 ip-podzoid -o jsonpath={.status.podIP}
# Step 3 - write the working command to the file
echo "kubectl get pods -n ca300 ip-podzoid -o jsonpath={.status.podIP}" > /home/ubuntu/podip.sh
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- Print the object as JSON first, then write the path. Guessing field names wastes more time than one `-o json` call.
- `.status.podIP` is the single address. `.status.podIPs[].ip` is the list form, added for dual-stack. Both work here.
- JSONPath works on any resource, so the same habit answers "give me just the load balancer hostname" or "just the image".
- Quote the expression in single quotes so the shell leaves `{`, `}`, and `$` alone.

Review: no lab teaches JSONPath. It appears as a copy-paste idiom in [labs/15-creating-ingress-rules.md](../labs/15-creating-ingress-rules.md) and [labs/16-deployment-strategies-canary-blue-green.md](../labs/16-deployment-strategies-canary-blue-green.md).

Docs: [JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

- I used `>>` (append). Run the command twice and the file holds two lines, which can fail the check. Use `>` when a task says save a command to a file.
- `.status.podIP` is shorter than `.status.podIPs[].ip` and returns the same value.

---

## Task 5 - Generate a Pod YAML manifest file

Namespace `core-system` · Skill: `--dry-run=client -o yaml`

**Requirement**

Generate a Pod manifest with exactly these values and save it to `/home/ubuntu/pod.yaml`:

- Pod name `borg1`
- Namespace `core-system`
- Image `busybox`
- Command `/bin/sh -c "echo borg.running... && sleep 3600"`
- Restart policy `Always`
- Label `platform=prod`
- Environment variable `system=borg`

<details><summary><b>My solution</b></summary>

```bash
cat <<EOF > /home/ubuntu/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: borg1
  namespace: core-system
  labels:
    platform: prod
spec:
  containers:
  - name: borg1
    image: busybox
    env:
    - name: system
      value: borg
    command: ["/bin/sh"]
    args: ["-c", "echo borg.running... && sleep 3600"]
  restartPolicy: Always
EOF
```

I then also created the Namespace, applied the manifest, and checked the logs.

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl run -n core-system borg1 --image=busybox --restart=Always --labels="platform=prod" --env system=borg -o yaml --dry-run=client -- /bin/sh -c "echo borg.running... && sleep 3600" > /home/ubuntu/pod.yaml
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `--dry-run=client -o yaml` builds the manifest without contacting the cluster, so the Namespace does not need to exist.
- `--env` sets one environment variable and can be repeated for more.
- Redirect straight to the target path. There is no need to open an editor when every value has a flag.

Review: [labs/01-pod-definition-basics.md](../labs/01-pod-definition-basics.md)

Docs: [Managing Objects using Imperative Commands](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

- One `kubectl run --dry-run=client -o yaml` line replaces the whole hand-written manifest.
- The task asked only to generate the file. Creating the Namespace, applying the Pod, and reading logs was extra work that earns no points.

---

## Task 6 - Launch a Pod and configure its termination shutdown time

Namespace `sys2` · Skill: field with no imperative flag

**Requirement**

Launch a web server Pod in Namespace `sys2`:

- Pod name `web-zeroshutdown`
- Image `nginx`
- Restart policy `Never`
- Configure `terminationGracePeriodSeconds` so the Pod terminates immediately

<details><summary><b>My solution</b></summary>

```bash
kubectl config set-context --current --namespace=sys2

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-zeroshutdown
  namespace: sys2
spec:
  containers:
  - name: web-zeroshutdown
    image: nginx
  restartPolicy: Never
  terminationGracePeriodSeconds: 0
EOF
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
# Step 1 - generate the manifest
kubectl run web-zeroshutdown -n sys2 --image=nginx --restart=Never --port=80 -o yaml --dry-run=client > pod-zeroshutdown.yaml
# Step 2 - check where the field belongs
kubectl explain pod.spec
# Step 3 - edit the file in vim and add terminationGracePeriodSeconds: 0 under spec
# Step 4 - create the Pod
kubectl apply -f pod-zeroshutdown.yaml
```

The edited manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web-zeroshutdown
  name: web-zeroshutdown
  namespace: sys2
spec:
  terminationGracePeriodSeconds: 0
  containers:
  - image: nginx
    name: web-zeroshutdown
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `terminationGracePeriodSeconds` has no `kubectl run` flag. This is the pattern to recognise: generate with `--dry-run=client -o yaml`, edit the one missing field, apply.
- `kubectl explain pod.spec` confirms the field sits under `spec`, not under the container. Getting the nesting wrong is the usual mistake.
- `0` means the kubelet kills the container immediately, with no grace period for it to shut down cleanly. The default is 30 seconds.

Review: [labs/01-pod-definition-basics.md](../labs/01-pod-definition-basics.md)

Docs: [Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) · [kubectl explain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#explain)

</details>

**Differences that matter**

- Mine typed the small manifest directly, which is fine and arguably quicker than generate, open vim, edit, save, apply.
- The official command adds `--port=80`, which the task did not ask for. Harmless, but extra.

---

## What to take into the next exam

1. Ask first: does every required field have a `kubectl run` flag? If yes, one line. If no, `--dry-run=client -o yaml` and edit.
2. `--restart=`, `--port=`, `--labels=`, `--env` cover most Core Concepts tasks.
3. Use `kubectl label --overwrite` instead of opening an editor.
4. Read the task verb. "Generate a manifest" does not mean create the object.
5. Use `>` when a task says save a command to a file.
6. Fields with no imperative flag so far: `terminationGracePeriodSeconds`.
