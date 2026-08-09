# 16 - Deployment Strategies: Canary and Blue/Green

> Course: Cloud Native Champions: CKAD Bootcamp  
> Lab: Using Kubernetes Primitives to Implement Common Deployment Strategies  
> Topics: deployment-strategies, canary, blue-green, rollingupdate, recreate, labels, selectors, services, loadbalancer, kubectl-patch, kubectl-set-image, readiness-probe

## Goal

Combine Deployments, Services, and labels to run Blue/Green and Canary deployments, beyond the built-in `RollingUpdate` and `Recreate` strategies.

## Notes

### The two strategies

- **Canary:** a small share of traffic goes to the new version first. You build confidence before a full rollout. Low blast radius if something breaks.
- **Blue/Green:** the old version is "blue", the new one is "green". You build green fully while blue still serves all traffic, then cut over at once. No mixing of old and new. It costs twice the resources of one environment.

Both avoid downtime, as does a rolling update.

### The trick that makes both work

The Service selector decides which Pods receive traffic.

| Selector | Effect |
|----------|--------|
| `app: web` | Every version behind the Service (used for canary) |
| `app: web, version: old` | Blue only |
| `app: web, version: new` | Green only |

Keep a shared label (`app: web`) for the application, and a second label (`version:`) for the release. In production the `version` value is usually a release number. Some teams use a `track` label instead, with values such as `stable` and `canary`.

### Built-in Deployment strategies

- `RollingUpdate` (default): replaces Pods gradually
- `Recreate`: deletes all Pods before creating new ones

### Canary traffic share

Traffic splits by Pod count. With 10 stable Pods and 1 canary Pod, roughly 1 request in 11 hits the canary.

### Browser caching

The browser can cache the old page. Use a private window, another browser, or `curl` to see the change.

## Steps

### Step 1 - Blue/Green and Canary with Kubernetes primitives

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace strategies
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=strategies
```

Expected output:

```text
namespace/strategies created
Context "kubernetes-admin@kubernetes" modified.
```

2. Create a Deployment with 10 replicas (this is the blue environment):

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    version: old
  name: app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web
      version: old
  strategy:
    type: RollingUpdate # Default value is RollingUpdate, Recreate also supported
  template:
    metadata:
      labels:
        app: web
        version: old
    spec:
      containers:
      - image: nginx:1.21.3-alpine
        name: nginx
        ports:
        - containerPort: 80
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          periodSeconds: 5
          successThreshold: 2
          timeoutSeconds: 10
EOF
```

The `app: web` label identifies the application. The `version: old` label marks this release.

3. Create a LoadBalancer Service in front of it:

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: app
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web
  type: LoadBalancer
EOF
```

Think of this Service as the entry point for production traffic. Putting it in front lets you add more Deployments behind it later.

4. Wait for EXTERNAL-IP, then press Ctrl+C:

```bash
kubectl get service -w
```

Example output:

```text
NAME   TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
app    LoadBalancer   10.111.181.253   aa71d9f1af2754be7a0e796b100f5e09-727351223.us-west-2.elb.amazonaws.com   80:31888/TCP   22s
```

5. Open the EXTERNAL-IP address in a new browser tab. You see the nginx welcome page.

Note: health checks can take a minute. Refresh every 15 seconds until the page appears.

Keep this tab open for the whole lab.

6. Pin the Service to blue only, by adding `version: old` to the selector:

```bash
cat << EOF | kubectl patch service app --patch-file /dev/stdin
spec:
  selector:
    app: web
    version: old
EOF
```

The heredoc feeds the patch to `kubectl patch` through standard input (`/dev/stdin`). `kubectl edit` also works.

Confirm with:

```bash
kubectl get service -o wide
```

The SELECTOR column shows `app=web,version=old`.

7. Create the green environment: 10 replicas, httpd image, `version: new`:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    version: new
  name: app-new
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web
      version: new
  template:
    metadata:
      labels:
        app: web
        version: new
    spec:
      containers:
      - image: httpd:2.4.49-alpine3.14
        name: httpd
        ports:
        - containerPort: 80
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          periodSeconds: 5
          successThreshold: 2
          timeoutSeconds: 10
EOF
```

Same `app: web` label, new `version: new` label, different image.

8. Wait until `app-new` shows 10/10, then press Ctrl+C:

```bash
kubectl get deployments -w
```

Example output:

```text
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
app       10/10   10           10          6m33s
app-new   10/10   10           10          50s
```

With Blue/Green you can test green before cutting over. A common approach is a second Service used only for internal testing, deleted once you trust the new version.

9. Refresh the browser tab. It still shows nginx, so blue is still serving.

10. Cut over to green:

```bash
cat << EOF | kubectl patch service app --patch-file /dev/stdin
spec:
  selector:
    app: web
    version: new
EOF
```

11. Refresh the browser. You see `It works!` from httpd.

Note: if the browser caches the old page, use incognito or another browser.

Blue is still running, so you could switch back instantly if green misbehaved. This lab assumes green is healthy.

12. Delete the blue environment:

```bash
kubectl delete deployments app
```

13. Prepare for canary by removing the version from the Service selector:

```bash
kubectl patch service app --type=json --patch='[{"op": "remove", "path": "/spec/selector/version"}]'
```

A JSON patch is needed to remove a key. `kubectl edit` also works.

The Service now selects Pods of any version.

14. Create a single-replica canary with the caddy image:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    version: canary
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: canary
  template:
    metadata:
      labels:
        app: web
        version: canary
    spec:
      containers:
      - image: caddy:2.4.5-alpine
        name: caddy
        ports:
        - containerPort: 80
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          periodSeconds: 5
          successThreshold: 2
          timeoutSeconds: 10
EOF
```

15. Wait until `app-canary` shows 1/1, then press Ctrl+C:

```bash
kubectl get deployments -w
```

Example output:

```text
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
app-canary   1/1     1            1           22s
app-new      10/10   10           10          4m
```

The Service picks up the canary Pod automatically. About 1 request in 11 goes to it.

16. Send one request per second until you see the Caddy response, then press Ctrl+C:

```bash
service_address=$(kubectl get service app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
while true; do
    curl --silent $service_address | grep "works"
    sleep 1
done
```

Example output:

```text
		<title>Caddy works!</title>
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>
```

17. Delete the canary and roll the new image into the main Deployment:

```bash
kubectl delete deployment app-canary
kubectl set image deployment app-new *=caddy:2.4.5-alpine
```

`*=` sets the image on every container in the Pod template. The rollout is a normal rolling update. When it finishes, all Pods serve Caddy. Refresh the browser or open the Service address to confirm.

## Summary

Services, Deployments, and labels are enough to build Blue/Green and Canary deployments. The Service selector is the switch: narrow it to one version for Blue/Green, widen it to the shared app label for Canary.
