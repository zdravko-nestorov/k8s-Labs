# 14 - Deployment using ConfigMaps and Helm

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Performing a Kubernetes Deployment using ConfigMaps and Helm  
> Lab: AWS login / IDE / Image build and namespace / NGINX ConfigMap / Deployment / Helm project / Helm templates  
> Topics: helm, helm-create, helm-template, helm-package, helm-install, values-file, configmap, deployment, multi-container, nginx, flask, docker-build, namespace, service, kubectl-expose, kubectl-describe, kubectl-logs, kubectl-exec, errimagepull, troubleshooting

## Goal

Build a Flask app image, deploy it behind an NGINX proxy in one Pod, feed NGINX its config from a ConfigMap, then repeat the same deployment with Helm templates and per-environment values files.

## Notes

### The application shape

One Pod, two containers:

- **nginx** listens on port 80 and proxies to `http://localhost:5000/`
- **flask** serves the app on port 5000 and reads the `APP_NAME` environment variable

Both containers share the Pod network, so NGINX can reach Flask on `localhost`.

### ConfigMap use here

The NGINX config lives in a ConfigMap, not in the image. The Deployment mounts one key of the ConfigMap over a single file with `subPath`:

```yaml
volumeMounts:
- name: nginx-proxy-config
  mountPath: /etc/nginx/conf.d/default.conf
  subPath: nginx.conf
```

### Helm terms

- **Chart:** a package of templates plus default values. `helm create <name>` scaffolds one.
- **Templates:** `templates/*.yaml` with placeholders such as `{{ .Values.replicaCount }}`.
- **Values file:** `values.yaml` holds the defaults. Extra files such as `values.dev.yaml` override them per environment.
- **`helm template`:** renders the manifest and prints it. Nothing is deployed. Good for checking output first.
- **`helm template ... | kubectl apply -f -`:** renders and applies. Kubernetes tracks the resources, Helm does not track a release.
- **`helm install`:** installs a chart and records a release you can list with `helm ls`.
- **`helm package`:** builds a `.tgz` chart archive, for example `app-0.1.0.tgz`.

Helm 3 is used in this lab.

### Troubleshooting an image pull failure

The lab plants a wrong image tag on purpose. The path to the fix:

1. `kubectl get deploy` shows `0/1` READY
2. `kubectl get pods` shows `1/2` containers and `ErrImagePull`
3. `kubectl describe pod $POD_NAME` shows the failed pull in Events
4. `docker images` shows the real tag
5. Fix the manifest and `kubectl apply` again

### NGINX logs inside the container

`access.log` and `error.log` are symlinks to `/dev/stdout` and `/dev/stderr`, so `kubectl logs` shows them.

---

## Step 1 - Logging in to the AWS Console

The lab uses AWS, so you manage resources from the AWS Management Console.

1. Open the lab's cloud environment from the lab UI.

2. Sign in with the IAM user created for your lab session:

- Account ID or alias: keep the pre-populated value
- IAM user name: `student`
- Password: shown in the lab UI for that session

3. Select the **US West (Oregon) us-west-2** region in the upper-right dropdown. This lab requires us-west-2.

Use an up-to-date Chrome or Firefox for the console.

---

## Step 2 - Connecting to the web-based IDE on port 8080

The Cloud Academy IDE listens on port 8080 over HTTP.

1. Open the IDE in your browser using the public IP of the `ide.containers.cloudacademy.platform.instance` EC2 instance:

```text
http://waiting_for_IP:8080
```

Notes:

- `waiting_for_IP` changes to the real IDE IP once the IDE is ready.
- Startup takes about 5 to 7 minutes.
- Port 8080 is used because ports 80 and 443 stay free for other parts of the lab.
- If outbound port 8080 is blocked and you are an Enterprise user, use the Enterprise Bridge.

---

## Step 3 - Build the image and create the namespace

Build a custom Docker image with a simple Flask web app, then create and switch to a new namespace.

### The source files

`lab/code/App/lab-code/flaskapp/Dockerfile`:

```dockerfile
FROM python:3

RUN pip install --no-cache-dir --progress-bar off flask

RUN mkdir -p /corp/app
WORKDIR /corp/app
COPY main.py .
ENV FLASK_APP=/corp/app/main.py

ENV APP_NAME=CloudAcademy.DevOps.K8s

CMD ["flask", "run", "--host=0.0.0.0"]
```

`lab/code/App/lab-code/flaskapp/main.py`:

```python
import os
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    appname = os.environ['APP_NAME']
    response = "%s - %s\n" %('Hello World', appname)
    return response
```

### Instructions

1. In the Explorer view, expand `lab/code/App/lab-code/flaskapp` and open both files.

If the Explorer has not loaded the project, add `/#/cloudacademy` to the browser URL and refresh.

2. Right-click the `flaskapp` folder and choose **Open in Terminal**.

3. List the folder contents:

```bash
ls -la
```

4. Build the image:

```bash
docker build -t cloudacademydevops/flaskapp .
```

5. Confirm the image tag `cloudacademydevops/flaskapp:latest` exists:

```bash
docker images
```

6. Create the namespace:

```bash
kubectl create ns cloudacademy
```

7. Switch to it:

```bash
kubectl config set-context --current --namespace=cloudacademy
```

### Validation (Step 3)

- **Check the Docker Image has Been Built Successfully:** image `cloudacademydevops/flaskapp:latest` exists
- **Check Namespace Has Been Created:** namespace `cloudacademy` exists

---

## Step 4 - Apply the NGINX ConfigMap

1. Change into the `k8s` directory and list it:

```bash
cd ../k8s && ls -la
```

2. Open `lab/code/App/lab-code/k8s/nginx.configmap.yaml`. It starts as a placeholder:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: cloudacademy
data:
  nginx.conf: |-
    #CODE1.0:
    #add the nginx.conf configuration - this will be referenced within the deployment.yaml
```

3. Add the server block under the `#CODE1.0` comment. The finished file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: cloudacademy
data:
  nginx.conf: |-
    #CODE1.0:
    #add the nginx.conf configuration - this will be referenced within the deployment.yaml
    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://localhost:5000/;
            proxy_set_header Host "localhost";
        }
    }
```

Notes:

- Save the file with CTRL+S before you continue. This matters.
- Keep the indentation.
- You can copy the finished version from `lab/code/App/solution-code/k8s/nginx.configmap.yaml`.

4. Apply it:

```bash
kubectl apply -f nginx.configmap.yaml
```

5. List ConfigMaps:

```bash
kubectl get configmaps
```

### Validation (Step 4)

- **Check ConfigMap Has Been Created:** ConfigMap `nginx-conf` exists

---

## Step 5 - Apply the Deployment

1. Open `lab/code/App/lab-code/k8s/deployment.yaml`. It ships with three placeholder blocks (`#CODE2.0`, `#CODE2.1`, `#CODE2.2`).

2. Add the NGINX container under `#CODE2.0`:

```yaml
      - name: nginx
        image: nginx:1.13.7
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
        volumeMounts:
        - name: nginx-proxy-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
```

It mounts the config from the ConfigMap created in Step 4.

3. Add the Flask container under `#CODE2.1`:

```yaml
      - name: flask
        image: cloudacademydevops/flask
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 5000
        env:
        - name: APP_NAME
          value: CloudAcademy.DevOps.K8s.Manifest
```

The image tag `cloudacademydevops/flask` is **wrong on purpose**. You fix it later in this step.

`APP_NAME` holds the string the Flask app returns in the HTTP response.

4. Add the volume under `#CODE2.2`:

```yaml
      - name: nginx-proxy-config
        configMap:
          name: nginx-conf
```

The finished file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: cloudacademy
  labels:
    role: frontend
    env: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.13.7
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
        volumeMounts:
        - name: nginx-proxy-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf

      - name: flask
        image: cloudacademydevops/flask
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 5000
        env:
        - name: APP_NAME
          value: CloudAcademy.DevOps.K8s.Manifest

      volumes:
      - name: nginx-proxy-config
        configMap:
          name: nginx-conf
```

Save with CTRL+S before you continue. The finished version is also in `lab/code/App/solution-code/k8s/deployment.yaml`.

5. Apply it:

```bash
kubectl apply -f deployment.yaml
```

6. List Deployments:

```bash
kubectl get deploy
```

`frontend` shows `0/1` replicas READY.

7. List Pods:

```bash
kubectl get pods
```

The Pod shows `1/2` containers READY and an `ErrImagePull` issue.

8. Store the Pod name:

```bash
POD_NAME=`kubectl get pods -o jsonpath='{.items[0].metadata.name}'`
echo $POD_NAME
```

9. Describe the Pod:

```bash
kubectl describe pod $POD_NAME
```

The Events section shows the failed pull of the non-existent `cloudacademydevops/flask` image.

10. Confirm the real tag:

```bash
docker images | grep cloudacademydevops
```

The correct tag is `cloudacademydevops/flaskapp`.

11. Fix the manifest in place:

```bash
sed -i 's/cloudacademydevops\/flask.*/cloudacademydevops\/flaskapp/g' deployment.yaml
```

12. In the editor, confirm the Flask container now uses `cloudacademydevops/flaskapp` (line 35).

13. Apply the fix:

```bash
kubectl apply -f deployment.yaml
```

14. Check the Deployment again:

```bash
kubectl get deploy
```

`frontend` now shows `1/1` replicas READY.

15. Expose the Deployment as a Service:

```bash
kubectl expose deployment frontend --port=80 --target-port=80
```

16. Show the Service:

```bash
kubectl get svc frontend
```

17. Store the ClusterIP:

```bash
FRONTEND_SERVICE_IP=`kubectl get service/frontend -o jsonpath='{.spec.clusterIP}'`
echo $FRONTEND_SERVICE_IP
```

18. Test it:

```bash
curl -i http://$FRONTEND_SERVICE_IP
```

The body contains `Hello World - CloudAcademy.DevOps.K8s.Manifest`. That string comes from the `APP_NAME` environment variable in the manifest.

19. Change `APP_NAME` in `deployment.yaml` to `CloudAcademy.DevOps.K8s.Manifest.v2.00` and save with CTRL+S.

20. Apply the change:

```bash
kubectl apply -f deployment.yaml
```

21. Test again:

```bash
curl -i http://$FRONTEND_SERVICE_IP
```

The body now contains `Hello World - CloudAcademy.DevOps.K8s.Manifest.v2.00`.

22. List Pods:

```bash
kubectl get pods
```

Repeat until only the single RUNNING Pod remains before you continue.

23. Store the Pod name:

```bash
FRONTEND_POD_NAME=`kubectl get pods --no-headers -o custom-columns=":metadata.name"`
echo $FRONTEND_POD_NAME
```

24. List the NGINX log directory inside the nginx container:

```bash
kubectl exec -it $FRONTEND_POD_NAME -c nginx -- ls -la /var/log/nginx/
```

Expected output:

```text
drwxr-xr-x 1 root root 41 Dec 12  2017 .
drwxr-xr-x 1 root root 19 Dec 12  2017 ..
lrwxrwxrwx 1 root root 11 Dec 12  2017 access.log -> /dev/stdout
lrwxrwxrwx 1 root root 11 Dec 12  2017 error.log -> /dev/stderr
```

Both logs point at stdout and stderr, so `kubectl logs` can read them.

25. Read the NGINX logs:

```bash
kubectl logs $FRONTEND_POD_NAME nginx
```

26. Read the Flask logs:

```bash
kubectl logs $FRONTEND_POD_NAME flask
```

### Validation (Step 5)

- **Check Deployment Exists:** `frontend` Deployment with 1 running replica
- **Check the Correct Web Response is Returned:** body contains `Hello World - CloudAcademy.DevOps.K8s.Manifest`

---

## Step 6 - Create a Helm project

Scaffold an empty chart and render it. The `test-app` chart is **not** deployed. It only shows how charts are built.

1. Move up one directory to `lab/code/App/lab-code/`:

```bash
cd ..
```

2. Scaffold a chart:

```bash
helm create test-app
```

3. View the structure:

```bash
tree test-app/
```

Open the generated template files in the editor to see how a default chart is laid out.

4. Render the templates into one manifest:

```bash
helm template test-app/
```

Example output:

```yaml
---
# Source: test-app/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: RELEASE-NAME-test-app
  labels:
    helm.sh/chart: test-app-0.1.0
    app.kubernetes.io/name: test-app
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: test-app/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: RELEASE-NAME-test-app
  labels:
    helm.sh/chart: test-app-0.1.0
    app.kubernetes.io/name: test-app
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: test-app
    app.kubernetes.io/instance: RELEASE-NAME
---
# Source: test-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: RELEASE-NAME-test-app
  labels:
    helm.sh/chart: test-app-0.1.0
    app.kubernetes.io/name: test-app
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: test-app
      app.kubernetes.io/instance: RELEASE-NAME
  template:
    metadata:
      labels:
        app.kubernetes.io/name: test-app
        app.kubernetes.io/instance: RELEASE-NAME
    spec:
      serviceAccountName: RELEASE-NAME-test-app
      securityContext:
        {}
      containers:
        - name: test-app
          securityContext:
            {}
          image: "nginx:1.16.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
---
# Source: test-app/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "RELEASE-NAME-test-app-test-connection"
  labels:
    helm.sh/chart: test-app-0.1.0
    app.kubernetes.io/name: test-app
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['RELEASE-NAME-test-app:80']
  restartPolicy: Never
```

Compare the output with `test-app/values.yaml` to see which defaults filled which placeholders.

### Validation (Step 6)

- **Check Helm Template Creates a Manifest:** `helm template` generated a manifest

---

## Step 7 - Deploy with Helm templates and values files

Deploy the provided `app` chart, then reuse it for dev and prod with different values files.

1. Remove the resources from the earlier steps:

```bash
kubectl delete deploy,pods,svc --all
```

2. Open `lab/code/App/lab-code/app/values.yaml`. It is a standard chart values file with a `#CODE3.0` placeholder at the end:

```yaml
# Default values for app.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  tag: stable
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths: []

  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}

nodeSelector: {}

tolerations: []

affinity: {}

#CODE3.0:
#create new configuration value which will store a message to be passed into the Flask web app as an environment variable
```

3. Add the value under `#CODE3.0`:

```yaml
webapp:
  message: CloudAcademy.DevOps.K8s.Helm
```

Save with CTRL+S. Keep the indentation. The finished version is in `lab/code/App/solution-code/app/values.yaml`.

4. Open `lab/code/App/lab-code/app/templates/deployment.yaml`. The Flask container has an empty `env:` with a `#CODE3.1` placeholder.

5. Add the environment variable under `#CODE3.1`:

```yaml
            - name: APP_NAME
              value: {{ .Values.webapp.message }}
```

The finished template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
{{ include "app.labels" . | indent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "app.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "app.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
    {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
    {{- end }}
      serviceAccountName: {{ template "app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: nginx
          image: nginx:1.13.7
          imagePullPolicy: IfNotPresent
          ports:
          - name: http
            containerPort: 80
          volumeMounts:
          - name: nginx-proxy-config
            mountPath: /etc/nginx/conf.d/default.conf
            subPath: nginx.conf
        - name: flask
          image: cloudacademydevops/flaskapp
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5000
          env:
            - name: APP_NAME
              value: {{ .Values.webapp.message }}
      volumes:
      - name: nginx-proxy-config
        configMap:
          name: nginx-conf
```

Save with CTRL+S. The finished version is in `lab/code/App/solution-code/app/templates/deployment.yaml`.

6. Render the chart without deploying:

```bash
helm template ./app
```

Check that `{{ .Values.webapp.message }}` became `CloudAcademy.DevOps.K8s.Helm` in the Deployment. The rendered Deployment container section:

```yaml
        - name: flask
          image: cloudacademydevops/flaskapp
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5000
          env:
            - name: APP_NAME
              value: CloudAcademy.DevOps.K8s.Helm
```

This only renders the manifest. Nothing is deployed yet, which makes it a good check before applying.

7. Render and apply in one line:

```bash
helm template cloudacademy ./app | kubectl apply -f -
```

8. List Services:

```bash
kubectl get svc
```

Note the new `cloudacademy-app` Service and its ClusterIP.

9. Store the ClusterIP:

```bash
CLOUDACADEMY_APP_IP=`kubectl get service/cloudacademy-app -o jsonpath='{.spec.clusterIP}'`
echo $CLOUDACADEMY_APP_IP
```

10. Test it:

```bash
curl -i http://$CLOUDACADEMY_APP_IP
```

The body contains `Hello World - CloudAcademy.DevOps.K8s.Helm`.

11. Create dev and prod values files:

```bash
cp app/values.yaml app/values.dev.yaml
cp app/values.yaml app/values.prod.yaml
```

12. In `app/values.dev.yaml`, set `webapp.message` to `CloudAcademy.DevOps.K8s.Helm.Dev` and save with CTRL+S.

13. Deploy with the dev values:

```bash
helm template cloudacademy -f ./app/values.dev.yaml ./app | kubectl apply -f -
```

14. Test again:

```bash
curl -i http://$CLOUDACADEMY_APP_IP
```

The body contains `Hello World - CloudAcademy.DevOps.K8s.Helm.Dev`.

15. In `app/values.prod.yaml`, set `webapp.message` to `CloudAcademy.DevOps.K8s.Helm.Prod` and save with CTRL+S.

16. Deploy with the prod values:

```bash
helm template cloudacademy -f ./app/values.prod.yaml ./app | kubectl apply -f -
```

17. Test again:

```bash
curl -i http://$CLOUDACADEMY_APP_IP
```

The body contains `Hello World - CloudAcademy.DevOps.K8s.Helm.Prod`.

18. Package the chart:

```bash
helm package app/
```

19. Confirm the archive exists:

```bash
ls -la
```

Expect `app-0.1.0.tgz` next to the `app`, `flaskapp`, `k8s`, and `test-app` directories.

20. Delete the resources deployed by the rendered templates:

```bash
helm template cloudacademy -f ./app/values.prod.yaml ./app | kubectl delete -f -
```

21. Install the packaged chart:

```bash
helm install cloudacademy-app app-0.1.0.tgz
```

22. List Helm releases in the current namespace:

```bash
helm ls
```

Example output:

```text
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
cloudacademy-app        cloudacademy    1               2026-08-08 14:38:18.68193263 +0000 UTC  deployed        app-0.1.0       1.0
```

23. Show the created resources:

```bash
kubectl get deploy,pods,svc
```

Example output:

```text
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cloudacademy-app   1/1     1            1           38s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/cloudacademy-app-585cdccbcc-h2jnw   2/2     Running   0          38s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/cloudacademy-app   ClusterIP   10.105.61.106   <none>        80/TCP    38s
```

24. Store the ClusterIP again:

```bash
CLOUDACADEMY_APP_IP=`kubectl get service/cloudacademy-app -o jsonpath='{.spec.clusterIP}'`
echo $CLOUDACADEMY_APP_IP
```

25. Test the Service:

```bash
curl -i http://$CLOUDACADEMY_APP_IP
```

### Validation (Step 7)

- **Check Deployment Exists:** `cloudacademy-app` Deployment with 1 running replica
- **Check the Correct Web Response is Returned:** body contains `Hello World - CloudAcademy.DevOps.K8s.Helm`

---

## Summary

You built a Flask image, ran it behind an NGINX proxy in one Pod, and kept the NGINX config in a ConfigMap mounted over a single file. You fixed a broken image tag using `kubectl get`, `describe`, and `docker images`. Then you scaffolded a chart with `helm create`, rendered manifests with `helm template`, moved the `APP_NAME` string into a Helm value, and reused the same chart for dev and prod through separate values files. Finally you packaged the chart with `helm package` and installed it with `helm install`.
