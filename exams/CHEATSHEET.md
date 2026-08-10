# Exam Cheat Sheet

Fastest known command for each task type met in the practice exams. Grows with every new exam.

Read this before an attempt. Every entry cites the exam it came from.

## Contents

| Section | Use it when |
|---------|-------------|
| [Pods](#pods) | Create a Pod with a restart policy, port, or env var |
| [Namespaces](#namespaces) | Create one, or work inside one |
| [Deployments](#deployments) | Create, scale, change image, rollout history, undo |
| [Autoscaling](#autoscaling) | Set an HPA with min, max, CPU target |
| [Jobs and CronJobs](#jobs-and-cronjobs) | Schedule work, or run a CronJob right now |
| [Labels](#labels) | Add, change, or filter by label, including OR |
| [Secrets](#secrets) | Create from a literal, mount as a volume |
| [ConfigMaps](#configmaps) | Create from literals, expose as env vars |
| [ServiceAccounts](#serviceaccounts) | Create one, attach it to a Deployment |
| [Update an existing object](#update-an-existing-object) | `kubectl set`, `patch`, `replace --force`, and what merges |
| [Security context](#security-context) | `runAsUser`, `fsGroup`, and where each belongs |
| [Resources](#resources) | Memory and CPU requests and limits |
| [Multi-container Pods](#multi-container-pods) | Add a container, share files, talk over localhost |
| [Volumes and PersistentVolumes](#volumes-and-persistentvolumes) | hostPath, emptyDir, PV and PVC, binding rules |
| [Probes](#probes) | Liveness and readiness blocks |
| [Services](#services) | Expose a Pod or Deployment, ClusterIP, NodePort, fixed nodePort |
| [Troubleshooting a dead Service](#troubleshooting-a-dead-service) | A Service answers nothing |
| [NetworkPolicies](#networkpolicies) | Allow or block traffic between Pods |
| [Logs across many Pods](#logs-across-many-pods) | Merge logs by label, count rows |
| [Get a file out of a container](#get-a-file-out-of-a-container) | `exec -- cat` against `kubectl cp` |
| [Resource usage](#resource-usage) | Find the highest CPU or memory Pod |
| [Output and JSONPath](#output-and-jsonpath) | Pull one field, sort, one name per line |
| [Generate a manifest](#generate-a-manifest) | `--dry-run=client -o yaml` for any resource |
| [Fields with no imperative flag](#fields-with-no-imperative-flag) | Decide when to stop and write YAML |
| [Verify your work](#verify-your-work) | Prove the change took effect |
| [Habits that save time](#habits-that-save-time) | Small rules that avoid lost marks |
| [Exams covered](#exams-covered) | Trace an entry back to its exam |

Section titles are the words to search for. Entry tags like `[E05 t3]` mean exam 05, task 3.

## Pods
<sup>[↑ contents](#contents)</sup>

```bash
# create a Pod with a restart policy and a TCP port                  [E01 t1, E06 t1]
kubectl run -n <ns> <pod> --image=<image> --restart=OnFailure --port=80

# create a Pod with labels and a shell command                            [E01 t2]
kubectl run -n <ns> <pod> --image=busybox --restart=Never \
  --labels="k1=v1,k2=v2" \
  -- /bin/sh -c "echo hello && sleep 3600"

# create a Pod with an environment variable                               [E01 t5]
kubectl run -n <ns> <pod> --image=<image> --env key=value
```

`kubectl run` defaults to `--restart=Always`. Set it explicitly when the task asks for `Never` or `OnFailure`.

Text after `--` becomes container `args` and replaces the image CMD. Add `--command` to replace the ENTRYPOINT instead.

`kubectl run` always creates **one** container. Two containers means YAML from the start.

`--port=80` writes `containerPort: 80`, and TCP is the default protocol, so "open to TCP" needs nothing extra.

## Namespaces
<sup>[↑ contents](#contents)</sup>

```bash
# create                                                                  [E01 t2]
kubectl create ns <ns>

# work inside one without typing -n every time
kubectl config set-context --current --namespace=<ns>
```

## Deployments
<sup>[↑ contents](#contents)</sup>

```bash
# create with image and replica count in one line                         [E05 t1]
kubectl create deployment <name> -n <ns> --image=<image> --replicas=2

kubectl scale deployment <name> -n <ns> --replicas=4                   # [E05 t1]
kubectl set image deployment <name> <container>=<image> -n <ns>        # [E05 t1]

# fill the CHANGE-CAUSE column, --record no longer exists                 [E05 t3]
kubectl annotate deployment <name> -n <ns> \
  kubernetes.io/change-cause="set image to <image>" --overwrite=true

kubectl rollout status deployment <name> -n <ns>
kubectl rollout history deployment <name> -n <ns>
kubectl rollout undo deployment <name> -n <ns> --to-revision=2
```

The container is named after the image, so `--image=nginx:1.17.8` gives a container called `nginx`.

Scaling makes no new revision. Only pod template changes do.

## Autoscaling
<sup>[↑ contents](#contents)</sup>

```bash
# a top-level verb, not part of kubectl create                            [E05 t4]
kubectl autoscale deployment <name> -n <ns> --min=2 --max=4 --cpu-percent=65
kubectl get hpa -n <ns>
```

Needs metrics-server, and the containers need CPU **requests**. Utilisation is a percentage of the
request, so with no request there is nothing to measure against.

## Jobs and CronJobs
<sup>[↑ contents](#contents)</sup>

```bash
# the most deeply nested manifest in CKAD, so use the command             [E05 t5]
kubectl create cronjob <name> -n <ns> --image=<image> --schedule='*/10 * * * *' -- <command>

kubectl create job <name> --image=<image> -- <command>
kubectl create job manual --from=cronjob/<name>          # run a CronJob right now
```

Quote the schedule. The shell would expand `*/10 * * * *`.

## Labels
<sup>[↑ contents](#contents)</sup>

```bash
# change a label on a running object, no restart                          [E01 t3]
kubectl label pod -n <ns> <pod> key=newvalue --overwrite

# relabel every object matching a selector, one command                   [E05 t2]
kubectl label pods -n <ns> --selector env=prod app=cloudacademy

# set labels at creation, -l is short for --labels                        [E02 t4]
kubectl run <pod> -n <ns> --image=nginx -l env=prod,type=processor

# filter by label
kubectl get pods -l key=value
kubectl get pods -l 'colour in (orange,red,yellow)'      # set-based, the closest to OR  [E05 t6]
kubectl get pods -l 'tier notin (frontend)'
kubectl get pods -l 'partition'                          # key exists, any value
kubectl get pods --show-labels
```

`--overwrite` is required when the key already exists.

`--selector` works with `label`, `delete`, `describe`, and `logs`, not only `get`. Spaces after the
commas inside `in (...)` are valid, the Kubernetes docs use them.

Labels the generators add, and the ones a Service selector must match: `kubectl run` writes
`run=<pod-name>`, `kubectl create deployment` writes `app=<deployment-name>`.

## Secrets
<sup>[↑ contents](#contents)</sup>

```bash
# create from a literal, no base64 by hand                                [E02 t1]
kubectl create secret generic <name> -n <ns> --from-literal=password=<value>

# other types
kubectl create secret docker-registry <name> --docker-server= --docker-username= --docker-password=
kubectl create secret tls <name> --cert=path --key=path

# read a value back
kubectl get secret <name> -n <ns> -o jsonpath='{.data.password}' | base64 -d
```

Writing a Secret as YAML? Use `stringData`, not `data`, and skip the encoding.

Mount as a volume (no flag, YAML only):

```yaml
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
```

## ConfigMaps
<sup>[↑ contents](#contents)</sup>

```bash
# create from literals, repeat the flag per pair                          [E02 t5]
kubectl create configmap <name> -n <ns> --from-literal COLOUR=red --from-literal SPEED=fast

# from a file or a whole directory
kubectl create configmap <name> --from-file=path
```

Consume every key as environment variables (no flag, YAML only):

```yaml
    envFrom:
    - configMapRef:
        name: config1
```

Use `env` with `configMapKeyRef` when you need one key or a different variable name.

## ServiceAccounts
<sup>[↑ contents](#contents)</sup>

```bash
# create                                                                  [E02 t2]
kubectl create sa <name> -n <ns>

# attach to an existing Deployment, one command, no patch file            [E02 t2]
kubectl set serviceaccount deployment <name> <serviceaccount> -n <ns>

# check the Pods really picked it up
kubectl get pods -n <ns> -o jsonpath='{.items[*].spec.serviceAccountName}'
```

## Update an existing object
<sup>[↑ contents](#contents)</sup>

```bash
# kubectl set covers these subcommands
kubectl set image deployment/<name> <container>=<image>
kubectl set env deployment/<name> KEY=value
kubectl set env --from=configmap/<name> deployment/<name>
kubectl set resources deployment/<name> --requests=memory=100Mi --limits=memory=200Mi
kubectl set serviceaccount deployment <name> <sa>
kubectl set selector service <name> 'key=value'          # replaces the whole selector  [E06 t4]

# anything kubectl set does not cover                                     [E02 t2]
kubectl patch deployment <name> -n <ns> --patch '<yaml or json>'

# immutable field, so delete and create in one command                    [E03 t1]
kubectl replace --force -f pod.yaml
```

`kubectl set` works on objects with a **pod template**, so Deployments yes, bare Pods no.

Know what a patch does before you send one. `kubectl patch` merges maps and replaces lists that have
no merge key.

| Field | A patch will | Why it matters |
|-------|--------------|----------------|
| `service.spec.selector` (map) | merge the keys | old keys stay, and a selector is an AND, so it can still match nothing  [E06 t4] |
| `service.spec.ports` (merge key `port`) | update the matching entry | safe, `targetPort` and the port name survive  [E06 t3] |
| `networkpolicy.spec.ingress` (no merge key) | replace the list | the whole rule set is whatever you send  [E06 t5] |

`kubectl get pod -o yaml` output is noisy but the API accepts it. Edit the `spec` and ignore the
rest. `kubectl edit` shows a cleaner manifest to copy from.

## Security context
<sup>[↑ contents](#contents)</sup>

No imperative flag. Generate, then edit. `fsGroup` is pod level, `runAsUser` works at both and the container value wins.

```yaml
spec:
  securityContext:
    fsGroup: 3000
  containers:
  - name: c1
    securityContext:
      runAsUser: 1000
```

```bash
# prove it took effect                                                    [E02 t3]
kubectl exec -n <ns> <pod> -c c1 -- id
```

## Resources
<sup>[↑ contents](#contents)</sup>

No `kubectl run` flag for requests or limits. Checked on kubectl v1.36.3.

```bash
# generate, then add the block by hand                                    [E02 t4]
kubectl run web1 -n ca100 --image=nginx -l env=prod --port=80 --dry-run=client -o yaml > pod.yaml
```

```yaml
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
```

## Multi-container Pods
<sup>[↑ contents](#contents)</sup>

A running Pod cannot gain a container. Dump, edit, recreate.

```bash
# dump, edit, recreate in one command instead of delete then create        [E03 t1]
kubectl -n <ns> get pod <pod> -o yaml > pod.yaml
kubectl -n <ns> replace --force -f pod.yaml

# read one container's logs
kubectl logs -n <ns> <pod> -c <container>
```

Share files with an `emptyDir` mounted in both containers:

```yaml
spec:
  containers:
  - name: main
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: second
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/random.log']    # -n+1 replays from line 1
    volumeMounts:
    - name: logs
      mountPath: /var/log
      readOnly: true                                            # readOnly is a mount field
  volumes:
  - name: logs
    emptyDir: {}
```

Networking inside a Pod: one network namespace, one IP. From container to container the host is
`localhost`, and these all work too.

```text
127.0.0.1
<pod-name>                                          # Kubernetes writes it into /etc/hosts
<pod-ip>                                            # 10.0.0.11
<pod-ip-with-dashes>.<ns>.pod.cluster.local         # 10-0-0-11.app1.pod.cluster.local
```

Two containers in one Pod cannot bind the same port.

## Volumes and PersistentVolumes
<sup>[↑ contents](#contents)</sup>

Nothing here has an imperative command. Copy a manifest from the docs and change the values. The two
pages worth bookmarking are [Configure a Pod to Use a Volume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
and [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/).

One volume, two containers: one entry in `spec.volumes`, one `volumeMounts` per container, same name.

```yaml
spec:
  containers:
  - name: c1
    volumeMounts:
    - name: vol1
      mountPath: /var/log/blah
  - name: c2
    volumeMounts:
    - name: vol1
      mountPath: /var/log/blah
  volumes:
  - name: vol1
    hostPath:
      path: /tmp/vol            # no type = no checks, the safe default   [E07 t2]
    # emptyDir: {}              # scratch space, dies with the Pod
    # persistentVolumeClaim:
    #   claimName: pvc          # the Pod names the claim, never the PV
```

A static PV and the claim that binds to it:

```yaml
apiVersion: v1
kind: PersistentVolume          # cluster scoped, so no namespace         [E07 t1]
metadata:
  name: pv
spec:
  storageClassName: host
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim     # namespaced
metadata:
  name: pvc
spec:
  storageClassName: host
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl -n <ns> get pv,pvc                  # both must read Bound        [E07 t1]
kubectl api-resources --namespaced=false    # what has no namespace
kubectl explain persistentvolume.spec       # field names
```

A claim binds when three things line up: equal `storageClassName`, an access mode the PV offers, and
PV capacity at least the request. Pin a claim to one PV with `spec.volumeName: pv`.

`ReadWriteOnce` is one **node**. `ReadWriteOncePod` is one **Pod**. `ReadOnlyMany` and
`ReadWriteMany` are many nodes. Tasks say "a single Node", which means `ReadWriteOnce`.

hostPath `type: Directory` makes the kubelet require the directory on the node, and the Pod stays in
`ContainerCreating` if it is missing. `DirectoryOrCreate` creates it. Leave `type` out if the task
does not name it.

## Probes
<sup>[↑ contents](#contents)</sup>

No imperative flag. Generate, then add the block to the container.

```yaml
    livenessProbe:            # fails -> container restarts
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10 # wait before the first check, default 0
      periodSeconds: 5        # gap between checks, default 10
    readinessProbe:           # fails -> Pod drops out of the Service endpoints
      httpGet:
        port: 80
```

```bash
# read the probe back as one line                                         [E04 t1]
kubectl describe pod -n <ns> <pod>
# Liveness: http-get http://:80/ delay=10s timeout=1s period=5s #success=1 #failure=3
```

## Services
<sup>[↑ contents](#contents)</sup>

```bash
# expose an existing object, the selector is copied from its labels        [E06 t2]
kubectl expose pod <pod> -n <ns> --name=<svc> --port=8080 --target-port=80
kubectl expose deployment <name> -n <ns> --name=<svc> --type=NodePort --port=80   # [E06 t3]

# expose has no --node-port flag, so patch the port in afterwards          [E06 t3]
kubectl patch svc <svc> -n <ns> --patch '{"spec": {"ports": [{"port": 80, "nodePort": 32080}]}}'

# nothing to expose? create service always writes selector app=<svc>, so fix it  [E06 t2]
kubectl create service clusterip <svc> -n <ns> --tcp=8080:80
kubectl set selector service <svc> -n <ns> 'run=<pod>'

# call it from a throwaway Pod, works when the target image has no shell tools
kubectl run client -n <ns> --image=curlimages/curl -it --rm --restart=Never -- curl http://<svc>:8080
```

Three ports, three addresses. `port` answers on the ClusterIP and the DNS name, `nodePort` on every
node IP, `targetPort` on the container. Curling the Service name on the nodePort always fails.

`--target-port` copies `--port` when you leave it out. ClusterIP is the default `--type`.

`--rm` only deletes the client Pod if you also pass `-it`.

## Troubleshooting a dead Service
<sup>[↑ contents](#contents)</sup>

Work in this order. Stop as soon as something looks wrong.

```bash
kubectl -n <ns> get endpoints                          # empty means no READY Pods  [E04 t2, E06 t4]
kubectl -n <ns> get service -o wide                    # check the selector
kubectl -n <ns> get pods --selector app=<label>        # do Pods match it
kubectl -n <ns> describe pods --selector app=<label>   # Events name the cause
kubectl -n <ns> edit deployment <name>                 # fix it in place
kubectl -n <ns> set selector service <svc> 'app=<label>'   # replace a wrong selector  [E06 t4]
```

Two causes empty the endpoints list, and they need different fixes. Either the selector matches no
Pod, or it matches Pods that are not Ready. A Service never routes to a Pod that is not Ready, and a
probe port that does not match the container port is the classic reason.

The Service selector must match the labels on the **Pods**, at `spec.template.metadata.labels` in a
Deployment, not the labels on the Deployment object.

```bash
# test from inside the cluster using Service DNS, no node IP lookup needed
kubectl -n <ns> exec -it <any-pod> -- curl <service>:80
# from outside, for a NodePort Service
kubectl get nodes -o wide && curl <NodeIP>:<NodePort>
```

## NetworkPolicies
<sup>[↑ contents](#contents)</sup>

No imperative command. Edit the object, or write the YAML.

```yaml
spec:
  podSelector:          # who is protected
    matchLabels:
      app: test
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:      # who may send
        matchLabels:
          app: client
```

```bash
kubectl -n <ns> edit netpol <name>                                         # [E06 t5]
kubectl -n <ns> describe netpol <name>       # the rules as readable text

# test Pod to Pod with the IP, because a Pod name is not a DNS name
pod1IP=$(kubectl get pod pod1 -n <ns> -o jsonpath='{.status.podIP}')
kubectl -n <ns> exec -it pod2 -- ping $pod1IP
```

Two selectors doing opposite jobs. `spec.podSelector` picks the Pods the policy protects,
`ingress.from.podSelector` picks who may send to them. A task that says "keep it applied to pod1"
is telling you not to touch `spec.podSelector`.

Both selectors are namespace scoped. Allowing another namespace needs `namespaceSelector`.

## Logs across many Pods
<sup>[↑ contents](#contents)</sup>

```bash
kubectl logs -n <ns> -l app=prod                # merges every matching Pod  [E04 t3]
kubectl logs -n <ns> -l app=prod --prefix       # tag each line with its Pod
kubectl logs -n <ns> -l app=prod | wc -l        # count rows, no loop, no temp file
kubectl logs -n <ns> <pod> -c <container>       # one container
kubectl logs -n <ns> <pod> --previous           # the crashed instance
```

`--max-log-requests` defaults to 5 and caps concurrent streams when following by selector.

Never loop over Pods for this. `for x in $(kubectl logs ...)` splits on **whitespace**, not
newlines, so multi-word lines get counted more than once.

## Get a file out of a container
<sup>[↑ contents](#contents)</sup>

```bash
kubectl exec -n <ns> <pod> -- cat /path/file > local.txt   # text, no dependencies  [E04 t4]
kubectl cp <ns>/<pod>:path/file local.txt                  # binaries and directories
```

`kubectl cp` runs `tar` **inside** the container. Slim images often have no `tar`, and it strips a
leading `/` from the source path with a warning.

Alpine based images are missing more than `tar`. `curl` is usually absent too, so
`kubectl exec -- curl` dies with `executable file not found` and tells you nothing about the Pod.

## Resource usage
<sup>[↑ contents](#contents)</sup>

```bash
kubectl top pods -n <ns> --sort-by=cpu --no-headers | head -n1 | awk '{print $1}'   # [E04 t5]
kubectl top pods -n <ns> --sort-by=memory
kubectl top nodes
```

`--sort-by` takes only `cpu` or `memory`, and sorts descending. Needs metrics-server.

Prefer `awk '{print $1}'` over `cut -d" " -f1`. Column widths vary and awk treats a run of spaces
as one separator.

## Output and JSONPath
<sup>[↑ contents](#contents)</sup>

```bash
# inspect the structure first                                             [E01 t4]
kubectl get pod -n <ns> <pod> -o json

# then pull one field
kubectl get pod -n <ns> <pod> -o jsonpath='{.status.podIP}'

# common paths
'{.status.podIP}'                                    # Pod IP
'{.spec.clusterIP}'                                  # Service cluster IP
'{.status.loadBalancer.ingress[0].hostname}'         # load balancer DNS name
'{.items[0].metadata.name}'                          # first item in a list
'{.items[*].spec.serviceAccountName}'                # one field across all Pods
'{.data.password}'                                   # Secret value, still base64

# one name per line, sorted by a field                                    [E05 t6]
kubectl get pods -n <ns> --sort-by=.status.podIP \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# same thing without JSONPath, but it prints a header row
kubectl get pods -n <ns> --sort-by=.status.podIP -o custom-columns="NAME:.metadata.name"
```

`--sort-by` takes a JSONPath **without** braces and sorts as text, so IPs sort lexicographically.

Find a path with `kubectl get pod <name> -o yaml` or by walking `kubectl explain pod.status`.
Anything the cluster assigns, such as an IP, lives under `status`.

## Generate a manifest
<sup>[↑ contents](#contents)</sup>

```bash
# manifest only, no cluster contact                                       [E01 t5]
kubectl run <pod> --image=<image> --dry-run=client -o yaml > pod.yaml

# same for other resources
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl create job <name> --image=<image> --dry-run=client -o yaml
kubectl create cronjob <name> --image=<image> --schedule='* * * * *' --dry-run=client -o yaml
kubectl create configmap <name> --from-literal K=V --dry-run=client -o yaml
kubectl create secret generic <name> --from-literal K=V --dry-run=client -o yaml
kubectl create service clusterip <name> --tcp=8080:80 --dry-run=client -o yaml
kubectl expose pod <pod> --port=80 --dry-run=client -o yaml
kubectl autoscale deployment <name> --min=2 --max=4 --cpu-percent=65 --dry-run=client -o yaml
```

`--image` is required for `kubectl run`, even with `--dry-run`.

If the dry run already prints what you want, **drop `--dry-run` and run it**. Do not retype the
output into a manifest.

## Fields with no imperative flag
<sup>[↑ contents](#contents)</sup>

Generate, edit the one field, apply.

| Field | Where it goes | Seen in |
|-------|---------------|---------|
| `terminationGracePeriodSeconds` | `spec` | E01 t6 |
| `resources.requests`, `resources.limits` | container | E02 t4 |
| `securityContext.fsGroup` | `spec` | E02 t3 |
| `securityContext.runAsUser` | `spec` or container | E02 t3 |
| `volumes`, `volumeMounts` | `spec` and container | E02 t1, E03 t1, E07 t2 |
| a PersistentVolume and its claim | their own objects | E07 t1 |
| `envFrom` | container | E02 t5 |
| a second container | `spec.containers` | E02 t3, E03 t1 |
| `lifecycle.postStart` | container | E03 t2 |
| `livenessProbe`, `readinessProbe` | container | E04 t1 |
| `ports[].nodePort` | Service `spec` | E06 t3 |
| the whole NetworkPolicy | its own object | E06 t5 |

Confirm nesting with `kubectl explain pod.spec` before editing.

## Verify your work
<sup>[↑ contents](#contents)</sup>

```bash
kubectl describe pod -n <ns> <pod>       # labels, ports, requests, limits, events
kubectl exec -n <ns> <pod> -c <c> -- id  # user and group IDs
kubectl logs -n <ns> <pod>               # environment variables the command printed
kubectl logs -n <ns> <pod> -c <c>        # one container of a multi-container Pod
kubectl get pods -n <ns>                 # it actually started
kubectl get ep -n <ns> <svc>             # a Service really found its Pods
kubectl get svc -n <ns> <svc>            # PORT(S) shows 80:32080/TCP for a fixed nodePort
```

Check inside the container, not just the manifest. The API accepts fields that do not behave the way you expect.

## Habits that save time
<sup>[↑ contents](#contents)</sup>

- Read the verb. "Generate a manifest" is not "create the Pod".
- Add nothing the task did not ask for. Every extra field is unpaid risk.
- Use `>` not `>>` when a task says save something to a file.
- When a task hands you a command to run, run it **after** the fix. Its output is what gets graded.
- Tab completion is enabled on the lab hosts. Use it instead of reading `--help`.
- `kubectl explain <resource>.<field>` beats searching the docs for field names.
- Combine a heredoc with `kubectl apply -f -` to paste a manifest without opening an editor.

## Exams covered
<sup>[↑ contents](#contents)</sup>

| Exam | Domain |
|------|--------|
| E01 | [Core Concepts](01-core-concepts.md) |
| E02 | [Configuration](02-configuration.md) |
| E03 | [Multi-Container Pods](03-multi-container-pods.md) |
| E04 | [Observability](04-observability.md) |
| E05 | [Pod Design](05-pod-design.md) |
| E06 | [Services and Networking](06-services-and-networking.md) |
| E07 | [State Persistence](07-state-persistence.md) |
