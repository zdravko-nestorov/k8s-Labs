# Exam 02 - Configuration

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: Configuration  
> Checks: 5  
> Solution guide: https://app.qa.com/resource/ckad-practice-exam-configuration-solution-guide/  
> Topics: secrets, configmaps, volumes, envFrom, serviceaccount, securityContext, runAsUser, fsGroup, resources, requests, limits, labels, multi-container, kubectl-create, kubectl-set, kubectl-patch, dry-run

## Format

Five checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

```bash
# 1 - Secret from a literal, no base64 by hand
kubectl create secret generic app-secret -n yqe --from-literal=password=abnaoieb2073xsj
# the Pod needs a volume, so write YAML for it

# 2 - new ServiceAccount on an existing Deployment
kubectl create sa secure-svc -n app
kubectl set serviceaccount deployment secapp secure-svc -n app

# 3 - two containers with different runAsUser: YAML only, kubectl run makes one container

# 4 - labels and port have flags, memory does not
kubectl run web1 -n ca100 --image=nginx -l env=prod,type=processor --port=80 \
  --dry-run=client -o yaml > pod.yaml
# add resources.requests.memory and resources.limits.memory, then apply

# 5 - ConfigMap from literals, then envFrom in YAML
kubectl create configmap config1 -n ca200 --from-literal COLOUR=red --from-literal SPEED=fast
```

## Exam strategy notes

- **`kubectl create` covers Secrets and ConfigMaps.** `--from-literal` handles the encoding, so you never type base64.
- **`kubectl set` beats `kubectl patch`** for the fields it supports: `serviceaccount`, `image`, `env`, `resources`, `selector`.
- **This domain forces YAML.** Volumes, second containers, security contexts, resource limits, and `envFrom` have no `kubectl run` flag. Expect to generate and edit.
- **Verify inside the container**, not in the manifest. `kubectl exec -- id` and `kubectl logs` prove the setting took effect. A correct-looking manifest can still fail.
- **`kubectl explain`** finds the field path fast, for example `kubectl explain pod.spec.securityContext`.

---

## Task 1 - Create and consume a Secret using a volume

Namespace `yqe` · Skill: `kubectl create secret`, secret volume

**Requirement**

- Secret named `app-secret` in Namespace `yqe`, holding `password=abnaoieb2073xsj`
- Pod named `app` running a `memcached` container
- The Pod consumes the Secret through a volume mounted at `/etc/app`

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: yqe
type: Opaque
data:
  password: $(echo -n "abnaoieb2073xsj" | base64 -w0)
---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: yqe
spec:
  containers:
  - name: app
    image: memcached
    volumeMounts:
    - name: app-secret
      mountPath: "/etc/app"
  volumes:
  - name: app-secret
    secret:
      secretName: app-secret
EOF

kubectl --namespace=yqe describe pod app
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl -n yqe create secret generic --from-literal=password=abnaoieb2073xsj app-secret

cat << EOF | kubectl apply -n yqe -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: memcached
    volumeMounts:
    - name: secret
      mountPath: "/etc/app"
  volumes:
  - name: secret
    secret:
      secretName: app-secret
EOF
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `generic` is the Opaque Secret type and the one the exam will ask for. The others are `docker-registry` and `tls`.
- `kubectl create secret generic --from-literal=key=value` encodes the value for you. Hand-rolled base64 is the main way to lose this task.
- If you must write the Secret as YAML, use `stringData` instead of `data`. Kubernetes encodes it on the way in.
- Secrets are namespaced, so the Pod must sit in the same Namespace even when the task does not say so.
- The volume name is internal. It only has to match between `volumes` and `volumeMounts`.
- Combining the heredoc with `kubectl apply -f -` means you can paste from the exam notepad without opening an editor.

Review: [labs/10-configmaps-and-secrets.md](../labs/10-configmaps-and-secrets.md)

Docs: [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Distribute credentials securely](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

</details>

**Differences that matter**

- `kubectl create secret generic --from-literal` is one short line and removes the encoding step entirely.
- My `$(echo -n "..." | base64 -w0)` works, but `-w0` is a GNU option. It fails on macOS and on BusyBox. Inside an unquoted heredoc the shell expands it, so it also silently breaks if you ever quote the delimiter as `<<'EOF'`.

---

## Task 2 - Update a Deployment with a new ServiceAccount

Namespace `app` · Skill: `kubectl set serviceaccount`

**Requirement**

- Deployment `secapp` in Namespace `app` currently uses the `default` ServiceAccount
- Create a ServiceAccount named `secure-svc` in the same Namespace
- Make the existing Deployment use it, so the replicas actually run with it

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-svc
  namespace: app
EOF

kubectl --namespace=app get serviceaccounts
kubectl --namespace=app set serviceaccount deployment secapp secure-svc
kubectl --namespace=app get deployment secapp -o yaml
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl create serviceaccount -n app secure-svc
```

Create a file named `patch.yaml`:

```yaml
spec:
  template:
    spec:
      serviceAccountName: secure-svc
```

Then patch the Deployment:

```bash
kubectl patch deployment -n app secapp --patch "$(cat patch.yaml)"
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl set serviceaccount (-f FILENAME | TYPE NAME) SERVICE_ACCOUNT` updates the pod template in place. Confirmed on kubectl v1.36.3.
- The field is `spec.template.spec.serviceAccountName`. Setting it on the template, not the Deployment itself, is what makes new Pods pick it up.
- Changing the pod template starts a rollout, so the running replicas are replaced. That is what "ensuring the replicas now run with it" means.
- Check the Pods, not the Deployment: `kubectl -n app get pods -o jsonpath='{.items[*].spec.serviceAccountName}'`.
- `kubectl create sa` is the short alias for `kubectl create serviceaccount`.
- Keep `kubectl patch` for fields that `kubectl set` does not cover.

Review: [labs/11-using-serviceaccounts.md](../labs/11-using-serviceaccounts.md), [labs/03-managing-pods-with-deployments.md](../labs/03-managing-pods-with-deployments.md)

Docs: [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) · [Update API objects in place using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)

</details>

**Differences that matter**

Mine is clearly better. `kubectl set serviceaccount` is one command. The official answer creates a file, then reads it back with `$(cat patch.yaml)`, which means an editor session and a chance to get the nesting wrong.

The only improvement to mine: `kubectl create sa secure-svc -n app` instead of the ServiceAccount manifest.

```bash
kubectl create sa secure-svc -n app
kubectl set serviceaccount deployment secapp secure-svc -n app
```

---

## Task 3 - Pod security context configuration

Namespace `dnn` · Skill: pod-level versus container-level `securityContext`

**Requirement**

- Pod named `secpod` in Namespace `dnn` with two containers, `c1` and `c2`
- Both run the `bash` image and the command `/usr/local/bin/bash -c sleep 3600`
- `c1` runs as user ID 1000, `c2` runs as user ID 2000
- Both use file system group ID 3000

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secpod
  namespace: dnn
spec:
  securityContext:
    fsGroup: 3000
  containers:
  - name: c1
    image: bash
    command: ["/usr/local/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      runAsUser: 1000
  - name: c2
    image: bash
    command: ["/usr/local/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      runAsUser: 2000
EOF

kubectl --namespace=dnn get pod secpod -o yaml | grep fsGroup
kubectl --namespace=dnn get pod secpod -o yaml | grep runAsUser
```

</details>

<details><summary><b>Official solution</b></summary>

Generate a one-container template, then duplicate the container in vim:

```bash
(kubectl run --image=bash -n dnn secpod -l env=prod --dry-run=client -o yaml -- /usr/local/bin/bash -c 'sleep 3600') > pod.yaml
```

The edited `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    env: prod
  name: secpod
  namespace: dnn
spec:
  securityContext:
    fsGroup: 3000
  containers:
  - name: c1
    securityContext:
      runAsUser: 1000
    args:
    - /usr/local/bin/bash
    - -c
    - sleep 3600
    image: bash
    resources: {}
  - name: c2
    securityContext:
      runAsUser: 2000
    args:
    - /usr/local/bin/bash
    - -c
    - sleep 3600
    image: bash
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

```bash
kubectl apply -f pod.yaml
kubectl get pods -n dnn

kubectl exec -it -n dnn secpod -c c1 -- id
# uid=1000 gid=0(root) groups=0(root),3000
kubectl exec -it -n dnn secpod -c c2 -- id
# uid=2000 gid=0(root) groups=0(root),3000
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl run` creates exactly one container. Any multi-container Pod is a YAML task, so start typing the manifest instead of hunting for a flag.
- `fsGroup` exists only at pod level, under `spec.securityContext`. It sets the group that owns mounted volumes.
- `runAsUser` exists at both levels. The container value wins where both are set, which is why two different users in one Pod are possible.
- `kubectl explain pod.spec.securityContext` and `kubectl explain pod.spec.containers.securityContext` show which fields live where.
- The `bash` image has no ENTRYPOINT, so `args` alone runs the command. Adding an explicit `command` does the same thing and reads more clearly.

Review: [labs/08-configuring-pod-security-contexts.md](../labs/08-configuring-pod-security-contexts.md)

Docs: [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

</details>

**Differences that matter**

- Mine is the better route here. Generating a single-container template and then duplicating the block in vim is more work than typing the manifest, because the generated output needs the same edits anyway.
- The official command adds `-l env=prod`, a label the task never asked for.
- The official verification is better than mine. `kubectl exec -c c1 -- id` proves the container really runs as 1000 and carries group 3000. Grepping the manifest only proves the API accepted the field.

---

## Task 4 - Pod resource constraints

Namespace `ca100` · Skill: fields with no `kubectl run` flag

**Requirement**

- Pod named `web1` in Namespace `ca100`, image `nginx`
- Labels `env=prod` and `type=processor`
- Memory request `100Mi`, memory limit `200Mi`
- Expose the Pod on port 80

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web1
  namespace: ca100
  labels:
    env: prod
    type: processor
spec:
  containers:
  - name: web1
    image: nginx
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
EOF

kubectl --namespace=ca100 get pod web1 -o yaml
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
(kubectl run web1 -n ca100 --image=nginx -l env=prod,type=processor --port=80 --dry-run=client -o yaml) > pod.yaml
```

Then add the resources block to the container in vim:

```yaml
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
```

```bash
kubectl apply -f pod.yaml
kubectl get pods -n ca100
kubectl describe pod -n ca100 web1
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl run` has no `--requests` or `--limits`. Checked on kubectl v1.36.3: the flag list has `--labels`, `--port`, `--env`, `--restart`, `--overrides`, but nothing for resources. Older kubectl versions had them, so do not rely on memory of an older exam.
- `kubectl set resources` cannot help either. It updates "objects with pod templates", and a bare Pod has none.
- So this is generate and edit, or hand-written YAML. Both are fine.
- "Expose the pod on port 80" here means `containerPort: 80`, which `--port=80` sets. Do not add `--expose`, that creates a Service nobody asked for.
- `-l` is the short form of `--labels` and takes a comma-separated list.
- `kubectl describe pod` shows requests and limits in readable form, which is faster to check than reading YAML.

Review: [labs/07-requesting-and-limiting-resources.md](../labs/07-requesting-and-limiting-resources.md)

Docs: [Assign memory resources to containers](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/) · [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

**Differences that matter**

Both answers do the same work. The official one lets `kubectl run` write the boilerplate, which pays off as the label and port list grows.

There is a one-line version using `--overrides`, but typing JSON correctly under time pressure is riskier than editing YAML:

```bash
kubectl run web1 -n ca100 --image=nginx -l env=prod,type=processor --port=80 \
  --overrides='{"spec":{"containers":[{"name":"web1","image":"nginx","ports":[{"containerPort":80}],"resources":{"requests":{"memory":"100Mi"},"limits":{"memory":"200Mi"}}}]}}'
```

---

## Task 5 - Create a Pod with ConfigMap environment variables

Namespace `ca200` · Skill: `kubectl create configmap`, `envFrom`

**Requirement**

- ConfigMap named `config1` in Namespace `ca200` with `COLOUR=red` and `SPEED=fast`
- Pod named `redfastcar` in the same Namespace, image `busybox`
- The Pod exposes the ConfigMap settings as environment variables
- The Pod runs `/bin/sh -c "env | grep -E 'COLOUR|SPEED'; sleep 3600"`

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: config1
  namespace: ca200
data:
  COLOUR: "red"
  SPEED: "fast"
---
apiVersion: v1
kind: Pod
metadata:
  name: redfastcar
  namespace: ca200
spec:
  containers:
  - name: redfastcar
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "env | grep -E 'COLOUR|SPEED'; sleep 3600"]
    envFrom:
      - configMapRef:
          name: config1
EOF

kubectl --namespace=ca200 describe pod redfastcar
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
kubectl create configmap config1 -n ca200 --from-literal COLOUR=red --from-literal SPEED=fast

(kubectl run -n ca200 redfastcar --image=busybox --dry-run=client -o yaml -- /bin/sh -c "env | grep -E 'COLOUR|SPEED'; sleep 3600") > pod.yaml
```

Then add `envFrom` to the container in vim:

```yaml
   envFrom:
   - configMapRef:
       name: config1
```

```bash
kubectl apply -f pod.yaml

kubectl logs -n ca200 redfastcar
# COLOUR=red
# SPEED=fast
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- `kubectl create configmap <name> --from-literal K=V --from-literal K2=V2` is the fast path. Repeat the flag per pair.
- `envFrom` with `configMapRef` imports every key as an environment variable, using the key names as-is. Use `env` with `configMapKeyRef` when you need one key or a different variable name.
- There is no `kubectl run` flag for `envFrom`. `--env COLOUR=red` sets a literal value that does not reference the ConfigMap, so it would likely fail the check.
- `kubectl set env --from=configmap/config1` exists, but every documented example targets a Deployment, meaning an object with a pod template. Do not count on it for a bare Pod.
- `kubectl logs` is the right check here, because the container command prints the variables.

Review: [labs/10-configmaps-and-secrets.md](../labs/10-configmaps-and-secrets.md)

Docs: [Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) · [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

</details>

**Differences that matter**

- `kubectl create configmap --from-literal` beats writing the ConfigMap manifest.
- The Pod half is equivalent. `envFrom` has no flag, so both answers end up editing YAML.
- The official guide has a copy error worth ignoring: its generated manifest shows the name `car1` before the text tells you to rename it to `redfastcar`.

---

## What to take into the next exam

1. Secrets and ConfigMaps come from `kubectl create ... --from-literal`. Never encode base64 by hand.
2. If a Secret must be YAML, use `stringData` and let Kubernetes encode it.
3. `kubectl set serviceaccount|image|env|resources` updates a pod template in one command. Reach for `kubectl patch` only when `kubectl set` has no subcommand for the field.
4. `kubectl run` builds one container. Two containers means YAML from the start.
5. `fsGroup` is pod level. `runAsUser` works at both levels and the container value wins.
6. Verify from inside: `kubectl exec -- id` for identity, `kubectl logs` for environment variables.
7. Fields with no imperative flag so far: `terminationGracePeriodSeconds`, `resources.requests`, `resources.limits`, `envFrom`, `securityContext`, `volumes` and `volumeMounts`.
