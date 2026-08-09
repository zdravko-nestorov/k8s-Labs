# 11 - Using Kubernetes ServiceAccounts

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Mastering Kubernetes Pod Configuration: Service Accounts  
> Lab: Using Kubernetes ServiceAccounts  
> Topics: serviceaccount, rbac, identity, least-privilege, projected-volume

## Goal

See how Pods get a cluster identity via ServiceAccounts. Compare the default ServiceAccount with a custom one on a Pod.

## Notes

- **ServiceAccount:** identity for Pods. Pods authenticate as that account and get only the API access bound to it.
- Admins create **roles** and **bindings** (RBAC). This lab focuses on ServiceAccounts, not role setup.
- Every Namespace has a **default** ServiceAccount with minimal access. Prefer a custom ServiceAccount per app (**least privilege**).
- Set `spec.serviceAccount` on the Pod (or `--serviceaccount` with `kubectl run`).
- On create, Kubernetes mounts a projected volume with credentials so the Pod can talk to the API server.
- Quick manifest tip:  
  `kubectl run ... --serviceaccount=app-sa --dry-run=client -o yaml`

## Steps

### Step 1 - Use ServiceAccounts

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace serviceaccounts
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=serviceaccounts
```

2. List ServiceAccounts:

```bash
kubectl get serviceaccounts
```

You should see `default`. It cannot read useful cluster state. Use a custom ServiceAccount when the app needs API access.

3. Create a Pod and inspect its YAML:

```bash
kubectl run default-pod --image=mongo:4.0.6
kubectl get pod default-pod -o yaml | more
```

`spec.serviceAccount` is set to `default` automatically.

4. Quit the pager (`q`), then create a custom ServiceAccount:

```bash
kubectl create serviceaccount app-sa
```

No role is bound yet, so no extra permissions. In real clusters an admin binds a role to this account.

5. Create a Pod that uses `app-sa`:

```bash
cat << 'EOF' > pod-custom-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-sa-pod
spec:
  containers:
  - image: mongo:4.0.6
    name: mongodb
  serviceAccount: app-sa
EOF
kubectl create -f pod-custom-sa.yaml
```

6. Confirm the ServiceAccount on the Pod:

```bash
kubectl get pod custom-sa-pod -o yaml | more
```

Expect `app-sa` and a projected volume for API authentication.

## Summary

ServiceAccounts give Pods an identity. Bind roles to grant API access. Prefer one ServiceAccount per app with the least access needed.
