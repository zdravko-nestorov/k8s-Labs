# Exam Cheat Sheet

Fastest known command for each task type met in the practice exams. Grows with every new exam.

Read this before an attempt. Every entry cites the exam it came from.

## Pods

```bash
# create a Pod with a restart policy and a TCP port                       [E01 t1]
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

## Namespaces

```bash
# create                                                                  [E01 t2]
kubectl create ns <ns>

# work inside one without typing -n every time
kubectl config set-context --current --namespace=<ns>
```

## Labels

```bash
# change a label on a running object, no restart                          [E01 t3]
kubectl label pod -n <ns> <pod> key=newvalue --overwrite

# set labels at creation, -l is short for --labels                        [E02 t4]
kubectl run <pod> -n <ns> --image=nginx -l env=prod,type=processor

# filter by label
kubectl get pods -l key=value
```

`--overwrite` is required when the key already exists.

## Secrets

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

```bash
# create                                                                  [E02 t2]
kubectl create sa <name> -n <ns>

# attach to an existing Deployment, one command, no patch file            [E02 t2]
kubectl set serviceaccount deployment <name> <serviceaccount> -n <ns>

# check the Pods really picked it up
kubectl get pods -n <ns> -o jsonpath='{.items[*].spec.serviceAccountName}'
```

## Update an existing object

```bash
# kubectl set covers these subcommands
kubectl set image deployment/<name> <container>=<image>
kubectl set env deployment/<name> KEY=value
kubectl set env --from=configmap/<name> deployment/<name>
kubectl set resources deployment/<name> --requests=memory=100Mi --limits=memory=200Mi
kubectl set serviceaccount deployment <name> <sa>
kubectl set selector service <name> 'key=value'

# anything kubectl set does not cover                                     [E02 t2]
kubectl patch deployment <name> -n <ns> --patch '<yaml or json>'

# immutable field, so delete and create in one command                    [E03 t1]
kubectl replace --force -f pod.yaml
```

`kubectl set` works on objects with a **pod template**, so Deployments yes, bare Pods no.

`kubectl get pod -o yaml` output is noisy but the API accepts it. Edit the `spec` and ignore the
rest. `kubectl edit` shows a cleaner manifest to copy from.

## Security context

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

## Probes

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

## Troubleshooting a dead Service

Work in this order. Stop as soon as something looks wrong.

```bash
kubectl -n <ns> get endpoints                          # empty means no READY Pods  [E04 t2]
kubectl -n <ns> get service -o wide                    # check the selector
kubectl -n <ns> get pods --selector app=<label>        # do Pods match it
kubectl -n <ns> describe pods --selector app=<label>   # Events name the cause
kubectl -n <ns> edit deployment <name>                 # fix it in place
```

A Service never routes to a Pod that is not Ready, so a failing readiness probe empties the
endpoints list. A probe port that does not match the container port is the classic cause.

```bash
# test from inside the cluster using Service DNS, no node IP lookup needed
kubectl -n <ns> exec -it <any-pod> -- curl <service>:80
# from outside, for a NodePort Service
kubectl get nodes -o wide && curl <NodeIP>:<NodePort>
```

## Logs across many Pods

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

```bash
kubectl exec -n <ns> <pod> -- cat /path/file > local.txt   # text, no dependencies  [E04 t4]
kubectl cp <ns>/<pod>:path/file local.txt                  # binaries and directories
```

`kubectl cp` runs `tar` **inside** the container. Slim images often have no `tar`, and it strips a
leading `/` from the source path with a warning.

## Resource usage

```bash
kubectl top pods -n <ns> --sort-by=cpu --no-headers | head -n1 | awk '{print $1}'   # [E04 t5]
kubectl top pods -n <ns> --sort-by=memory
kubectl top nodes
```

`--sort-by` takes only `cpu` or `memory`, and sorts descending. Needs metrics-server.

Prefer `awk '{print $1}'` over `cut -d" " -f1`. Column widths vary and awk treats a run of spaces
as one separator.

## Output and JSONPath

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
```

## Generate a manifest

```bash
# manifest only, no cluster contact                                       [E01 t5]
kubectl run <pod> --image=<image> --dry-run=client -o yaml > pod.yaml

# same for other resources
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl create job <name> --image=<image> --dry-run=client -o yaml
kubectl create configmap <name> --from-literal K=V --dry-run=client -o yaml
kubectl create secret generic <name> --from-literal K=V --dry-run=client -o yaml
kubectl expose pod <pod> --port=80 --dry-run=client -o yaml
```

`--image` is required for `kubectl run`, even with `--dry-run`.

## Fields with no imperative flag

Generate, edit the one field, apply.

| Field | Where it goes | Seen in |
|-------|---------------|---------|
| `terminationGracePeriodSeconds` | `spec` | E01 t6 |
| `resources.requests`, `resources.limits` | container | E02 t4 |
| `securityContext.fsGroup` | `spec` | E02 t3 |
| `securityContext.runAsUser` | `spec` or container | E02 t3 |
| `volumes`, `volumeMounts` | `spec` and container | E02 t1, E03 t1 |
| `envFrom` | container | E02 t5 |
| a second container | `spec.containers` | E02 t3, E03 t1 |
| `lifecycle.postStart` | container | E03 t2 |
| `livenessProbe`, `readinessProbe` | container | E04 t1 |

Confirm nesting with `kubectl explain pod.spec` before editing.

## Verify your work

```bash
kubectl describe pod -n <ns> <pod>       # labels, ports, requests, limits, events
kubectl exec -n <ns> <pod> -c <c> -- id  # user and group IDs
kubectl logs -n <ns> <pod>               # environment variables the command printed
kubectl logs -n <ns> <pod> -c <c>        # one container of a multi-container Pod
kubectl get pods -n <ns>                 # it actually started
```

Check inside the container, not just the manifest. The API accepts fields that do not behave the way you expect.

## Habits that save time

- Read the verb. "Generate a manifest" is not "create the Pod".
- Use `>` not `>>` when a task says save something to a file.
- Tab completion is enabled on the lab hosts. Use it instead of reading `--help`.
- `kubectl explain <resource>.<field>` beats searching the docs for field names.
- Combine a heredoc with `kubectl apply -f -` to paste a manifest without opening an editor.

## Exams covered

| Exam | Domain |
|------|--------|
| E01 | [Core Concepts](01-core-concepts.md) |
| E02 | [Configuration](02-configuration.md) |
| E03 | [Multi-Container Pods](03-multi-container-pods.md) |
| E04 | [Observability](04-observability.md) |
