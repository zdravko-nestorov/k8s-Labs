# 02 - Pod Labels, Selectors, and Annotations

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Kubernetes Pod Design for Application Developers: Definition Basics  
> Lab: Working With Pod Labels, Selectors, and Annotations  
> Topics: labels, selectors, annotations, namespaces, kubectl-get, kubectl-annotate, kubectl-label

## Goal

Work with labels, selectors, and annotations on Pods. Labels and selectors apply to all Kubernetes resources, not only Pods.

## Notes

- **Labels:** key-value pairs that identify and organize resources. They do not need to be unique across resources of the same kind.
- **Label selectors:** conditions that select objects by which labels are present or absent, and which values are allowed or not.
- **Annotations:** non-identifying key-value metadata. You cannot select objects by annotations. Clients and extensions often use them.
- Namespaces organize resources more coarsely than labels and work well with labels.
- Multiple resources in one YAML file are separated by `---`.
- Enclose selectors in single quotes so the shell does not misread `!` or other characters.
- `kubectl annotate` and `kubectl label` are similar. Annotation values have fewer restrictions (for example, label values cannot have spaces).
- Append `-` after a key to remove a label or annotation (example: `Lab-`).

## Steps

### Step 1 - Work with labels, selectors, and annotations

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace labels
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=labels
```

2. Create four Pods from a multi-resource manifest:

```bash
# Write the manifest file
cat << 'EOF' > pod-labels.yaml
apiVersion: v1
kind: Pod
metadata:
  name: red-frontend
  namespace: labels # declare namespace in metadata
  labels: # labels mapping in metadata
    color: red
    tier: frontend
  annotations: # Example annotation
    Lab: Kubernetes Pod Design for Application Developers
spec:
  containers:
  - image: httpd:2.4.38
    name: web-server
---
apiVersion: v1
kind: Pod
metadata:
  name: green-frontend
  namespace: labels
  labels:
    color: green
    tier: frontend
spec:
  containers:
  - image: httpd:2.4.38
    name: web-server
---
apiVersion: v1
kind: Pod
metadata:
  name: red-backend
  namespace: labels
  labels:
    color: red
    tier: backend
spec:
  containers:
  - image: postgres:11.2-alpine
    name: db
---
apiVersion: v1
kind: Pod
metadata:
  name: blue-backend
  namespace: labels
  labels:
    color: blue
    tier: backend
spec:
  containers:
  - image: postgres:11.2-alpine
    name: db
EOF
# Create the Pods
kubectl create -f pod-labels.yaml
```

Each Pod is in the `labels` namespace and has `color` and `tier` labels. Only `red-frontend` has an annotation.

3. Show label columns:

```bash
kubectl get pods -L color,tier
```

If you do not know the labels, use `--show-labels` to list all labels in one column.

4. Select all Pods that have a `color` label:

```bash
kubectl get pods -L color,tier -l color
```

A selector with only a key matches any resource that has that label.

5. Select all Pods that do not have a `color` label:

```bash
kubectl get pods -L color,tier -l '!color'
```

6. Select all Pods with `color=red`:

```bash
kubectl get pods -L color,tier -l 'color=red'
```

Use `=` or `==` for key and value.

7. Select Pods with `color=red` that are not frontend:

```bash
kubectl get pods -L color,tier -l 'color=red,tier!=frontend'
```

Join conditions with commas. `!=` means not equals. The label key must exist for `!=` (Pods with no `tier` label are not selected).

8. Select Pods with green or blue color:

```bash
kubectl get pods -L color,tier -l 'color in (blue,green)'
```

`in` allows a list of values. `notin` disallows a list of values.

9. Show the annotation on `red-frontend`:

```bash
kubectl describe pod red-frontend | grep Annotations
```

You can also use `kubectl get pod red-frontend -o yaml`.

10. Remove the `Lab` annotation and verify:

```bash
kubectl annotate pod red-frontend Lab- &&
kubectl describe pod red-frontend | grep Annotations -A 3
```

Only network-plugin annotations should remain.

11. Review annotate examples:

```bash
kubectl annotate --help
```

12. Review label examples:

```bash
kubectl label --help
```

13. Delete the Pods:

```bash
kubectl delete -f pod-labels.yaml
```

## Summary

You learned the difference between labels and annotations, and how selectors filter resources with label conditions.
