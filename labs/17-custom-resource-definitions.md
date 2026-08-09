# 17 - Using Custom Resource Definitions (CRDs)

> Course: Cloud Native Champions: CKAD Bootcamp  
> Lab: Using Kubernetes Custom Resource Definitions (CRDs)  
> Topics: crd, custom-resources, api-resources, argocd, gitops, kubectl-get, kubectl-describe, kubectl-edit, port-forward, secrets, shortnames

## Goal

Discover CRDs in a cluster and work with a custom resource using normal kubectl commands. The example is an Argo CD `Application` that deploys a guestbook app, updated to a new version by editing the resource.

## Notes

### What a CRD is

A **CustomResourceDefinition** extends the Kubernetes API with your own resource types. Once defined, custom resources behave like built-in ones: `kubectl get`, `describe`, `apply`, and `edit` all work.

A CRD alone stores data. A **custom controller** must watch the resource and act on it. Argo CD ships both.

### Finding custom resources

- `kubectl get crds` lists the definitions
- `kubectl api-resources` lists every resource type, built-in and custom, with short names and whether it is namespaced

### A CRD is a normal object

It has `apiVersion`, `kind`, `metadata`, and `spec`. The `spec` holds:

- `group`: for example `argoproj.io`
- `names`: `kind`, `plural`, `singular`, `shortNames`
- `scope`: `Namespaced` or `Cluster`
- `versions[].schema.openAPIV3Schema`: the field schema you build objects against
- `versions[].additionalPrinterColumns`: the extra columns `kubectl get` prints

### Argo CD terms used here

- **Application:** one resource holding everything needed to deploy an app. It creates the Namespace, Deployment, and Service for you.
- **AppProject:** a logical group of Applications.
- `spec.source.targetRevision` sets which Git revision is deployed. Change it to release a new version.

Argo CD is only one example. Artifact Hub lists many more.

## Steps

### Step 1 - Discover and use custom resources

1. List the CRDs in the cluster:

```bash
kubectl get crds
```

Example output:

```text
NAME                                                  CREATED AT
applications.argoproj.io                              2026-08-09T10:37:04Z
appprojects.argoproj.io                               2026-08-09T10:37:04Z
bgpconfigurations.crd.projectcalico.org               2025-06-26T21:14:07Z
bgppeers.crd.projectcalico.org                        2025-06-26T21:14:07Z
globalnetworkpolicies.crd.projectcalico.org           2025-06-26T21:14:07Z
ippools.crd.projectcalico.org                         2025-06-26T21:14:07Z
networkpolicies.crd.projectcalico.org                 2025-06-26T21:14:08Z
networksets.crd.projectcalico.org                     2025-06-26T21:14:08Z
```

Two sources here: Argo CD (`argoproj.io`) and Calico (`projectcalico.org`), the cluster network.

Custom resources also appear in:

```bash
kubectl api-resources
```

The Argo rows show the names and short names you can type:

```text
NAME           SHORTNAMES         APIVERSION             NAMESPACED   KIND
applications   app,apps           argoproj.io/v1alpha1   true         Application
appprojects    appproj,appprojs   argoproj.io/v1alpha1   true         AppProject
```

Filter to one group with grep:

```bash
kubectl api-resources | grep crd.
```

2. View the Application CRD:

```bash
kubectl get crds applications.argoproj.io -o yaml | more
```

Most fields do not matter here. The important part is the shape:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.argoproj.io
spec:
  group: argoproj.io
  names:
    kind: Application
    listKind: ApplicationList
    plural: applications
    shortNames:
    - app
    - apps
    singular: application
  scope: Namespaced
  versions:
  - additionalPrinterColumns:
    - jsonPath: .status.sync.status
      name: Sync Status
      type: string
    - jsonPath: .status.health.status
      name: Health Status
      type: string
    name: v1alpha1
    schema:
      openAPIV3Schema:
        description: Application is a definition of Application resource.
```

`names` gives the forms you can use with kubectl. The schema tells you how to build an Application object.

3. Press `q` to leave the pager.

4. List Applications in every namespace:

```bash
kubectl get applications --all-namespaces
```

Example output:

```text
NAMESPACE   NAME        SYNC STATUS   HEALTH STATUS
default     guestbook   Synced        Progressing
```

Note: if the Application is missing or not yet Healthy, repeat the command every minute. It can take up to 10 minutes after lab start.

5. Describe it:

```bash
kubectl describe application guestbook
```

Useful parts of the output:

```text
Spec:
  Destination:
    Namespace:  guestbook
    Server:     https://kubernetes.default.svc
  Project:      default
  Source:
    Path:             guestbook
    Repo URL:         https://github.com/cloudacademy/argocd-example-apps
    Target Revision:  guestbook-v0.1
  Sync Policy:
    Automated:
    Sync Options:
      CreateNamespace=true
```

The Sync Result and Resources sections show the built-in objects this one custom resource manages:

```text
Kind:        Namespace   Message: namespace/guestbook created
Kind:        Service     Message: service/guestbook-ui created
Kind:        Deployment  Message: deployment.apps/guestbook-ui created
```

Note: an error can show for up to 10 minutes before the sync turns healthy.

6. Port-forward the Argo server in the background:

```bash
nohup kubectl port-forward service/my-argo-cd-argocd-server -n default --address 0.0.0.0 8001:443 &>/dev/null &
```

The shell prints the background process id. `--address 0.0.0.0` listens on all interfaces. `&>/dev/null` hides the output.

7. Print the web address:

```bash
# Get the bastion's public IP
bastion_ip=$(curl --silent http://169.254.169.254/latest/meta-data/public-ipv4)
# Output the Argo web address
echo "http://$bastion_ip:8001"
```

Output looks like `http://<bastion-public-ip>:8001`.

8. Open the link in a new tab.

9. The certificate warning is expected, because the server uses a self-signed certificate. Accept the risk and continue. In Chrome: Advanced, then Proceed.

10. Get the initial admin password from a Secret:

```bash
kubectl -n default get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

The command prints the password for your lab session.

11. Paste the password into the Password field.

12. Enter `admin` as Username and click SIGN IN. The `guestbook` Application appears as a tile.

13. Click the tile to open the Application detail view. It shows the Service, the Deployment, and their child resources. One custom resource created all of them.

14. Click the `guestbook-ui` Service to see its details.

15. Copy the Service hostname. It is the load balancer serving the guestbook site, for example:

```text
ab243ed45ce734a0d940709bf3008aa6-1876650473.us-west-2.elb.amazonaws.com
```

You can get the same value with `kubectl get service`.

16. Open that address in a new tab to see the guestbook app.

17. Edit the Application with kubectl:

```bash
kubectl edit applications guestbook
```

It opens in Vim, exactly like a built-in resource.

18. Move the cursor to the end of the `spec.source.targetRevision` line.

19. Press `a` to edit, then Backspace and type `2`, so the value becomes `guestbook-v0.2`.

20. Press Escape, then type `:wq` and Enter to save.

The Argo controllers see the new desired state and reconcile the cluster to match.

21. In the Argo CD web interface, close the `guestbook-ui` detail view. The sync status now shows `guestbook-v0.2`.

22. Refresh the guestbook app tab to see the new version.

## Validation

- **Application Version Updated:** the Application version changed

## Summary

Kubernetes can be extended with custom resources, and everything you know about kubectl still applies to them. Custom resources need custom controllers before anything happens. Argo CD is one example: a single `Application` resource creates and keeps in sync a Namespace, Deployment, and Service.
