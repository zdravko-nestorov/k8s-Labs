# 03 - Managing Pods with Deployments

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Kubernetes Pod Design for Application Developers: Definition Basics  
> Lab: Managing Pods with Deployments  
> Topics: deployments, replicas, rolling-updates, rollback, replicaset, services, loadbalancer, kubectl-scale, kubectl-set-image, kubectl-rollout

## Goal

Create a Deployment for a web server. Practice rolling updates, rollbacks, scaling, and exposing the Deployment with a LoadBalancer Service.

## Notes

- Prefer Deployments over standalone Pods for high availability and fault tolerance.
- A Deployment declares desired state. The controller reconciles the cluster to match it.
- Deployments support configurable rolling updates and revision history for rollbacks.
- `kubectl create deployment ... --dry-run=client -o yaml` generates a manifest without creating the resource.
- Deployments use ReplicaSets per revision. You usually work with the Deployment, not the ReplicaSets directly.
- Rolling update bounds: **max unavailable** (pods that may be down) and **max surge** (pods above desired count).
- `kubectl edit ... --record` saves the command in an annotation (CHANGE-CAUSE).
- Events on `kubectl describe` are a good first place to check when something looks wrong.
- `kubectl rollout undo` goes to the previous revision. Use `--to-revision` for an older one.
- LoadBalancer Services expose Pods outside the cluster. ClusterIP and NodePort stay inside the cluster.

## Steps

### Step 1 - Manage Pods with a Deployment

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace deployment
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=deployment
```

2. Generate a Deployment manifest (dry run, YAML only):

```bash
kubectl create deployment --image=httpd:2.4.38 web-server --dry-run=client -o yaml
```

Important parts of the Deployment `spec`:

- **replicas:** how many Pod copies to run
- **selector:** label selector so the controller tracks its Pods (`matchLabels` is like `app=web-server`). See `kubectl explain deployment.spec.selector`.
- **template:** Pod template (same idea as a Pod manifest). The Pod must have the label the selector uses (`app: web-server`).

Dry-run output is not minimal (`status`, `creationTimestamp`, and similar may appear). Those fields are harmless in a file.

3. Create the Deployment for real:

```bash
kubectl create deployment --image=httpd:2.4.38 web-server
```

You can also save YAML, edit it, and run `kubectl create -f`.

4. Describe the Deployment:

```bash
kubectl describe deployments web-server
```

Highlights:

- **StrategyType:** `RollingUpdate` (new desired state rolls out gradually)
- **RollingUpdateStrategy:** `max unavailable` and `max surge` (see `kubectl explain deployments.spec.strategy.rollingUpdate`)
- References to **ReplicaSet** (one per Deployment revision)

5. Confirm one Pod is running (matches desired replicas):

```bash
kubectl get pods
```

The Pod name starts with the Deployment name.

6. Scale to 6 replicas:

```bash
kubectl scale deployment web-server --replicas=6
```

7. Confirm six Pods:

```bash
kubectl get pods
```

Scaling up does not roll out a new Pod version. The original Pod can stay.

8. View rollout history (only revision 1 so far):

```bash
kubectl rollout history deployment web-server
```

9. Edit the Deployment (records the change cause):

```bash
kubectl edit deployment web-server --record
```

Default editor is vim. For exams, practice a CLI editor (`vimtutor` helps).

10. In the editor, open port 80 on the container:

- Move to the line with `image: httpd:2.4.38` (or search with `/image` then Enter)
- Press `o` for a new line below
- Type `ports:` at the same indent as `image:`
- Press Enter, then type `- containerPort: 80` at the next indent level

Result shape:

```yaml
    image: httpd:2.4.38
    ports:
    - containerPort: 80
```

11. Save and quit: Esc, then `:wq`, then Enter.

A new desired state starts a rolling update.

12. Confirm the rollout finished:

```bash
kubectl rollout status deployment web-server
```

13. View history. CHANGE-CAUSE should show the edit command:

```bash
kubectl rollout history deployment web-server
```

14. Change the container image:

```bash
kubectl set image deployment web-server httpd=httpd:2.4.38-alpine
```

15. Check latest events:

```bash
kubectl describe deployments web-server
```

A new ReplicaSet grows (respecting max surge). The old ReplicaSet shrinks (respecting max unavailable).

16. Roll back the previous change:

```bash
kubectl rollout undo deployment web-server
```

17. Expose the Deployment with a LoadBalancer Service on port 80:

```bash
kubectl expose deployment web-server --type=LoadBalancer --port=80
```

18. Watch until EXTERNAL-IP shows a DNS address:

```bash
watch kubectl get services
```

19. Copy the EXTERNAL-IP DNS address. Press Ctrl+C to stop watching.

20. Open that DNS address in a browser to confirm the Service exposes the Deployment.

Note: If the page does not load at first, wait 1-2 minutes for load balancer health checks. Refresh every minute until it loads.

## Validation

- **Service Exposing Deployment Via Load Balancer:** check that the Deployment is reachable through a LoadBalancer Service.

## Summary

Deployments manage Pods with rolling updates, rollbacks, and continuous reconciliation of desired vs actual state.
