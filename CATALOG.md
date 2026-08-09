# CKAD Catalog

Notes from the course **Cloud Native Champions: CKAD Bootcamp**.

| Folder | Content |
|--------|---------|
| `labs/` | Course labs with worked instructions |
| `exams/` | CKAD practice exams: tasks, my solution, official solution |

Lab numbers follow the order the labs were added. Multi-step labs keep all steps in one file. Exams have their own number series, one file per exam domain.

## Labs

| # | Lab | File | Steps | Topics |
|---|-----|------|-------|--------|
| 00 | Introduction to Kubernetes Playground | [labs/00-introduction-to-kubernetes-playground.md](labs/00-introduction-to-kubernetes-playground.md) | 1 | intro, playground, pods, services, multi-container, service-discovery, deployments, autoscaling, rolling-updates, probes, init-containers, volumes, configmaps, secrets |
| 01 | Pod Definition Basics | [labs/01-pod-definition-basics.md](labs/01-pod-definition-basics.md) | 1 | pods, yaml, manifests, kubectl-explain, kubectl-get, kubectl-create, kubectl-delete |
| 02 | Pod Labels, Selectors, and Annotations | [labs/02-pod-labels-selectors-annotations.md](labs/02-pod-labels-selectors-annotations.md) | 1 | labels, selectors, annotations, namespaces, kubectl-get, kubectl-annotate, kubectl-label |
| 03 | Managing Pods with Deployments | [labs/03-managing-pods-with-deployments.md](labs/03-managing-pods-with-deployments.md) | 1 | deployments, replicas, rolling-updates, rollback, replicaset, services, loadbalancer, kubectl-scale, kubectl-set-image, kubectl-rollout |
| 04 | Jobs and CronJobs | [labs/04-jobs-and-cronjobs.md](labs/04-jobs-and-cronjobs.md) | 1 | jobs, cronjobs, batch, completions, parallelism, backoffLimit, restartPolicy |
| 05 | Container Logging and Sidecar Logging Agent | [labs/05-container-logging-and-sidecar.md](labs/05-container-logging-and-sidecar.md) | 2 | logging, kubectl-logs, kubectl-exec, kubectl-cp, multi-container, sidecar, fluentd, configmap, volumes, s3 |
| 06 | Probes and Application Monitoring | [labs/06-probes-and-monitoring.md](labs/06-probes-and-monitoring.md) | 2 | readiness-probe, liveness-probe, startup-probe, httpGet, tcpSocket, exec-probe, metrics-server, kubectl-top, events, resources |
| 07 | Requesting and Limiting Resources for Pods | [labs/07-requesting-and-limiting-resources.md](labs/07-requesting-and-limiting-resources.md) | 1 | resources, requests, limits, cpu, memory, scheduler, kubectl-top, metrics-server |
| 08 | Configuring Pod Security Contexts | [labs/08-configuring-pod-security-contexts.md](labs/08-configuring-pod-security-contexts.md) | 1 | securityContext, privileged, runAsUser, runAsGroup, runAsNonRoot, readOnlyRootFilesystem, seLinux |
| 09 | Using Persistent Data with Pods | [labs/09-persistent-data-with-pods.md](labs/09-persistent-data-with-pods.md) | 1 | persistentvolume, persistentvolumeclaim, pvc, pv, emptyDir, ebs, storageclass, mongodb |
| 10 | ConfigMaps and Secrets | [labs/10-configmaps-and-secrets.md](labs/10-configmaps-and-secrets.md) | 2 | configmap, secrets, volumes, env, secretKeyRef, configMapRef, base64, stringData |
| 11 | Using Kubernetes ServiceAccounts | [labs/11-using-serviceaccounts.md](labs/11-using-serviceaccounts.md) | 1 | serviceaccount, rbac, identity, least-privilege, projected-volume |
| 12 | Utilizing Ephemeral Volume Types | [labs/12-ephemeral-volume-types.md](labs/12-ephemeral-volume-types.md) | 1 | emptyDir, ephemeral-storage, volumes, volumeMounts, sizeLimit, multi-container |
| 13 | Deploy a Stateful Application (MySQL StatefulSet) | [labs/13-deploy-stateful-application-mysql.md](labs/13-deploy-stateful-application-mysql.md) | 4 | statefulset, headless-service, configmap, storageclass, pvc, pv, ebs, init-containers, sidecar, mysql, replication, probes, drain, cordon, scale, loadbalancer, dashboard, rbac, clusterrolebinding, port-forward, taints, kube-system |
| 14 | Deployment using ConfigMaps and Helm | [labs/14-deployment-with-configmaps-and-helm.md](labs/14-deployment-with-configmaps-and-helm.md) | 7 | helm, helm-create, helm-template, helm-package, helm-install, values-file, configmap, deployment, multi-container, nginx, flask, docker-build, namespace, service, kubectl-expose, kubectl-describe, kubectl-logs, kubectl-exec, errimagepull, troubleshooting |
| 15 | Creating Ingress Rules | [labs/15-creating-ingress-rules.md](labs/15-creating-ingress-rules.md) | 1 | ingress, ingress-controller, nginx-ingress, path-routing, host-routing, virtual-hosting, ingressClassName, rewrite-target, annotations, services, deployments, loadbalancer, kubectl-logs, curl |
| 16 | Deployment Strategies: Canary and Blue/Green | [labs/16-deployment-strategies-canary-blue-green.md](labs/16-deployment-strategies-canary-blue-green.md) | 1 | deployment-strategies, canary, blue-green, rollingupdate, recreate, labels, selectors, services, loadbalancer, kubectl-patch, kubectl-set-image, readiness-probe |
| 17 | Using Custom Resource Definitions (CRDs) | [labs/17-custom-resource-definitions.md](labs/17-custom-resource-definitions.md) | 1 | crd, custom-resources, api-resources, argocd, gitops, kubectl-get, kubectl-describe, kubectl-edit, port-forward, secrets, shortnames |
| 18 | API Access Control: Authentication, Authorization, Admission | [labs/18-api-access-control-authn-authz-admission.md](labs/18-api-access-control-authn-authz-admission.md) | 3 | authentication, authorization, admission-control, kubeconfig, contexts, certificates, rbac, roles, clusterroles, rolebinding, clusterrolebinding, serviceaccount, tokens, projected-volume, can-i, impersonation, kube-apiserver |
| 19 | Getting Started with Docker on Linux | [labs/19-getting-started-with-docker-on-linux.md](labs/19-getting-started-with-docker-on-linux.md) | 7 | docker, docker-install, docker-group, docker-cli, docker-run, docker-ps, docker-exec, docker-logs, docker-search, dockerfile, docker-build, images, containers, layers, dockerhub, port-publishing, ec2 |

## Practice exams

| # | Domain | File | Checks | Topics |
|---|--------|------|--------|--------|
| 01 | Core Concepts | [exams/01-core-concepts.md](exams/01-core-concepts.md) | 6 | pods, namespaces, labels, restartPolicy, jsonpath, dry-run, kubectl-run, kubectl-label, kubectl-edit, kubectl-explain, terminationGracePeriodSeconds, env |
| 02 | Configuration | [exams/02-configuration.md](exams/02-configuration.md) | 5 | secrets, configmaps, volumes, envFrom, serviceaccount, securityContext, runAsUser, fsGroup, resources, requests, limits, labels, multi-container, kubectl-create, kubectl-set, kubectl-patch, dry-run |
| 03 | Multi-Container Pods | [exams/03-multi-container-pods.md](exams/03-multi-container-pods.md) | 3 | multi-container, sidecar, emptyDir, volumes, volumeMounts, readOnly, logging, tail, pod-networking, localhost, dns, configmap, configMapKeyRef, postStart, lifecycle, kubectl-replace, kubectl-logs |
| 04 | Observability | [exams/04-observability.md](exams/04-observability.md) | 5 | probes, livenessProbe, readinessProbe, httpGet, troubleshooting, endpoints, services, nodeport, kubectl-logs, label-selector, kubectl-exec, kubectl-cp, kubectl-top, metrics-server, kubectl-edit, kubectl-describe |
| 05 | Pod Design | [exams/05-pod-design.md](exams/05-pod-design.md) | 6 | deployments, replicas, scale, set-image, rollout-history, change-cause, annotations, labels, label-selector, set-based-selector, hpa, autoscale, cronjob, schedule, jsonpath, sort-by, custom-columns, kubectl-create |

Fastest commands across all exams: [exams/CHEATSHEET.md](exams/CHEATSHEET.md). Solutions inside exam files are collapsed, so a file can be re-attempted without spoilers.

## Topic index

Lab numbers unless marked with E, which means an exam.

| Topic | Labs |
|-------|------|
| Pods and manifests | 00, 01, E01, E02 |
| Labels, selectors, annotations | 02, 16, E01, E02, E05 |
| Namespaces | 02, E01 |
| Deployments, rollouts, rollbacks | 00, 03, 14, 16, E05 |
| Autoscaling (HPA) | 00, E05 |
| Deployment strategies (canary, blue/green) | 16 |
| StatefulSets and headless Services | 13 |
| Services and exposure | 00, 03, 05, 13, 14, 15, 16, E04 |
| Ingress and HTTP routing | 15 |
| Jobs and CronJobs | 04, E05 |
| Logging | 00, 05, 14, E03, E04 |
| Multi-container and sidecars | 00, 05, 12, 13, 14, E02, E03 |
| Pod networking (localhost, Pod DNS) | E03 |
| Init containers | 00, 13 |
| Probes | 00, 06, 13, 16, E04 |
| Monitoring and metrics-server | 00, 06, 07, 13, E04 |
| Resources, requests, limits | 00, 07, 12, E02 |
| Security contexts | 08, E02 |
| Persistent volumes and PVCs | 00, 09, 13 |
| StorageClasses and dynamic provisioning | 13 |
| Ephemeral volumes | 12, E03 |
| ConfigMaps and Secrets | 00, 10, 13, 14, 17, E02, E03 |
| ServiceAccounts and RBAC | 11, 13, 18, E02 |
| API access control (authn, authz, admission) | 18 |
| kubeconfig and contexts | 18 |
| CRDs and custom resources | 17 |
| API discovery (api-resources, explain) | 01, 17, E01, E02 |
| Imperative kubectl and dry-run | E01, E02, E05 |
| Updating live objects (set, patch, edit, replace) | 03, 16, 17, E01, E02, E03, E04, E05 |
| JSONPath and output formats | 10, 12, 13, 14, 15, 16, 17, E01, E05 |
| Helm charts, templates, values | 14 |
| Troubleshooting (describe, events, endpoints) | 06, 14, 15, E04 |
| Cluster operations (drain, cordon, scale) | 13 |
| Kubernetes Dashboard and port-forward | 13, 17 |
| Docker image build | 14, 19 |
| Docker basics (CLI, containers, Dockerfile) | 19 |

## How this is used

The `ckad-labs` skill in `.cursor/skills/ckad-labs/SKILL.md` reads this catalog first, then greps and opens only the files it needs.

Add a row here whenever a new lab or exam file is created.
