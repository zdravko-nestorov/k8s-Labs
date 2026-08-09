# Exam 05 - Pod Design

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Pod Design  
> Checks: 6  
> Solution guide: https://cloudacademy.com/resource/ckad-practice-exam-pod-design-solution-guide-14022022224444/  
> Topics: deployments, replicas, scale, set-image, rollout-history, change-cause, annotations, labels, label-selector, set-based-selector, hpa, autoscale, cronjob, schedule, jsonpath, sort-by, custom-columns, kubectl-create

## Format

Six checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

```bash
# 1 - create, scale, update image: three one-liners, no YAML
kubectl -n zap create deployment webapp --image=nginx:1.17.8 --replicas=2
kubectl -n zap scale deployment webapp --replicas=4
kubectl -n zap set image deployment webapp nginx=nginx:1.19.0

# 2 - label every Pod that matches a selector
kubectl -n gzz label pods --selector env=prod app=cloudacademy

# 3 - update the image and record why
kubectl -n fre set image deployment cloudforce nginx=nginx:1.19.0-perl
kubectl -n fre annotate deployment cloudforce \
  kubernetes.io/change-cause="set image to nginx:1.19.0-perl" --overwrite=true

# 4 - the HPA has its own command
kubectl -n xx1 autoscale deployment eclipse --min=2 --max=4 --cpu-percent=65

# 5 - CronJob has flags for image and schedule
kubectl -n saas create cronjob matrix --image=radial/busyboxplus:curl \
  --schedule='*/10 * * * *' -- curl www.google.com

# 6 - filter, sort, print names only
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' \
  --sort-by=.status.podIP \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' > /home/ubuntu/pod001
```

## Exam strategy notes

- **This whole domain is imperative.** `kubectl create deployment`, `scale`, `set image`, `autoscale`, and `create cronjob` cover five of the six checks with one line each. Writing YAML here is the slow path.
- **`kubectl create` builds more than Pods.** It handles deployment, cronjob, job, service, configmap, secret, serviceaccount, role, and more. Learn what it can make.
- **`--record` is gone.** Removed by kubectl v1.36.3. The `kubernetes.io/change-cause` annotation is the only way to fill the rollout history.
- **Add `--overwrite=true` when annotating.** Without it, `kubectl annotate` refuses to change a key that already exists.
- **`--selector` works across verbs**, not just `get`. `kubectl label pods --selector env=prod` relabels a whole group in one command.
- **Build long queries in stages.** Selector first, confirm with `--show-labels`, then add `--sort-by`, then change the output format. Do not type it all at once.
- **`kubectl explain` finds a JSONPath.** `kubectl explain pod.status` shows `podIP`, which becomes `.status.podIP`.

---

## Task 1 - Create and manage Deployments

Namespace `zap` · Skill: `create deployment`, `scale`, `set image`

**Requirement**

- Create a Deployment named `webapp` in Namespace `zap`
- Image `nginx:1.17.8`, 2 replicas to start
- Scale it from 2 to 4
- Update it to `nginx:1.19.0`

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: zap
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.8
EOF

kubectl -n zap get deployment

kubectl -n zap scale deployment/webapp --replicas=4
kubectl -n zap rollout status deployment/webapp
kubectl -n zap annotate deployment/webapp kubernetes.io/change-cause="kubectl -n zap scale deployment/webapp --replicas=4"
kubectl -n zap rollout history deployment/webapp

kubectl -n zap set image deployment/webapp nginx=nginx:1.19.0
kubectl -n zap rollout status deployment/webapp
kubectl -n zap annotate deployment/webapp kubernetes.io/change-cause="kubectl -n zap set image deployment/webapp nginx=nginx:1.16.1"
kubectl -n zap rollout history deployment/webapp
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n zap create deployment --image=nginx:1.17.8 --replicas=2 webapp
kubectl -n zap scale deployment --replicas=4 webapp
kubectl -n zap set image deployment webapp nginx=nginx:1.19.0
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl create deployment` takes `--image` and `--replicas`, so the whole first step is one line. Confirmed on kubectl v1.36.3.
- The container is named after the image, so `nginx:1.17.8` gives a container called `nginx`. That is the name `set image deployment webapp nginx=...` refers to.
- `deployment webapp` and `deployment/webapp` mean the same thing. Help output uses the slash form.
- For a manifest instead, add `--dry-run=client -o yaml > webapp.yaml`, then edit and `kubectl apply -f`.
- `kubectl edit deployment webapp` combines editing and applying, which can cover both the scale and the image change in one step.
- Scaling does **not** create a rollout revision. Only pod template changes do, so the image update is a new revision and the scale is not.

Review: [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md)

Docs: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

- Three one-liners replace a 20-line manifest. `kubectl create deployment` is the command to remember here.
- The task never asked for a change cause on this check, so both `annotate` calls were unpaid work. Only check 3 asks for it.
- The second annotation records `nginx:1.16.1` while the command actually set `nginx:1.19.0`. Nothing scores it here, but on check 3 the history **is** the answer, so a wrong message would be the wrong answer.

---

## Task 2 - Create Pod labels

Namespace `gzz` · Skill: `kubectl label --selector`

**Requirement**

- Add the label `app=cloudacademy` to every Pod in Namespace `gzz` that already has `env=prod`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n gzz get pods -l env=prod --show-labels=true
kubectl -n gzz label pods -l env=prod app=cloudacademy
kubectl -n gzz get pods -l env=prod --show-labels=true
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n gzz label pods --selector env=prod app=cloudacademy
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `-l` and `--selector` are the same flag. This exact pattern appears in the Kubernetes docs as `kubectl label pods -l app=nginx tier=fe`.
- No `--overwrite` needed here, because `app` is a new key on those Pods. It is only required when replacing an existing key.
- Checking with `--show-labels` before and after costs a few seconds and confirms you hit the right Pods. Worth it when a task changes many objects at once.
- The slow alternative is `kubectl edit` on each Pod. Avoid it.

Review: [labs/02-pod-labels-selectors-annotations.md](../labs/02-pod-labels-selectors-annotations.md)

Docs: [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

None, the command is the same. The two `--show-labels` calls are verification, not extra work.

---

## Task 3 - Rollback Deployment, record the change

Namespace `fre` · Skill: `change-cause` annotation

**Requirement**

- Update the `nginx` container in Deployment `cloudforce`, Namespace `fre`, to `nginx:1.19.0-perl`
- Make sure the command used is recorded in the rollout history

<details><summary><b>My solution</b></summary>

```bash
kubectl -n fre rollout history deployment/cloudforce
kubectl -n fre get deployments cloudforce
kubectl -n fre set image deployment/cloudforce nginx=nginx:1.19.0-perl
kubectl -n fre annotate deployment/cloudforce kubernetes.io/change-cause="kubectl -n fre set image deployment/cloudforce nginx=nginx:1.19.0-perl"
kubectl -n fre rollout history deployment/cloudforce
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n fre set image deployment cloudforce nginx=nginx:1.19.0-perl
kubectl -n fre annotate deployment cloudforce kubernetes.io/change-cause="set image to nginx:1.19.0-perl" --overwrite=true
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- The rollout history column CHANGE-CAUSE is filled from the `kubernetes.io/change-cause` annotation. Nothing writes it for you.
- The old `--record=true` flag did this automatically. It is **removed**: `kubectl set image --help` on v1.36.3 lists no `--record`. Annotating is now the only way.
- `set image` then `annotate` is the right order. The controller copies the annotation onto the ReplicaSet for that revision.
- `nginx=nginx:1.19.0-perl` reads as container name equals new image. Get the container name from `kubectl describe deployment` if you are unsure.
- Check with `kubectl rollout history deployment cloudforce`, and roll back with `kubectl rollout undo deployment cloudforce --to-revision=N`.

Review: [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md)

Docs: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

My annotate call is missing `--overwrite=true`. If the Deployment already carries a `change-cause` from an earlier rollout, `kubectl annotate` refuses to replace it and errors instead. Adding the flag makes the command safe either way, so always include it here.

---

## Task 4 - Configure Pod autoscaling

Namespace `xx1` · Skill: `kubectl autoscale`

**Requirement**

- Deployment `eclipse` in Namespace `xx1` runs 2 replicas
- Autoscale it on CPU: minimum 2, maximum 4, target 65%

<details><summary><b>My solution</b></summary>

```bash
kubectl -n xx1 get hpa
kubectl -n xx1 autoscale deployment/eclipse --min=2 --max=4 --cpu-percent=65 --dry-run=client -o yaml

kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: eclipse
  namespace: xx1
spec:
  minReplicas: 2
  maxReplicas: 4
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 65
        type: Utilization
    type: Resource
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: eclipse
  namespace: xx1
EOF

kubectl -n xx1 get hpa
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n xx1 autoscale deployment --min=2 --max=4 --cpu-percent=65 eclipse
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl autoscale` creates the HorizontalPodAutoscaler. It is easy to forget because it is a top-level verb, not part of `create`. Tab completion after `kubectl` lists it.
- Check the result with `kubectl -n xx1 get hpa`. The TARGETS column shows current usage against the target.
- The HPA needs metrics-server, and it needs CPU **requests** set on the containers. Utilisation is a percentage of the request, so with no request there is nothing to take a percentage of.
- `--cpu-percent=65` maps to `averageUtilization: 65` in the `autoscaling/v2` API.

Review: [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md), [labs/07-requesting-and-limiting-resources.md](../labs/07-requesting-and-limiting-resources.md)

Docs: [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) · [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

</details>

**Differences that matter**

- I had the answer and then did the work twice. The first command with `--dry-run=client -o yaml` was already correct. Removing `--dry-run=client -o yaml` creates the HPA. Instead I read the generated YAML, retyped it, and applied it.
- The retyped manifest also has a bug. The trailing `namespace: xx1` sits under `spec`, not under `metadata`. `spec.namespace` is not a field of HorizontalPodAutoscaler, and kubectl validates fields by default, so the apply is rejected as an unknown field. The `metadata.namespace` above it was already doing the job.
- Lesson: when a dry run prints what you want, drop the dry run. Do not transcribe it.

---

## Task 5 - Create a CronJob

Namespace `saas` · Skill: `kubectl create cronjob`

**Requirement**

- CronJob named `matrix` in Namespace `saas`
- Image `radial/busyboxplus:curl`, schedule `*/10 * * * *`
- Runs `curl www.google.com`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n saas get cronjob matrix

kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: matrix
  namespace: saas
spec:
  schedule: "*/10 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: matrix
            image: radial/busyboxplus:curl
            imagePullPolicy: IfNotPresent
            command: ["curl", "www.google.com"]
          restartPolicy: Never
EOF

kubectl -n saas get cronjob matrix
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n saas create cronjob --image=radial/busyboxplus:curl --schedule='*/10 * * * *' matrix -- curl www.google.com
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl create cronjob` takes `--image`, `--schedule`, and `--restart`. Everything after `--` becomes the command. Confirmed on kubectl v1.36.3.
- Quote the schedule. `*/10 * * * *` is full of characters the shell would otherwise expand.
- The nesting is the reason YAML is painful here: CronJob to `jobTemplate` to `template` to `spec` to `containers`, four levels before you reach the image.
- If a task words the schedule in English, for example "every hour", the cron syntax is in the CronJob documentation.
- Trigger one immediately for testing with `kubectl create job test --from=cronjob/matrix`.

Review: [labs/04-jobs-and-cronjobs.md](../labs/04-jobs-and-cronjobs.md)

Docs: [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) · [Running Automated Tasks with a CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

</details>

**Difference that matters**

One line against roughly 20. The CronJob manifest is the most deeply nested object in CKAD, which makes it the one where the imperative command saves the most time.

---

## Task 6 - Filter and sort Pods

Namespace `rep` · Skill: set-based selector, `--sort-by`, JSONPath range

**Requirement**

- List Pods in Namespace `rep` whose `colour` label is `orange`, `red`, or `yellow`
- Output only the Pod names, nothing else
- Order them by the Pod IP address
- Save to `/home/ubuntu/pod001`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n rep get pods -l 'colour in (orange, red, yellow)' --sort-by='.status.podIP' -o=jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' > /home/ubuntu/pod001
cat /home/ubuntu/pod001
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' > /home/ubuntu/pod001
```

The guide recommends building it in stages:

```bash
# 1 - get the selector right
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --show-labels
# 2 - add the sort, with -o wide to see the IPs
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP -o wide
# 3 - names on one line
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP -o jsonpath='{.items[*].metadata.name}'
# 4 - one name per line with range
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

Alternatives that avoid JSONPath:

```bash
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP -o custom-columns="NAME:.metadata.name"
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP | cut -d' ' -f1 | tail +2
kubectl -n rep get pods --selector 'colour in (orange,red,yellow)' --sort-by=.status.podIP | grep -o -e "pod[0-9]*"
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `in (a,b,c)` is a set-based selector, the closest thing to OR. There is no `||` operator in label selectors.
- Spaces after the commas are fine. The Kubernetes docs use `kubectl get pods -l 'environment in (production, qa)'`.
- Quote the selector. The parentheses would otherwise be interpreted by the shell.
- `--sort-by` takes a JSONPath with no braces. To find one, run `kubectl get pod <name> -o yaml` and trace the field, or walk `kubectl explain pod.status`. The IP is assigned by the cluster, so it lives in `status`, not `spec`.
- Sorting is lexicographic here, so IPs sort as strings. That is what the task wants, so do not overthink it.
- `{range .items[*]}...{end}` is the JSONPath loop. Without it, `{.items[*].metadata.name}` prints every name on one line separated by spaces.
- `-o custom-columns="NAME:.metadata.name"` is easier to remember, but it prints a header row you then have to strip.

Review: [labs/02-pod-labels-selectors-annotations.md](../labs/02-pod-labels-selectors-annotations.md), [exams/01-core-concepts.md](01-core-concepts.md) for JSONPath basics

Docs: [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

</details>

**Difference that matters**

None. My command matches the official answer, spaces included. This is the hardest check in the exam and it came out right first time.

---

## What to take into the next exam

1. Pod Design is an imperative domain. `create deployment`, `scale`, `set image`, `autoscale`, `create cronjob` each replace a manifest.
2. `kubectl create deployment <name> --image= --replicas=` covers creation in one line.
3. `kubectl autoscale deployment <name> --min= --max= --cpu-percent=` creates the HPA. It is a top-level verb, not part of `create`.
4. `kubectl create cronjob <name> --image= --schedule='...' -- <command>`. Quote the schedule.
5. `--record` no longer exists. Fill the rollout history with `kubectl annotate ... kubernetes.io/change-cause="..." --overwrite=true`.
6. When a dry run prints what you want, drop `--dry-run` and run it. Do not retype the output.
7. `--selector` works with `label`, `delete`, `describe`, and `logs`, not only `get`.
8. Set-based selector for OR: `-l 'key in (a,b,c)'`.
9. One name per line: `-o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'`.
