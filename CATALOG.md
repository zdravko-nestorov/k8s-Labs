# CKAD Labs Catalog

Lab notes from the course **Cloud Native Champions: CKAD Bootcamp**.

Numbers follow the order the labs were added. Multi-step labs keep all steps in one file.

## Labs

| # | Lab | File | Steps | Topics |
|---|-----|------|-------|--------|
| 00 | Introduction to Kubernetes Playground | [00-introduction-to-kubernetes-playground.md](00-introduction-to-kubernetes-playground.md) | 1 | intro, playground, pods, services, multi-container, service-discovery, deployments, autoscaling, rolling-updates, probes, init-containers, volumes, configmaps, secrets |
| 01 | Pod Definition Basics | [01-pod-definition-basics.md](01-pod-definition-basics.md) | 1 | pods, yaml, manifests, kubectl-explain, kubectl-get, kubectl-create, kubectl-delete |
| 02 | Pod Labels, Selectors, and Annotations | [02-pod-labels-selectors-annotations.md](02-pod-labels-selectors-annotations.md) | 1 | labels, selectors, annotations, namespaces, kubectl-get, kubectl-annotate, kubectl-label |
| 03 | Managing Pods with Deployments | [03-managing-pods-with-deployments.md](03-managing-pods-with-deployments.md) | 1 | deployments, replicas, rolling-updates, rollback, replicaset, services, loadbalancer, kubectl-scale, kubectl-set-image, kubectl-rollout |
| 04 | Jobs and CronJobs | [04-jobs-and-cronjobs.md](04-jobs-and-cronjobs.md) | 1 | jobs, cronjobs, batch, completions, parallelism, backoffLimit, restartPolicy |
| 05 | Container Logging and Sidecar Logging Agent | [05-container-logging-and-sidecar.md](05-container-logging-and-sidecar.md) | 2 | logging, kubectl-logs, kubectl-exec, kubectl-cp, multi-container, sidecar, fluentd, configmap, volumes, s3 |
| 06 | Probes and Application Monitoring | [06-probes-and-monitoring.md](06-probes-and-monitoring.md) | 2 | readiness-probe, liveness-probe, startup-probe, httpGet, tcpSocket, exec-probe, metrics-server, kubectl-top, events, resources |
| 07 | Requesting and Limiting Resources for Pods | [07-requesting-and-limiting-resources.md](07-requesting-and-limiting-resources.md) | 1 | resources, requests, limits, cpu, memory, scheduler, kubectl-top, metrics-server |
| 08 | Configuring Pod Security Contexts | [08-configuring-pod-security-contexts.md](08-configuring-pod-security-contexts.md) | 1 | securityContext, privileged, runAsUser, runAsGroup, runAsNonRoot, readOnlyRootFilesystem, seLinux |
| 09 | Using Persistent Data with Pods | [09-persistent-data-with-pods.md](09-persistent-data-with-pods.md) | 1 | persistentvolume, persistentvolumeclaim, pvc, pv, emptyDir, ebs, storageclass, mongodb |
| 10 | ConfigMaps and Secrets | [10-configmaps-and-secrets.md](10-configmaps-and-secrets.md) | 2 | configmap, secrets, volumes, env, secretKeyRef, configMapRef, base64, stringData |
| 11 | Using Kubernetes ServiceAccounts | [11-using-serviceaccounts.md](11-using-serviceaccounts.md) | 1 | serviceaccount, rbac, identity, least-privilege, projected-volume |
| 12 | Utilizing Ephemeral Volume Types | [12-ephemeral-volume-types.md](12-ephemeral-volume-types.md) | 1 | emptyDir, ephemeral-storage, volumes, volumeMounts, sizeLimit, multi-container |
| 13 | Deploy a Stateful Application (MySQL StatefulSet) | [13-deploy-stateful-application-mysql.md](13-deploy-stateful-application-mysql.md) | 4 | statefulset, headless-service, configmap, storageclass, pvc, pv, ebs, init-containers, sidecar, mysql, replication, probes, drain, cordon, scale, loadbalancer, dashboard, rbac, clusterrolebinding, port-forward, taints, kube-system |
| 14 | Deployment using ConfigMaps and Helm | [14-deployment-with-configmaps-and-helm.md](14-deployment-with-configmaps-and-helm.md) | 7 | helm, helm-create, helm-template, helm-package, helm-install, values-file, configmap, deployment, multi-container, nginx, flask, docker-build, namespace, service, kubectl-expose, kubectl-describe, kubectl-logs, kubectl-exec, errimagepull, troubleshooting |

## Topic index

| Topic | Labs |
|-------|------|
| Pods and manifests | 00, 01 |
| Labels, selectors, annotations | 02 |
| Deployments, rollouts, rollbacks | 00, 03, 14 |
| StatefulSets and headless Services | 13 |
| Services and exposure | 00, 03, 05, 13, 14 |
| Jobs and CronJobs | 04 |
| Logging | 00, 05, 14 |
| Multi-container and sidecars | 00, 05, 12, 13, 14 |
| Helm charts, templates, values | 14 |
| Troubleshooting (describe, events, image pull) | 06, 14 |
| Docker image build | 14 |
| Init containers | 00, 13 |
| Probes | 00, 06, 13 |
| Monitoring and metrics-server | 00, 06, 07, 13 |
| Resources, requests, limits | 00, 07, 12 |
| Security contexts | 08 |
| Persistent volumes and PVCs | 00, 09, 13 |
| StorageClasses and dynamic provisioning | 13 |
| Ephemeral volumes | 12 |
| ConfigMaps and Secrets | 00, 10, 13, 14 |
| ServiceAccounts and RBAC | 11, 13 |
| Cluster operations (drain, cordon, scale) | 13 |
| Kubernetes Dashboard and port-forward | 13 |

## How this is used

The `ckad-labs` skill in `.cursor/skills/ckad-labs/SKILL.md` reads this catalog first, then greps and opens only the labs it needs.

Add a row here whenever a new lab file is created.
