# 01 - Pod Definition Basics

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Kubernetes Pod Design for Application Developers: Definition Basics  
> Lab: Reviewing Pod Definition Basics  
> Topics: pods, yaml, manifests, kubectl-explain, kubectl-get, kubectl-create, kubectl-delete

## Goal

Review the basics of Pod configuration with a minimal YAML manifest. Practice core `kubectl` commands for create, inspect, and delete. Prefer higher-level controllers (such as Deployments) in real work; this lab uses a standalone Pod to learn the basics.

## Notes

- A Pod is one or more containers that share the same network space.
- The minimal manifest uses only required fields. Usual httpd fields (such as container ports) are omitted on purpose.
- Use `kubectl explain <Resource_Kind>.<Path_To_Field>` for field help. Example: `kubectl explain Pod.spec.containers.image`.
- You can delete with the resource name or with `-f` and the manifest file.

## Steps

### Step 1 - Review Pod definition basics

1. Create a simple Pod manifest:

```bash
cat << 'EOF' > first-pod.yaml
apiVersion: v1            # The API path for the Pod resource
kind: Pod                 # The kind of resource (Pod)
metadata:
  name: first-pod         # Name of the Pod
spec:
  containers:             # List of containers in the Pod
  - image: httpd:2.4.38   # Container image (using a tag to specify version 2.4.38)
    name: first-container # Name of the container
EOF
```

Read the comments (after `#`). All fields in this file are required to create a Pod.

2. Explain the Pod `spec` (press space to page):

```bash
kubectl explain Pod.spec | more
```

3. Create the Pod:

```bash
kubectl create -f first-pod.yaml
```

4. Get a summary of Pods in the default namespace:

```bash
kubectl get pod
```

For one Pod: `kubectl get pod first-pod`.

- **READY:** how many containers in the Pod are ready
- **STATUS:** `Running` when all containers were created and at least one is still running

5. Show all fields as YAML:

```bash
kubectl get pod first-pod -o yaml | more
```

Kubernetes adds many fields (for example `resourceVersion`, `selfLink`, `uid`, and `status`). Fields under `spec` are what you declare in manifests.

6. Delete the Pod:

```bash
kubectl delete pod first-pod
```

Alternative:

```bash
kubectl delete -f first-pod.yaml
```

## Summary

You reviewed how to declare Pods and used basic `kubectl` commands to work with them.
