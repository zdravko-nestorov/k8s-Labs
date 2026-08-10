# Challenge 01 - Kubernetes Certification Challenge

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: Lab Challenge, no official solution  
> Checks: 4  
> Result: 4 of 4 passed  
> Reviewed against: `labs/`, `exams/`, `exams/CHEATSHEET.md`  
> Topics: deployments, replicas, revisionHistoryLimit, kubectl-create, kubectl-patch, services, endpoints, label-selector, set-selector, troubleshooting, kubectl-top, metrics-server, secrets, secretKeyRef, env, kubectl-exec, printenv

## Format

Four validation checks, no solution guide. Nothing here is an official answer. The review blocks
compare my commands against my own labs and exams, so a wrong note in those files becomes a wrong
note here.

Solutions are collapsed. Open one only after you have tried the task.

## Score card

| Task | Result | What it drills |
|------|--------|----------------|
| 1 - Deployment with a revision history limit | Correct | imperative create, then patch the one field with no flag |
| 2 - Service does not reach its Pods | Correct, not verified | reading endpoints, and knowing when a patch on a selector is safe |
| 3 - Highest CPU Pod to a file | Correct | `kubectl top --sort-by`, cut to one field |
| 4 - Secret exposed as an env var | Correct, verification broken | `secretKeyRef`, and how to read an env var back |

**All four checks passed.** Every solution was right, so the entire gap is in **proving** the work.
Task 2 was left uncertain because the test command was not to hand, and task 4 used a verification
that cannot print anything. Both of those commands were already written in my notes.

## Best commands I know now

```bash
# 1 - no flag for revisionHistoryLimit, so create then patch
kubectl -n cal create deployment chk1 --image=nginx:1.15.12-alpine --replicas=3
kubectl -n cal patch deployment chk1 --patch='{"spec":{"revisionHistoryLimit":50}}'
kubectl -n cal get deployment chk1 -o jsonpath='{.spec.revisionHistoryLimit}'

# 2 - read the endpoints first, then replace the selector rather than patching it
kubectl -n pwn get ep sitelb
kubectl -n pwn get deployment site -o jsonpath='{.spec.template.metadata.labels}'
kubectl -n pwn set selector service sitelb 'app=site'
kubectl -n pwn get ep sitelb
kubectl -n pwn run client --image=curlimages/curl -it --rm --restart=Never -- curl http://sitelb:80

# 3 - one pipe, no loop
kubectl -n zz8 top pod --sort-by=cpu --no-headers | head -n1 | awk '{print $1}' > /home/ubuntu/hcp001

# 4 - secret from a literal, then reference the key by name in the Pod
kubectl -n sjq create secret generic xh8jqk7z --from-literal=tkn=hy8szK2iu
kubectl -n sjq exec server -- printenv SECRET_TKN
```

---

## Task 1 - Created specified Deployment

Namespace `cal` · Skill: imperative create, then patch the field that has no flag

**Requirement**

- Deployment named `chk1` in Namespace `cal`
- Image `nginx:1.15.12-alpine`
- 3 replicas
- `revisionHistoryLimit` set to 50

<details><summary><b>My solution</b></summary>

```bash
kubectl -n cal create deployment chk1 --image=nginx:1.15.12-alpine --replicas=3
kubectl -n cal patch deployment chk1 --patch='{"spec":{"revisionHistoryLimit":50}}'
kubectl -n cal get deployment chk1 -o yaml

kubectl -n cal get pod,deployment --show-labels
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: correct, and the fastest route available.**

- `kubectl create deployment --image --replicas` is exactly the entry in the cheat sheet Deployments section, taken from exam 05 task 1. Used without hesitation, which is the point of having it there.
- `revisionHistoryLimit` has no flag on `kubectl create deployment`, so a patch or an edit is required. Choosing patch over generating YAML was the faster call.
- The patch is safe. `revisionHistoryLimit` is a single number, not a map or a list, so there is no merge-versus-replace question. That question only appears for `spec.selector` (merges) and lists (depends on the merge key), as recorded in the cheat sheet table under "Update an existing object".
- Changing `revisionHistoryLimit` starts no rollout. It sits outside the pod template, and only pod template changes create a revision, per exam 05 task 1.

Two small refinements, neither of them errors:

- `get deployment chk1 -o yaml` prints the whole object to check one number. `-o jsonpath='{.spec.revisionHistoryLimit}'` prints just `50`. Exam 01 task 4 already established that habit.
- `get pod,deployment --show-labels` was not needed. No part of this task involves labels. It costs seconds, and on a timed exam those add up.

Source in my notes: [exams/05-pod-design.md](../exams/05-pod-design.md) task 1, [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Deployments, [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md)

Docs: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#revision-history-limit)

</details>

**What to fix next time**

Verify one field with JSONPath, not a full `-o yaml` dump. Skip checks the task does not ask about.

---

## Task 2 - Resolve configuration issues

Namespace `pwn` · Skill: endpoints as the diagnostic, and replacing a selector instead of merging it

**Requirement**

- The `site` Deployment in Namespace `pwn` should be reachable by clients outside the cluster through the `sitelb` Service
- Requests sent to the Service do not reach the Deployment's Pods
- Fix the Service configuration so they do

<details><summary><b>My solution</b></summary>

```bash
kubectl -n pwn get service sitelb -o yaml
kubectl -n pwn get endpoints sitelb -o yaml
kubectl -n pwn get deployment -o yaml
kubectl -n pwn get pods -o yaml

kubectl -n pwn patch service sitelb --patch='{"spec":{"selector":{"app":"site"}}}'
kubectl -n pwn get endpoints sitelb -o yaml
```

The check passed. What was missing was a request through the Service to confirm it.

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: correct, and the check passed. The only thing missing was the test that would have proved it.**

The investigation order was right. Service, then endpoints, then the workload, then the Pods. That is
the ladder in the cheat sheet under "Troubleshooting a dead Service", and reading endpoints early is
what splits a selector problem from an application problem. Diagnosing a broken selector from those
four outputs is the skill this check exists to test, and it worked.

**Why the patch was safe here, and when it would not be.** `kubectl patch` sends a strategic merge
patch, and `spec.selector` is a map, so its keys **merge** rather than being replaced. It worked
because the broken selector used the key `app`, so the patch overwrote that value. Had the old
selector used a different key, the patch would have added `app: site` and kept the old one:

```yaml
selector:
  app: site         # what the patch would add
  role: web-old     # what was already there and would stay
```

A selector is an AND across every key, so the Service would then match nothing and the endpoints
would stay empty, with no error to warn you. That behavior was verified locally while writing up
exam 06 task 4 and is the table row in the cheat sheet. Two commands replace the selector outright
and are safe whatever the old key was:

```bash
kubectl -n pwn set selector service sitelb 'app=site'
kubectl -n pwn edit svc sitelb
```

**Three faults can produce "requests do not reach the Pods", and they need different fixes.** This
one was the selector. Working through them in this order rules out a cause per check and would have
turned the guess into a decision:

```bash
# 1 - does the Service select any Ready Pod?
kubectl -n pwn get ep sitelb
# empty  -> the selector is wrong, or the Pods are not Ready
# filled -> the selector is fine, the fault is a port or the type

# 2 - what labels do the Pods actually carry? Compare the two outputs, do not eyeball the YAML
kubectl -n pwn get deployment site -o jsonpath='{.spec.template.metadata.labels}'
kubectl -n pwn get svc sitelb -o jsonpath='{.spec.selector}'

# 3 - if endpoints exist, does targetPort match the container port?
kubectl -n pwn get svc sitelb -o jsonpath='{.spec.ports}'
kubectl -n pwn get deployment site -o jsonpath='{.spec.template.spec.containers[*].ports}'

# 4 - the task says clients outside the cluster, so the type matters
kubectl -n pwn get svc sitelb          # ClusterIP cannot serve outside clients
```

The selector must match the labels on the **Pods**, at `spec.template.metadata.labels`, not the
labels on the Deployment object. Exam 06 task 4 was the same fault, and exam 04 task 2 was the
variant where the selector was right and a failing readiness probe emptied the endpoints instead.

**The missing command.** A filled endpoints list proves the Service found Pods. A request proves the
whole path works, and it takes one line with a throwaway Pod that deletes itself:

```bash
kubectl -n pwn get ep sitelb
kubectl -n pwn run client --image=curlimages/curl -it --rm --restart=Never -- curl http://sitelb:80
```

`--rm` deletes the Pod when the command exits, and it only works together with `-it`.
`--restart=Never` makes it a bare Pod, so nothing recreates it. This exact pattern is in the cheat
sheet Services section, and exam 06 task 4 handed the same command out as part of the task.

Source in my notes: [exams/06-services-and-networking.md](../exams/06-services-and-networking.md) task 4, [exams/04-observability.md](../exams/04-observability.md) task 2, [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Troubleshooting a dead Service

Docs: [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

</details>

**What to fix next time**

Memorise the throwaway client:
`kubectl -n <ns> run client --image=curlimages/curl -it --rm --restart=Never -- curl http://<svc>:<port>`.
Prefer `kubectl set selector` or `kubectl edit` over `kubectl patch` for a selector, and compare the
two selectors with JSONPath instead of reading four `-o yaml` dumps.

---

## Task 3 - Highest CPU Pod

Namespace `zz8` · Skill: `kubectl top --sort-by`, reduced to a single field

**Requirement**

- Find the Pod in Namespace `zz8` using the most CPU
- Write only its name to `/home/ubuntu/hcp001` on the Bastion node

<details><summary><b>My solution</b></summary>

```bash
kubectl -n zz8 top pod --sort-by=cpu --no-headers=true | head -n 1 | awk '{print $1}' > /home/ubuntu/hcp001
cat /home/ubuntu/hcp001
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: correct, and it matches my own recorded answer exactly.**

- `--sort-by=cpu` sorts descending, so `head -n1` is the busiest Pod. The flag accepts only `cpu` or `memory`.
- `--no-headers` removes the header row, which is what makes `head -n1` land on data instead of a title.
- `awk '{print $1}'` beats `cut -d' ' -f1` here, because column widths vary and awk treats a run of spaces as one separator. That reasoning is already in the cheat sheet.
- `>` and not `>>`, so a second attempt overwrites instead of appending. The cheat sheet lists this under habits, and it is the kind of thing that quietly fails a check.
- `cat` afterwards is the right instinct. The graded artefact is the file, so look at the file.
- Needs metrics-server in the cluster. If `kubectl top` returns an error rather than rows, that is the cause, not the command.

This is the same pipeline as exam 04 task 5, minus the duplicated `--no-headers=true` that appeared
there. Cleaner this time.

Source in my notes: [exams/04-observability.md](../exams/04-observability.md) task 5, [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Resource usage, [labs/06-probes-and-monitoring.md](../labs/06-probes-and-monitoring.md)

Docs: [kubectl top pod](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#top)

</details>

**What to fix next time**

Nothing. Keep this one as it is.

---

## Task 4 - Pod Secret

Namespace `sjq` · Skill: `secretKeyRef` with a renamed variable, and reading an env var back

**Requirement**

- Secret named `xh8jqk7z` in Namespace `sjq`, generic, key `tkn`, value `hy8szK2iu`
- Pod named `server` using image `httpd:2.4.39-alpine`
- The container reads the `tkn` key through an environment variable named `SECRET_TKN`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n sjq create secret generic xh8jqk7z --from-literal=tkn=hy8szK2iu
kubectl -n sjq get secret xh8jqk7z

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: server
  namespace: sjq
spec:
  containers:
  - name: server
    image: httpd:2.4.39-alpine
    env:
    - name: SECRET_TKN
      valueFrom:
        secretKeyRef:
          name: xh8jqk7z 
          key: tkn
EOF

kubectl -n sjq get pods -o yaml
kubectl -n sjq exec server -it -- echo "$SECRET_TKN"
```

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: the answer is right, the verification is broken.**

What is right, and why each choice was the only correct one:

- `kubectl create secret generic --from-literal` avoids encoding base64 by hand. Straight from exam 02 task 1.
- `env` with `secretKeyRef` is the **only** way to satisfy this wording. The task renames the variable to `SECRET_TKN` while the key is `tkn`, and only `env` plus `secretKeyRef` can rename. `envFrom` with `secretRef` would have created a variable called `TKN`. The cheat sheet states this distinction for ConfigMaps and it applies identically to Secrets.
- YAML was unavoidable. `kubectl run --env` can only set a literal value, which would hardcode the secret instead of referencing it, and `kubectl set env --from=secret/...` needs an object with a pod template, so it does not work on a bare Pod. That limit was tested and recorded in exam 02.
- The trailing space after `name: xh8jqk7z ` is harmless. YAML strips it.

**The broken part.** This command cannot work:

```bash
kubectl -n sjq exec server -it -- echo "$SECRET_TKN"
```

Two independent reasons:

1. The double quotes let the **local** shell on the Bastion expand `$SECRET_TKN` before kubectl runs. That variable is not set locally, so it expands to nothing and kubectl is asked to run `echo` with an empty argument. The output is a blank line.
2. Even with single quotes it would fail. `exec` runs `echo` directly, with no shell inside the container to expand anything, so it would print the literal text `$SECRET_TKN`.

Any of these do the job:

```bash
kubectl -n sjq exec server -- printenv SECRET_TKN         # shortest
kubectl -n sjq exec server -- sh -c 'echo $SECRET_TKN'    # single quotes, the container's shell expands it
kubectl -n sjq exec server -- env | grep SECRET_TKN       # see it in context
```

`-it` is not needed for a command that prints and exits. It matters for an interactive shell, and for
`kubectl run --rm`, where `--rm` only deletes the Pod if `-it` is present.

`kubectl get pods -o yaml` does confirm the `secretKeyRef` block reached the API, but that only
proves the manifest was accepted. It does not prove the container received the value. The cheat sheet
already says to check inside the container, not just the manifest.

Source in my notes: [exams/02-configuration.md](../exams/02-configuration.md) tasks 1 and 5, [labs/10-configmaps-and-secrets.md](../labs/10-configmaps-and-secrets.md), [exams/CHEATSHEET.md](../exams/CHEATSHEET.md) Secrets and Verify your work

Docs: [Using Secrets as environment variables](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables) · [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

</details>

**What to fix next time**

`kubectl exec <pod> -- printenv VAR` to read an environment variable. Never put `$VAR` in double
quotes after `--`, because the local shell eats it first.

---

## Pattern check

**Doing well**

1. Reaching for the imperative command first. Tasks 1 and 3 were answered with the exact commands recorded in the cheat sheet, without detouring through YAML.
2. Knowing when YAML is unavoidable. Task 4 needed `secretKeyRef` and got it, including the rename that rules out `envFrom`.
3. Diagnosing a broken Service from its own outputs. Service, endpoints, workload, Pods is the correct ladder, and task 2 was solved by walking it.
4. Looking at the graded artefact. `cat` on the output file in task 3 is the right reflex.

**To fix**

1. **Verification is the weak spot, not the solutions.** Four out of four were solved. Task 2 was still marked uncertain and task 4 used a command that prints a blank line. The knowledge is there, the read-back habit is not.
2. **The throwaway curl Pod is not yet automatic.** `kubectl run client --image=curlimages/curl -it --rm --restart=Never -- curl http://<svc>:<port>` is one line and it is what turns a guess into a pass. Drill it until it is typed without thinking.
3. **`kubectl patch` on a selector got away with it.** It merges map keys, and it only worked because the broken selector used `app`. `kubectl set selector` and `kubectl edit` are safe whatever the old key was. Exam 06 recorded this trap one exam earlier.
4. **Reading whole objects to check one field.** Three `-o yaml` dumps in task 2 and one in task 1. JSONPath prints the single value being checked.
5. **Unrequested extra checks.** `--show-labels` in task 1 answered a question nobody asked.

**One habit that would have removed all doubt**

After every task, run the one command that reads back exactly what the check grades. For a field, that
is JSONPath. For a Service, `get ep` and then a curl from a throwaway Pod. For an environment
variable, `printenv` inside the container.
