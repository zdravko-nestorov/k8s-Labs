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

# filter by label
kubectl get pods -l key=value
```

`--overwrite` is required when the key already exists.

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
'{.data.password}'                                   # Secret value, still base64
```

## Generate a manifest

```bash
# manifest only, no cluster contact                                       [E01 t5]
kubectl run <pod> --image=<image> --dry-run=client -o yaml > pod.yaml

# same for other resources
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl create job <name> --image=<image> --dry-run=client -o yaml
kubectl expose pod <pod> --port=80 --dry-run=client -o yaml
```

`--image` is required for `kubectl run`, even with `--dry-run`.

## Fields with no imperative flag

Generate, edit the one field, apply.

| Field | Where it goes | Seen in |
|-------|---------------|---------|
| `terminationGracePeriodSeconds` | `spec` | E01 t6 |

Confirm nesting with `kubectl explain pod.spec` before editing.

## Habits that save time

- Read the verb. "Generate a manifest" is not "create the Pod".
- Use `>` not `>>` when a task says save something to a file.
- Tab completion is enabled on the lab hosts. Use it instead of reading `--help`.
- `kubectl explain <resource>.<field>` beats searching the docs for field names.

## Exams covered

| Exam | Domain |
|------|--------|
| E01 | [Core Concepts](01-core-concepts.md) |
