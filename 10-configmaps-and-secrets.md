# 10 - ConfigMaps and Secrets

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Mastering Kubernetes Pod Configuration: Config Maps and Secrets  
> Lab: Configuring Pods using ConfigMaps / Storing Sensitive Data with Secrets  
> Topics: configmap, secrets, volumes, env, secretKeyRef, configMapRef, base64, stringData

## Goal

Create a ConfigMap and mount it as files in a Pod. Create a Secret and inject it as an environment variable. Keep config and secrets out of images.

## Notes

- **ConfigMaps** hold non-sensitive config as key-value pairs so images stay portable. Namespaced: only Pods in the same Namespace can use them.
- Create ConfigMaps from: env files (`key=value`), files/dirs (filename = key, content = value), `--from-literal`, or a YAML `kind: ConfigMap`.
- Consume ConfigMaps as volumes or env vars (`envFrom.configMapRef`). See `kubectl explain pod.spec.containers.envFrom.configMapRef`.
- Do **not** put passwords or API keys in ConfigMaps. Use **Secrets**.
- Secrets are like ConfigMaps but for sensitive data. Types include generic, TLS, and Docker registry. This lab uses generic.
- Secrets are **base64-encoded**, not encrypted at rest by default. RBAC can still separate who may read Secrets vs ConfigMaps.
- `kubectl create secret` encodes for you. In a manifest `data:` you must base64 yourself, or use `stringData:` and let the API encode. See `kubectl explain secret`.
- Inside the container, env and volume mounts show the **decoded** value.

---

## Step 1 - ConfigMaps as volumes

### Instructions

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace configmaps
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=configmaps
```

2. Create a ConfigMap from literals:

```bash
kubectl create configmap app-config --from-literal=DB_NAME=testdb \
  --from-literal=COLLECTION_NAME=messages
```

3. Show the ConfigMap YAML (same shape as a manifest):

```bash
kubectl get configmaps app-config -o yaml
```

4. Create a Pod that mounts the ConfigMap at `/config`:

```bash
cat << 'EOF' > pod-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
  - image: mongo:4.0.6
    name: mongodb
    # Mount as volume
    volumeMounts:
    - name: config
      mountPath: /config
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: config
    # Declare the configMap to use for the volume
    configMap:
      name: app-config
EOF
kubectl create -f pod-configmap.yaml
```

5. List mounted keys (as files):

```bash
kubectl exec db -it -- ls /config
```

6. Read `DB_NAME`:

```bash
kubectl exec db -it -- cat /config/DB_NAME && echo
```

7. More create examples:

```bash
kubectl create configmap --help | more
```

### Summary (Step 1)

ConfigMaps separate config from images. Mount as volumes or use env vars. Sensitive data belongs in Secrets.

---

## Step 2 - Secrets as environment variables

### Instructions

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace secrets
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=secrets
```

2. Create a generic Secret:

```bash
kubectl create secret generic app-secret --from-literal=password=123457
```

More options: `kubectl create secret generic --help`.

3. Show Secret YAML:

```bash
kubectl get secret app-secret -o yaml
```

Under `data`, `password` looks like `MTIzNDU3` (base64 of `123457`), not plain text.

4. Decode and confirm:

```bash
kubectl get secret app-secret -o jsonpath="{.data.password}" \
  | base64 --decode \
  && echo
```

5. Create a Pod that sets `PASSWORD` from the Secret:

```bash
cat << EOF > pod-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
spec:
  containers:
  - image: busybox:1.30.1
    name: busybox
    args:
    - sleep
    - "3600"
    env:
    - name: PASSWORD      # Name of environment variable
      valueFrom:
        secretKeyRef:
          name: app-secret  # Name of secret
          key: password     # Name of secret key
EOF

kubectl create -f pod-secret.yaml
```

Use `valueFrom.secretKeyRef` for Secret-backed env vars.

6. Print the env var in the container (already decoded):

```bash
kubectl exec pod-secret -- /bin/sh -c 'echo $PASSWORD'
```

### Summary (Step 2)

Secrets hold sensitive key-value data. You can also create them from files and mount them as volumes, same pattern as ConfigMaps.

---

## Summary (both steps)

| Resource   | Use for              | Typical consume        |
|------------|----------------------|------------------------|
| ConfigMap  | Non-sensitive config | Volume or env          |
| Secret     | Sensitive data       | Volume or `secretKeyRef` env |
