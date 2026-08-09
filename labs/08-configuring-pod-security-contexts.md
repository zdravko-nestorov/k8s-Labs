# 08 - Configuring Pod Security Contexts

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Mastering Kubernetes Pod Configuration: Security Contexts  
> Lab: Configuring Pod Security Contexts  
> Topics: securityContext, privileged, runAsUser, runAsGroup, runAsNonRoot, readOnlyRootFilesystem, seLinux

## Goal

Review Pod-level and container-level security contexts. Compare a normal Pod, a privileged Pod, and a hardened Pod with `runAsUser` / read-only root filesystem.

## Notes

- A **security context** sets access control for Pods, and for containers and volumes when it applies.
- Examples: user/group IDs, volume group ID, read-only root filesystem, SELinux options, privileged mode, allow privilege escalation.
- If the same field is set on the Pod and on a container, the **container** setting wins.
- Hardening Pod/container security contexts lowers risk from third-party images.
- Prefer non-root and read-only root filesystems. Mount volumes for paths that must be writable.
- Privileged containers see host devices (including disks). Use them only when you truly need them.

## Steps

### Step 1 - Configure Pod security contexts

1. Explain Pod-level security context fields:

```bash
kubectl explain pod.spec.securityContext | more
```

2. Explain container-level security context fields:

```bash
kubectl explain pod.spec.containers.securityContext | more
```

3. Create a Pod with no security context (sleep only):

```bash
cat << EOF > pod-no-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-test-1
spec:
  containers:
  - image: busybox:1.30.1
    name: busybox
    args:
    - sleep
    - "3600"
EOF
```

4. Create the Pod and list devices:

```bash
kubectl create -f pod-no-security-context.yaml
kubectl exec security-context-test-1 -- ls /dev
```

Note: If you get "container not found", wait 10-20 seconds and retry.

Expect a small device set. No host block devices such as `nvme0n1p1`.

5. Delete it and create a privileged Pod:

```bash
kubectl delete -f pod-no-security-context.yaml
cat > pod-privileged.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: security-context-test-2
spec:
  containers:
  - image: busybox:1.30.1
    name: busybox
    args:
    - sleep
    - "3600"
    securityContext:
      privileged: true
EOF
kubectl create -f pod-privileged.yaml
```

6. List devices again:

```bash
kubectl exec security-context-test-2 -- ls /dev
```

Host devices appear, including the host disk (`nvme0n1p1`). That is a serious risk if misused.

7. Delete it and create a Pod with Pod + container security contexts:

```bash
kubectl delete -f pod-privileged.yaml
cat << EOF > pod-runas.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-test-3
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - image: busybox:1.30.1
    name: busybox
    args:
    - sleep
    - "3600"
    securityContext:
      runAsUser: 2000
      readOnlyRootFilesystem: true
EOF
kubectl create -f pod-runas.yaml
```

Pod level: non-root, UID/GID 1000. Container level: UID 2000 and read-only root filesystem.

8. Open a shell:

```bash
kubectl exec security-context-test-3 -it -- /bin/sh
```

Prompt is `$` (not `#`) → not root.

9. List processes:

```bash
ps
```

USER is 2000. Container `runAsUser` overrides the Pod value.

10. Try to write under `/tmp`, then exit:

```bash
touch /tmp/test-file
exit
```

Fails with a read-only filesystem error.

11. Delete the Pod:

```bash
kubectl delete -f pod-runas.yaml
```

## Summary

Pod and container security contexts control privilege and identity. Avoid privileged and root when you can. Prefer read-only root filesystems and volumes for writable data.
