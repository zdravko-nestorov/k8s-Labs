# 15 - Creating Ingress Rules

> Course: Cloud Native Champions: CKAD Bootcamp  
> Lab: Creating Ingress Rules  
> Topics: ingress, ingress-controller, nginx-ingress, path-routing, host-routing, virtual-hosting, ingressClassName, rewrite-target, annotations, services, deployments, loadbalancer, kubectl-logs, curl

## Goal

Deploy two apps, then route to them with one Ingress resource. First by URL path (`/blue`, `/red`), then by hostname (`blue.example.com`, `red.example.com`).

## Notes

### What an Ingress gives you

An Ingress manages external access to Services in the cluster, usually HTTP and HTTPS routes. It can provide:

- Load balancing
- SSL termination
- HTTP virtual hosting (route by domain name)
- HTTP URL path-based routing

### Ingress needs a controller

An Ingress resource does nothing on its own. An **Ingress Controller** must run in the cluster. Popular ones:

- NGINX
- Traefik
- Haproxy
- Envoy

Some service meshes, such as Istio, also provide an Ingress controller. This lab uses the NGINX Ingress Controller.

### How the controller gets traffic

- On AWS, the NGINX Ingress Controller creates a Network Load Balancer.
- It also supports Google Cloud and Azure.
- On bare metal, it creates a Service with a NodePort to receive external traffic.

### Default backend

With no matching rule, the controller sends the request to the **default backend**. With NGINX, that is the 404 Not Found page.

### Useful facts

- `nginx.ingress.kubernetes.io/rewrite-target: /` rewrites the request to `/` before it reaches the Service.
- A cluster can hold many Ingress resources. Group related endpoints in one resource to keep things readable.
- Routing by hostname is also called **virtual hosting** or **name based routing**.
- Controller logs help when routing misbehaves.
- In real life you point DNS names at the cluster. In the lab you fake it with a `Host` header in `curl`.

## Steps

### Step 1 - Create and test Ingress rules

1. Verify the NGINX Ingress Controller is running:

```bash
watch kubectl get all --namespace ingress-nginx
```

Example output:

```text
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-98dv6        0/1     Completed   1          2m48s
pod/ingress-nginx-admission-patch-92zp5         0/1     Completed   2          2m48s
pod/ingress-nginx-controller-7557c66f69-t7rqn   1/1     Running     0          2m49s

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.109.100.251   aceb491f6c7824ce8bf26f9947bb49d6-9af7901b41406d66.elb.us-west-2.amazonaws.com   80:32120/TCP,443:30799/TCP   2m50s
service/ingress-nginx-controller-admission   ClusterIP      10.108.114.139   <none>                                                                          443/TCP                      2m50s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           2m49s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-7557c66f69   1         1         1       2m49s

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           94s        2m48s
job.batch/ingress-nginx-admission-patch    Complete   1/1           102s       2m48s
```

Note: wait until `pod/ingress-nginx-controller-` shows Status `Running` before you continue. It can take up to a minute after lab setup finishes.

2. Look at EXTERNAL-IP on `service/ingress-nginx-controller`. That is the DNS name of the AWS Network Load Balancer created with the controller.

3. Copy that value and open it in a new browser tab. You get a 404 Not Found page.

No Ingress resources exist yet, so every request goes to the default backend. With NGINX that is the 404 page.

Note: If you do not see the 404 page, the controller is still starting. Wait a minute or two and retry. Setup is usually done once the controller Pod is about 5m40s old.

Keep this browser tab open. You need it later.

4. Back in the terminal, press Ctrl+C to exit `watch`.

5. Create the first Deployment and Service:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-app
spec:
  selector:
    matchLabels:
      app: blue-app
  replicas: 2
  template:
    metadata:
      labels:
        app: blue-app
    spec:
      containers:
      - name: blue-app
        image: public.ecr.aws/cloudacademy-labs/cloudacademy/labs/k8s-ingress-app:f9a36c8
        env:
        - name: COLOR
          value: '#A7C7E7'
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: blue-app
spec:
  selector:
    app: blue-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
EOF
```

This uses the Bash heredoc to pipe a multiline manifest into `kubectl`.

The app shows a blank page whose background is the `COLOR` environment variable. The container listens on 8000. The Service maps port 80 to 8000. Replicas is 2.

6. Create the second Deployment and Service:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red-app
spec:
  selector:
    matchLabels:
      app: red-app
  replicas: 2
  template:
    metadata:
      labels:
        app: red-app
    spec:
      containers:
      - name: red-app
        image: public.ecr.aws/cloudacademy-labs/cloudacademy/labs/k8s-ingress-app:f9a36c8
        env:
        - name: COLOR
          value: '#FAA0A0'
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: red-app
spec:
  selector:
    app: red-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
EOF
```

Only two things differ from the first: the name `red-app` instead of `blue-app`, and the `COLOR` value.

7. Verify Pods and Services:

```bash
kubectl get pod,service
```

Expect four running Pods and three Services.

8. Create the Ingress with path-based rules:

```bash
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-app
            port:
              number: 80
      - path: /red
        pathType: Prefix
        backend:
          service:
            name: red-app
            port:
              number: 80
EOF
```

One HTTP rule with two paths. `/blue` goes to the blue Service, `/red` to the red one. The rewrite annotation sends the request to `/` before it reaches the Service.

Full annotation list: see the Annotations page of the NGINX Ingress Controller documentation.

9. In the browser tab with the Ingress DNS name, add `/blue` to the URL. You get a blue page.

10. Change `/blue` to `/red`. You get a red page.

Path-based routing is useful for microservices. You control which Service answers which HTTP path.

11. View the Ingress controller logs:

```bash
kubectl logs -l app.kubernetes.io/name=ingress-nginx --namespace ingress-nginx
```

You see GET requests for both `/blue` and `/red`. Example lines:

```text
I0809 10:02:59.780896       7 store.go:365] "Found valid IngressClass" ingress="default/lab-ingress" ingressclass="nginx"
I0809 10:02:59.781460       7 controller.go:152] "Configuration changes detected, backend reload required"
I0809 10:02:59.847081       7 controller.go:169] "Backend successfully reloaded"
<your-public-ip> - - [09/Aug/2026:10:04:52 +0000] "GET /blue HTTP/1.1" 200 99 "-" "Mozilla/5.0 ..." 511 0.004 [default-blue-app-80] [] 192.168.23.130:8000 99 0.004 200 50cbb22b9f922dd21cef678188e16bb4
<your-public-ip> - - [09/Aug/2026:10:05:18 +0000] "GET /red HTTP/1.1" 200 99 "-" "Mozilla/5.0 ..." 510 0.003 [default-red-app-80] [] 192.168.23.131:8000 99 0.003 200 0a3e51595f8d7ea5cf9b5035cfa897aa
```

12. Switch the Ingress to hostname-based routing:

```bash
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: blue.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: blue-app
            port:
              number: 80
  - host: red.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: red-app
            port:
              number: 80
EOF
```

Two rules, one per host.

13. Test hostname routing with `curl` and a `Host` header:

```bash
load_balancer_dns=$(kubectl -n ingress-nginx get service ingress-nginx-controller -o jsonpath={.status.loadBalancer.ingress[0].hostname})
curl --header "Host: blue.example.com" http://$load_balancer_dns
curl --header "Host: red.example.com" http://$load_balancer_dns
```

Expected output, with different background colors:

```text
<html><head><title>Application</title></head><body style="background-color:#A7C7E7"></body></html>
<html><head><title>Application</title></head><body style="background-color:#FAA0A0"></body></html>
```

Setting the `Host` header shows how one endpoint answers different hostnames. Outside a lab you would create real DNS names pointing at the cluster.

## Summary

You deployed two apps, created one Ingress that routes HTTP paths to different Services, switched the same Ingress to route by hostname, and confirmed both work.
