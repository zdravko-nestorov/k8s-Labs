# Exam 06 - Services and Networking

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Services & Networking  
> Checks: 5  
> Solution guide: https://cloudacademy.com/resource/ckad-practice-exam-services-networking-solution-guide/  
> Topics: services, clusterip, nodeport, target-port, endpoints, kubectl-expose, kubectl-create-service, set-selector, label-selector, troubleshooting, networkpolicy, podselector, pod-networking, pods, restartPolicy, kubectl-run, kubectl-patch, kubectl-edit

## Format

Five checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

```bash
# 1 - Pod with a restart policy and a container port
kubectl -n red run basic --image=nginx:stable-alpine-perl --restart=OnFailure --port=80

# 2 - expose copies the selector off the Pod, so one command is enough
kubectl -n red expose pod basic --name=cloudacademy-svc --port=8080 --target-port=80

# 3 - expose has no --node-port flag, so patch the port in afterwards
kubectl -n ca1 expose deployment cloudforce --name=cloudforce-svc --type=NodePort --port=80
kubectl -n ca1 patch svc cloudforce-svc --patch '{"spec": {"ports": [{"port": 80, "nodePort": 32080}]}}'

# 4 - no endpoints means a wrong selector, set selector replaces it outright
kubectl -n skynet get ep t2-svc
kubectl -n skynet set selector service t2-svc 'app=t2'
kubectl run client -n skynet --image=appropriate/curl -it --rm --restart=Never -- curl http://t2-svc:8080 > /home/ubuntu/svc-output.txt

# 5 - change only spec.ingress[0].from[0].podSelector, never spec.podSelector
kubectl -n sec1 edit netpol netpol1
```

## Exam strategy notes

- **`kubectl expose` is the Service command.** It reads the Pod or Deployment and copies its labels into the Service selector. `kubectl create service` cannot do that, so it always costs a second command.
- **A Service has three ports and each answers at a different address.** `port` on the ClusterIP and the DNS name, `nodePort` on every node IP, `targetPort` on the container. Curling the ClusterIP on the nodePort always fails.
- **`kubectl get ep <svc>` is the first command when a Service does not answer.** An empty list means the selector matches no Ready Pod. It splits "wrong selector" from "broken app" in one step.
- **Know which update command merges and which replaces.** `kubectl patch` merges maps, `kubectl set selector` replaces the whole selector. Picking the wrong one can leave a stale label behind and no endpoints.
- **Pod names are not DNS names.** Only Services get DNS entries. Test Pod to Pod with the Pod IP from `kubectl get pod <name> -o jsonpath='{.status.podIP}'`.
- **Keep a throwaway client Pod in muscle memory.** `kubectl run client --image=curlimages/curl -it --rm --restart=Never -- curl http://<svc>:<port>` works even when the target image has no shell tools.
- **A NetworkPolicy has two selectors doing opposite jobs.** Read which one the task means before editing.

---

## Task 1 - Create and configure a basic Pod

Namespace `red` · Skill: `kubectl run` flags

**Requirement**

- Pod named `basic` in Namespace `red`
- Image `nginx:stable-alpine-perl` for its only container
- Restart the Pod only `OnFailure`
- Port 80 open to TCP traffic

<details><summary><b>My solution</b></summary>

```bash
kubectl -n red run basic --image=nginx:stable-alpine-perl --restart=OnFailure --port=80
kubectl -n red get pod basic -o yaml
kubectl -n red exec -it basic -- curl localhost:80
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl run -n red --image=nginx:stable-alpine-perl --restart=OnFailure --port=80 basic
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- Flag order and the position of the Pod name do not matter. The two commands are the same command.
- `--port=80` writes `containerPort: 80`. TCP is the default protocol, so "open to TCP" needs nothing extra.
- `kubectl run` defaults to `--restart=Always`. Set it whenever the task names `Never` or `OnFailure`.
- Three ways to reach a manifest if you need one: copy an example from the docs, run `--dry-run=client -o yaml` (the `--image` flag is still required), or dump an existing Pod with `-o yaml` and delete the noise. The third is slow, a live Pod prints hundreds of lines.
- The lab host runs `source <(kubectl completion bash)` for you, so tab completion lists the flags faster than `--help` does.
- `kubectl explain pod.spec` gives field names when you write YAML by hand.
- Practise vim before the real exam. The exam notepad is the other way to compose a manifest before pasting it in.

Review: [labs/01-pod-definition-basics.md](../labs/01-pod-definition-basics.md), [exams/01-core-concepts.md](01-core-concepts.md) task 1 is the same shape

Docs: [Configure Pods and Containers](https://kubernetes.io/docs/tasks/configure-pod-container/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Difference that matters**

None for the answer itself. One warning about the verification: alpine based images ship a minimal toolset and `curl` is not part of it, so `kubectl exec -it basic -- curl localhost:80` usually dies with `executable file not found`. That failure says nothing about the Pod. Check with `kubectl get pod basic -o yaml` instead, or send a throwaway curl Pod at it as you did in task 2.

---

## Task 2 - Expose a Pod

Namespace `red` · Skill: `kubectl expose pod`

**Requirement**

- Expose the Pod named `basic` in Namespace `red`
- Service name `cloudacademy-svc`
- Service port 8080
- Target port 80
- Service type ClusterIP

<details><summary><b>My solution</b></summary>

```bash
kubectl -n red create service clusterip cloudacademy-svc --tcp=8080:80
kubectl -n red set selector service cloudacademy-svc 'run=basic'

kubectl -n red get service cloudacademy-svc -o yaml
kubectl -n red exec -it basic -- curl http://cloudacademy-svc:8080
kubectl -n red run client --image=appropriate/curl -it --rm --restart=Never -- curl http://cloudacademy-svc:8080
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl expose pod basic -n red --name=cloudacademy-svc --port=8080 --target-port=80
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl expose` exists to copy a selector. It reads the Pod, takes its labels, and writes them into `spec.selector`. That is why it beats `create service` on every task that names an existing object.
- `kubectl create service clusterip` has no `--selector` flag, checked on kubectl v1.36.3. It always writes `app: <service-name>`, so a wrong selector is guaranteed unless you fix it afterwards.
- ClusterIP is the default type for both commands, so `--type` can be left out here.
- `--port` is what the Service listens on, `--target-port` is the container port. Omit `--target-port` and it copies `--port`, which would have been wrong here because the two differ.
- Add `--dry-run=client -o yaml` to either command to get a manifest instead.

Review: [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md) Step 17, [labs/14-deployment-with-configmaps-and-helm.md](../labs/14-deployment-with-configmaps-and-helm.md) Step 15

Docs: [Service](https://kubernetes.io/docs/concepts/services-networking/service/) · [Connect Applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)

</details>

**Difference that matters**

- One command against two. Mine had to name the label `run=basic`, which `expose` would have found on its own.
- `run=basic` was the right guess because `kubectl run` labels the Pod `run=<pod-name>`. Worth remembering next to its sibling: `kubectl create deployment` labels its Pods `app=<deployment-name>`.
- Both end at the same Service. Mine also names the port `8080-80`, a side effect of `--tcp=8080:80`, which changes nothing for a single-port Service.

---

## Task 3 - Expose an existing Deployment as NodePort

Namespace `ca1` · Skill: `expose` plus a patch for the nodePort

**Requirement**

- Deployment `cloudforce` already exists in Namespace `ca1`
- Service name `cloudforce-svc`
- Service type NodePort
- Service port 80
- NodePort 32080

<details><summary><b>My solution</b></summary>

```bash
kubectl -n ca1 get deployment cloudforce -o yaml
kubectl -n ca1 expose deployment cloudforce --name=cloudforce-svc --type=NodePort --port=80
kubectl -n ca1 patch service cloudforce-svc -p '{"spec":{"ports":[{"port":80,"nodePort":32080}]}}'

kubectl -n ca1 get svc cloudforce-svc -o yaml
kubectl -n ca1 get endpoints cloudforce-svc -o yaml

kubectl -n ca1 exec -it cloudforce-7659595cc6-jznnw -- curl http://cloudforce-svc:80
kubectl -n ca1 run client --image=appropriate/curl -it --rm --restart=Never -- curl http://cloudforce-svc:80
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl expose deployment -n ca1 cloudforce --name=cloudforce-svc --port=80 --type=NodePort
kubectl patch -n ca1 svc cloudforce-svc --patch '{"spec": {"ports": [{"port": 80, "nodePort": 32080}]}}'
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl expose` has no `--node-port` flag. The full flag list on v1.36.3 is `--cluster-ip`, `--external-ip`, `--load-balancer-ip`, `--name`, `--overrides`, `--port`, `--protocol`, `--selector`, `--session-affinity`, `--target-port`, `--type`, `-l`. Without a second step the API server picks a random port from 30000-32767.
- The patch works because `spec.ports` merges on the `port` key. Sending `{"port": 80, "nodePort": 32080}` finds the existing entry for port 80 and adds the nodePort, leaving `targetPort`, `protocol`, and the port name untouched.
- `--target-port` was not needed. Left out, it copies `--port`, giving `targetPort: 80`, which matches the container.
- `kubectl edit svc cloudforce-svc` and typing `nodePort: 32080` does the same job if you type faster than you build JSON.
- Confirm with `kubectl -n ca1 get svc cloudforce-svc`. The PORT(S) column must read `80:32080/TCP`.

Review: [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md), [exams/04-observability.md](04-observability.md) task 2

Docs: [Service type NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) · [Update API Objects in Place Using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)

</details>

**Difference that matters**

None, the two solutions are the same pair of commands.

The verification is where the trap sits. `curl http://cloudforce-svc:80` from inside the cluster is correct, and `curl http://cloudforce-svc:32080` would have failed. The Service name resolves to the ClusterIP, and the ClusterIP only listens on 80. Port 32080 lives on the node IPs, so testing it means `kubectl get nodes -o wide` and then `curl <NodeIP>:32080`.

---

## Task 4 - Fix a Service that returns nothing

Namespace `skynet` · Skill: endpoints as the diagnostic, repairing a selector

**Requirement**

- Deployment `t2` in Namespace `skynet` is exposed by a ClusterIP Service named `t2-svc`
- `t2-svc` should return a valid HTTP response but does not, so find the cause and fix it
- Then run the provided command to save the response:

```bash
kubectl run client -n skynet --image=appropriate/curl -it --rm --restart=Never -- curl http://t2-svc:8080 > /home/ubuntu/svc-output.txt
```

<details><summary><b>My solution</b></summary>

```bash
kubectl -n skynet get service t2-svc -o yaml
kubectl -n skynet get endpoints t2-svc -o yaml
kubectl -n skynet get pods -l app=t2
kubectl -n skynet patch service t2-svc -p '{"spec":{"selector":{"app":"t2"}}}'

kubectl run client -n skynet --image=appropriate/curl -it --rm --restart=Never -- curl http://t2-svc:8080 > /home/ubuntu/svc-output.txt
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
# Compare the Service selector against the pod labels in the Deployment
kubectl describe deployments.apps -n skynet t2
kubectl describe svc -n skynet t2-svc
# No endpoints are registered
kubectl get ep -n skynet t2-svc
# Correct the selector to app=t2
kubectl edit svc -n skynet t2-svc
# Endpoints appear
kubectl get ep -n skynet t2-svc
# Save the response
kubectl run client -n skynet --image=appropriate/curl -it --rm --restart=Never -- curl http://t2-svc:8080 > /home/ubuntu/svc-output.txt
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- The Service selector must match the labels on the **Pods**, which live at `spec.template.metadata.labels` in the Deployment. The labels on the Deployment object itself are a different set and are not what the Service uses.
- An empty endpoints list is the signal. It separates a selector problem from an application problem before you read any logs.
- `kubectl get ep` is the short form of `kubectl get endpoints`, and both take a Service name.
- `kubectl edit svc` is the official route and needs no JSON. `kubectl set selector service t2-svc 'app=t2'` does it in one non-interactive line.
- The provided curl command is the graded artefact. Run it after the fix, never before, or the file holds the failure.

Review: [labs/14-deployment-with-configmaps-and-helm.md](../labs/14-deployment-with-configmaps-and-helm.md) for the describe and events loop, [exams/04-observability.md](04-observability.md) task 2 for the same skill with a probe as the cause

Docs: [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/) · [Service](https://kubernetes.io/docs/concepts/services-networking/service/)

</details>

**Difference that matters**

My patch worked here, but it is the riskier command, and the reason is worth knowing.

`kubectl patch` sends a strategic merge patch, and `spec.selector` is a map, so the keys **merge** instead of replacing. Starting from a broken selector of `application: t2` and `tier: web`, the patch `{"spec":{"selector":{"app":"t2"}}}` produces all three keys:

```yaml
selector:
  app: t2
  application: t2
  tier: web
```

A selector is an AND across every key, so the Service would still match nothing and the endpoints would stay empty. Mine only worked because the broken selector used the same key, `app`, and the patch overwrote its value.

Two commands do replace the whole selector, and either is safer here:

```bash
kubectl -n skynet set selector service t2-svc 'app=t2'
kubectl -n skynet edit svc t2-svc
```

Either way, prove it with `kubectl -n skynet get ep t2-svc` before running the graded curl.

---

## Task 5 - Fix a NetworkPolicy

Namespace `sec1` · Skill: the two selectors in a NetworkPolicy

**Requirement**

- `pod1` carries the label `app=test`, `pod2` carries `app=client`
- NetworkPolicy `netpol1` currently blocks traffic from `pod2` to `pod1`
- Update it so `pod2` can reach `pod1`
- The policy must still apply to `pod1`

<details><summary><b>My solution</b></summary>

```bash
kubectl -n sec1 exec -it pod2 -- ping pod1

kubectl -n sec1 get pods --show-labels
kubectl -n sec1 get networkpolicy -o yaml
kubectl -n sec1 patch networkpolicy netpol1 -p '{"spec":{"ingress":[{"from":[{"podSelector":{"matchLabels":{"app":"client"}}}]}],"podSelector":{"matchLabels":{"app":"test"}},"policyTypes":["Ingress"]}}'
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
# Confirm the traffic is blocked, using the Pod IP
pod1IP=$(kubectl get pods pod1 -n sec1 -o jsonpath='{.status.podIP}')
kubectl -n sec1 exec -it pod2 -- ping $pod1IP
# Compare the Pod labels against the policy
kubectl get pods -n sec1 --show-labels
kubectl -n sec1 describe netpol netpol1
# Fix only the ingress pod selector
kubectl edit -n sec1 netpol netpol1
# Confirm the traffic now passes
pod1IP=$(kubectl get pods pod1 -n sec1 -o jsonpath='{.status.podIP}')
kubectl -n sec1 exec -it pod2 -- ping $pod1IP
```

The spec after editing, with only `spec.ingress[0].from[0].podSelector` changed:

```yaml
spec:
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
  podSelector:
    matchLabels:
      app: test
  policyTypes:
  - Ingress
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- A NetworkPolicy holds two selectors that do opposite jobs. `spec.podSelector` picks the Pods the policy protects, here `pod1` with `app=test`. `spec.ingress[].from[].podSelector` picks who is allowed to send, here `pod2` with `app=client`. Only the second one was wrong.
- "Ensure the NetworkPolicy is still being applied to pod1" is the task warning you not to touch `spec.podSelector`. Changing it would move the policy to different Pods and fail the check even though traffic flows.
- Both selectors are namespace scoped by default, so `podSelector` under `from` only matches Pods in `sec1`. Reaching other namespaces needs `namespaceSelector`.
- `kubectl describe netpol netpol1` prints the rules as readable text, which is easier to compare against `--show-labels` output than raw YAML.
- The policy only does anything if the cluster runs a CNI plugin that enforces NetworkPolicy. Without one the object is accepted and ignored.

Review: no lab covers NetworkPolicy. The nearest file is [labs/18-api-access-control-authn-authz-admission.md](../labs/18-api-access-control-authn-authz-admission.md), but that is about who may call the API, not which Pods may exchange traffic. This is a real gap in these notes.

Docs: [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) · [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

</details>

**Differences that matter**

- `ping pod1` cannot work. Kubernetes publishes DNS names for Services, not for bare Pods. A Pod finds its own name in its own `/etc/hosts`, but `pod2` has no entry for `pod1`, so the command fails with a name resolution error whether the policy blocks traffic or not. It proves nothing either before or after the fix. The official grabs `.status.podIP` first for exactly this reason.
- My patch was correct, and for a reason worth keeping. `spec.ingress` is a list with no merge key, so a strategic merge patch **replaces** it rather than merging. I confirmed this locally: patching a policy whose ingress allowed `app: wrong-label` left a single rule allowing `app: client`, with `podSelector` and `policyTypes` untouched.
- Resending `podSelector` and `policyTypes` unchanged was harmless but not free. It puts the one field the task forbids you to change into your command, where a typo would silently fail the check. Editing only the ingress selector removes that risk.

---

## What to take into the next exam

1. `kubectl expose` copies the selector off the object. `kubectl create service` writes `app=<service-name>` and always needs a `kubectl set selector` after it.
2. `kubectl expose` has no `--node-port`. Create the Service, then patch `{"spec": {"ports": [{"port": <port>, "nodePort": <nodePort>}]}}`.
3. A NodePort Service answers on three addresses. Service name and `port` inside the cluster, node IP and `nodePort` outside, `targetPort` in the container.
4. `kubectl get ep <svc>` is the first move when a Service is silent. Empty means the selector matches no Ready Pod.
5. `kubectl patch` merges maps and can leave a stale label in a selector. `kubectl set selector` replaces the whole thing.
6. `kubectl patch` replaces a list that has no merge key, which is why the NetworkPolicy ingress patch was safe.
7. Pod names are not DNS names. Test Pod to Pod with `kubectl get pod <name> -o jsonpath='{.status.podIP}'`.
8. NetworkPolicy: `spec.podSelector` is who is protected, `ingress.from.podSelector` is who may send. Read the task twice before choosing.
9. Throwaway client: `kubectl run client --image=curlimages/curl -it --rm --restart=Never -- curl http://<svc>:<port>`. `--rm` needs `-it` to work.
10. `kubectl run` labels its Pod `run=<name>`. `kubectl create deployment` labels its Pods `app=<name>`.
