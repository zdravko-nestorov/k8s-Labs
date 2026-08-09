# 18 - API Access Control: Authentication, Authorization, Admission

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Understand Kubernetes API Access Control Mechanisms  
> Lab: Authentication / Authorization / Admission Control  
> Topics: authentication, authorization, admission-control, kubeconfig, contexts, certificates, rbac, roles, clusterroles, rolebinding, clusterrolebinding, serviceaccount, tokens, projected-volume, can-i, impersonation, kube-apiserver

## Goal

Follow a request through the three access control layers of the Kubernetes API: authentication (who are you), authorization (are you allowed), and admission control (does any plugin object).

## Notes

### The three layers

Every request to the cluster goes through the API Server in this order:

1. **Authentication:** proves who sends the request, user or ServiceAccount. Rejected if it fails.
2. **Authorization:** the requested action must be allowed for that identity.
3. **Admission control:** all configured admission plugins must accept the request. Read-only requests skip this layer.

Only then is the object stored in etcd and the controllers act on it.

### Authentication facts

- kubectl reads a **kubeconfig** file, by default `~/.kube/config`.
- The file holds `clusters`, `contexts`, `current-context`, and `users`.
- A **context** is a triple: cluster + user + namespace (default namespace if omitted). It has a name, for example `kubernetes-admin@kubernetes`.
- Client certificate and key are the default authentication method. Passwords and tokens are also supported. A request passes if **any** authentication module accepts it.
- In the client certificate, Kubernetes maps the **CN** (common name) to the user and the **O** (organization) to a group.
- With no valid certificate, requests fall back and end up as `system:anonymous`, which is usually forbidden.
- kubeconfig loading order: `--kubeconfig` flag wins alone, else `$KUBECONFIG` paths are merged, else `~/.kube/config`.

### ServiceAccount authentication

Every Pod gets a **projected volume** at `/var/run/secrets/kubernetes.io/serviceaccount` holding:

- a **token** from kube-apiserver, bound to the Pod, expiring after about 1 hour by default or when the Pod is deleted
- a **ConfigMap** (`kube-root-ca.crt`) with the CA bundle used to verify the API server

### Authorization (RBAC)

| Resource | Scope | Purpose |
|----------|-------|---------|
| `Role` | Namespaced | Allowed actions inside one namespace |
| `ClusterRole` | Cluster-wide | Allowed actions anywhere, including non-namespaced resources such as Nodes |
| `RoleBinding` | Namespaced | Binds a Role or ClusterRole to subjects in one namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to subjects across the cluster |

Subjects are users, groups, or ServiceAccounts.

Rule structure: `apiGroups`, `resources`, `verbs`, and optionally `resourceNames` or `nonResourceURLs`. The **core API group is the empty string** `""`. Follow least privilege.

Why the lab user can do everything: the certificate says group `system:masters`, and the `cluster-admin` ClusterRole is bound to that group by the `cluster-admin` ClusterRoleBinding, with `*` verbs on `*` resources.

Identities live outside Kubernetes, so kubectl cannot show user or group objects.

### Admission control

- A cluster can enable many admission plugins. **No plugin may reject** the request.
- Plugins can also **change** a request. `LimitRanger` can set default CPU and memory limits when a Pod sets none. `ResourceQuota` enforces namespace quotas.
- Default enabled plugins appear in `kube-apiserver -h`. Extra ones appear in the `--enable-admission-plugins` flag on the running process, which you can only see from a control plane node.

---

## Step 1 - Authentication

1. Get Pods in the default namespace:

```bash
kubectl get pods
```

Output: `No resources found in default namespace.`

2. Repeat with verbosity level 6:

```bash
kubectl get pods --v=6
```

Two lines matter:

```text
I0809 12:27:54.051763    1257 loader.go:402] Config loaded from file:  /home/ubuntu/.kube/config
I0809 12:27:54.069746    1257 round_trippers.go:632] "Response" verb="GET" url="https://10.0.0.100:6443/api/v1/namespaces/default/pods?limit=500" status="200 OK" milliseconds=10
```

The first shows the kubeconfig used. The second shows the REST request sent to the API Server.

3. Show the kubeconfig:

```bash
cat /home/ubuntu/.kube/config
```

Structure, with certificate and key values left out:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <base64 CA certificate>
    server: https://10.0.0.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <base64 client certificate>
    client-key-data: <base64 client private key>
```

Treat `client-key-data` as a secret. It is a private key.

4. See the config subcommands:

```bash
kubectl config --help
```

Useful ones: `current-context`, `get-contexts`, `use-context`, `set-context`, `set-credentials`, `view`, and the `delete-*` commands.

5. Summarise contexts:

```bash
kubectl config get-contexts
```

The `*` marks the current context. Exam clusters often define several contexts, so switch with `kubectl config use-context`.

6. Extract and read the client certificate:

```bash
grep "client-cert" ~/.kube/config | \
  sed 's/\(.*client-certificate-data: \)\(.*\)/\2/' | \
  base64 --decode \
  > cert.pem
openssl x509 -in cert.pem -text -noout
```

The line that matters:

```text
Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin
```

CN becomes the user, O becomes the group. The cluster verifies the certificate, so the request is authenticated as `kubernetes-admin`.

7. Back up the kubeconfig, strip the user section, and retry:

```bash
cp .kube/config .kube/config.orig
sed -i '15,$d' .kube/config
kubectl get pods --v=6
```

With no certificate, kubectl falls back to username and password prompts.

8. Type any username and password. The request fails:

```text
status="403 Forbidden"
Error from server (Forbidden): pods is forbidden: User "system:anonymous" cannot list resource "pods" in API group "" in the namespace "default"
```

No password authentication is configured, so any credentials are rejected. An expired or invalid client certificate gives the same result.

9. Press Ctrl+C and restore the kubeconfig:

```bash
cp .kube/config.orig .kube/config
```

10. Create a Pod and inspect its projected volume:

```bash
kubectl run nginx --image=nginx
kubectl describe pod nginx
```

Relevant parts:

```text
Service Account:  default
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-t9njc (ro)
Volumes:
  kube-api-access-t9njc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
```

11. Read the token:

```bash
kubectl exec nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token && echo
```

It prints a JWT, for example `eyJhbGciOiJSUzI1NiIsImtpZCI6...`. Its claims include the audience, expiry, issuer, and a `kubernetes.io` block naming the namespace, node, Pod, and ServiceAccount. The subject looks like `system:serviceaccount:default:default`.

12. Delete the Pod:

```bash
kubectl delete pod nginx
```

### Summary (Step 1)

Requests from kubectl, client libraries, or raw REST all pass the same authentication. Users authenticate with a client certificate from the kubeconfig. ServiceAccounts authenticate with a projected, Pod-bound token.

---

## Step 2 - Authorization (RBAC)

1. List Roles everywhere:

```bash
kubectl get roles --all-namespaces
```

Example output:

```text
NAMESPACE              NAME                                             CREATED AT
kube-public            kubeadm:bootstrap-signer-clusterinfo             2025-06-26T21:14:05Z
kube-system            extension-apiserver-authentication-reader        2025-06-26T21:14:04Z
kube-system            kube-proxy                                       2025-06-26T21:14:06Z
kube-system            system::leader-locking-kube-scheduler            2025-06-26T21:14:04Z
kubernetes-dashboard   kubernetes-dashboard                             2025-06-26T21:14:10Z
```

Every Role has a namespace. These built-in Roles give each component only what it needs.

2. Read one Role:

```bash
kubectl get -n kube-system role kube-proxy -o yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kube-proxy
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resourceNames:
  - kube-proxy
  resources:
  - configmaps
  verbs:
  - get
```

This allows `get` on the ConfigMap named `kube-proxy` only. `apiGroups: [""]` is the core group. ClusterRole rules use the same structure.

3. List ClusterRoles:

```bash
kubectl get clusterroles
```

Kubernetes ships several by default, including `admin`, `edit`, and `cluster-admin`.

4. Read `cluster-admin`:

```bash
kubectl get clusterrole cluster-admin -o yaml
```

```yaml
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

Everything on everything, plus all non-resource URLs. `kubectl describe clusterrole cluster-admin` shows the same as a table:

```text
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

5. See who that role is bound to:

```bash
kubectl get clusterrolebinding cluster-admin -o yaml
```

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

`roleRef` names the role, `subjects` lists who gets it. The certificate from Step 1 puts `kubernetes-admin` in `system:masters`, so every request is allowed.

6. Ask whether an action is allowed:

```bash
kubectl auth can-i list nodes
```

```text
Warning: resource 'nodes' is not namespace scoped

yes
```

More options: `kubectl auth can-i --help`.

7. Check another user by impersonation:

```bash
kubectl auth can-i list nodes --as=Tracy
```

Answer is `no`. No binding exists for Tracy. `--as` works because a cluster admin may impersonate.

### Summary (Step 2)

Roles and ClusterRoles list allowed actions. RoleBindings and ClusterRoleBindings attach them to subjects. `kubectl auth can-i`, with `--as`, answers the access question directly.

---

## Step 3 - Admission control

1. Show which plugins are enabled by default:

```bash
kubectl exec -n kube-system kube-apiserver-ip-10-0-0-100.us-west-2.compute.internal -- kube-apiserver -h | grep "enable-admission-plugins strings"
```

The help text lists the defaults:

```text
NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, PodSecurity, Priority,
DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize,
RuntimeClass, CertificateApproval, CertificateSigning, ClusterTrustBundleAttest, CertificateSubjectRestriction,
DefaultIngressClass, PodTopologyLabels, MutatingAdmissionPolicy, MutatingAdmissionWebhook,
ValidatingAdmissionPolicy, ValidatingAdmissionWebhook, ResourceQuota
```

It also prints the full list of plugins you may enable, such as `AlwaysPullImages`, `NodeRestriction`, `PodNodeSelector`, and `EventRateLimit`.

To see extra plugins passed with `--enable-admission-plugins`, look at the running API server process on a control plane node.

2. Connect to the control plane node:

```bash
ssh 10.0.0.100 -oStrictHostKeyChecking=no
```

3. Find the flag on the running process:

```bash
ps -ef | grep kube-apiserver | grep enable-admission-plugins
```

Among the flags:

```text
--enable-admission-plugins=NodeRestriction
--authorization-mode=Node,RBAC
```

So this cluster adds `NodeRestriction` on top of the defaults.

### Summary (Step 3)

Admission control is the last gate. Plugins can reject or modify a request. Defaults come from the API server build, extras come from the `--enable-admission-plugins` flag.

---

## Summary (all steps)

A request must pass authentication, then authorization, then every admission plugin. Authentication comes from the kubeconfig certificate for users and a projected token for ServiceAccounts. Authorization is RBAC, and `kubectl auth can-i --as=<user>` tells you the answer without guessing. Admission plugins add the final checks and defaults.
