# 04 - Jobs and CronJobs

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Kubernetes Pod Design for Application Developers: Definition Basics  
> Lab: Using Jobs to Manage Pods that Run to Completion  
> Topics: jobs, cronjobs, batch, completions, parallelism, backoffLimit, restartPolicy

## Goal

Explore Jobs and CronJobs for Pods that run to completion (batch work), including retries, parallelism, and scheduled Jobs.

## Notes

- **Deployments** fit long-running Pods that should not exit (for example web servers).
- **Jobs** run a set number of Pods to completion. Failed Pods are replaced until the success count is met. Pods can run in parallel. Completed Pods stay so you can read logs; deleting the Job deletes its Pods.
- **CronJobs** create Jobs on a schedule (like Unix cron).
- Job Pods default to not restarting themselves. The Job controller starts new Pods on failure.
- Useful Job fields: `backoffLimit`, `completions`, `parallelism`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`.
- A common pattern is Jobs plus a queue of work items.
- Cron schedule help: see cron schedule syntax docs. For fields: `kubectl explain cronjob.spec`.

## Steps

### Step 1 - Use Jobs and CronJobs

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace jobs
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=jobs
```

2. Create a Job named `one-off` that sleeps 30 seconds:

```bash
kubectl create job one-off --image=alpine -- sleep 30
```

By default it starts one Pod right away.

3. Read the Job spec:

```bash
kubectl get jobs one-off -o yaml | more
```

Important fields:

- **backoffLimit:** retries before the Job is marked failed
- **completions:** successful Pod completions required
- **parallelism:** how many Pods may run at once
- **spec.template.spec.restartPolicy:** Job Pods usually never restart; the Job manages retries
- The Job uses a selector to track its Pods

4. List other Job spec fields:

```bash
kubectl explain job.spec | more
```

`activeDeadlineSeconds` and `ttlSecondsAfterFinished` help auto-stop and auto-delete Jobs.

5. Create a Job whose Pod always fails:

```bash
cat << 'EOF' > pod-fail.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pod-fail
spec:
  backoffLimit: 3
  completions: 6
  parallelism: 2
  template:
    spec:
      containers:
      - image: alpine
        name: fail
        command: ['sleep 20 && exit 1']
      restartPolicy: Never
EOF
kubectl create -f pod-fail.yaml
```

The Pod fails after 20 seconds (`exit 1`). Up to two Pods run in parallel.

6. Watch Job progress:

```bash
watch kubectl describe jobs pod-fail
```

Expect 2 Running and 0 Succeeded. Events show new Pods as others fail. Eventually an event shows the backoff limit was exceeded and retries stop.

7. Stop watching: Ctrl+C.

8. List Pods:

```bash
kubectl get pods
```

Pods stay until you delete them or their Job. `ttlSecondsAfterFinished` can clean them up automatically.

9. Create a CronJob that runs every minute:

```bash
cat << 'EOF' > cronjob-example.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-example
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: alpine
            name: fail
            command: ['date']
          restartPolicy: Never
EOF
kubectl create -f cronjob-example.yaml
```

Main pieces: `schedule` and `jobTemplate`. More fields: `kubectl explain cronjob.spec`.

10. Confirm Jobs are created every minute:

```bash
watch kubectl describe cronjob cronjob-example
```

## Summary

Jobs and CronJobs manage Pods that run to completion, with optional schedules. Jobs often work with a queue of work items.
