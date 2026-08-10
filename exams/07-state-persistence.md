# Exam 07 - State Persistence

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: State Persistence  
> Checks: 2  
> Solution guide: https://app.qa.com/resource/ckad-practice-exam-state-persistence-solution-guide/  
> Topics: persistentvolume, persistentvolumeclaim, pv, pvc, storageclass, hostpath, accessmodes, capacity, binding, volumes, volumeMounts, multi-container, cluster-scoped, kubectl-explain, kubectl-apply

## Format

Two checks, each a task like the real exam. You may use the official Kubernetes documentation during the exam, but the clock keeps running. Run the checks at any time to see your score.

Solutions are collapsed. Open one only after you have tried the task.

## Fast answers

Both checks are pure YAML. There is no imperative command for a volume, so the speed comes from
copying the right documentation page and editing four fields.

```bash
# 1 - PV is cluster scoped, PVC and Pod are not
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  storageClassName: host
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
EOF

cat << EOF | kubectl -n qq3 apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  storageClassName: host
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: persist
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc
  containers:
    - name: persist
      image: redis
      volumeMounts:
        - mountPath: "/data"
          name: data
EOF

kubectl -n qq3 get pv,pvc,pod        # both Bound, Pod Running

# 2 - one volume in spec.volumes, one volumeMounts entry per container
#     leave hostPath type out unless the directory is known to exist
```

## Exam strategy notes

- **Volumes are the YAML domain.** No `kubectl create` subcommand builds a PV, a PVC, or a volume mount. The fastest route is copying a manifest from the docs and changing the values.
- **Two documentation pages cover the whole domain.** [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) has all three objects for check 1. [Configure a Pod to Use a Volume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/) covers check 2. Bookmark both.
- **Know what is cluster scoped.** PersistentVolume, StorageClass, and Node have no namespace. PersistentVolumeClaim and Pod do. `kubectl api-resources --namespaced=false` prints the full list.
- **`kubectl explain` beats searching.** `kubectl explain persistentvolume.spec` and `kubectl explain pod.spec.volumes` answer "what is this field called" in seconds.
- **Do not add fields the task did not ask for.** Every extra field is a chance to fail a check and costs time you do not have.

---

## Task 1 - Persisting data

Namespace `qq3` · Skill: a static PV, a PVC that binds to it, and a Pod that mounts the claim

**Requirement**

- PersistentVolume named `pv` with `storageClassName: host`, 2Gi of capacity, read-write access for a single Node, and a hostPath of `/mnt/data`
- PersistentVolumeClaim named `pvc` that claims that volume, requesting 1Gi of capacity
- Pod named `persist` with one `redis` container, mounting the `pvc` claim at `/data`

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
  namespace: qq3
  labels:
    app: pv
spec:
  storageClassName: host
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: qq3
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: host
  selector:
    matchLabels:
      app: pv
---
apiVersion: v1
kind: Pod
metadata:
  name: persist
  namespace: qq3
spec:
  containers:
  - name: persist
    image: redis
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc
EOF

kubectl -n qq3 get pod,pvc,pv
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  storageClassName: host
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
EOF

cat << EOF | kubectl -n qq3 apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  storageClassName: host
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

cat << EOF | kubectl -n qq3 apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: persist
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc
  containers:
    - name: persist
      image: redis
      volumeMounts:
        - mountPath: "/data"
          name: data
EOF
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- The documentation task page carries all three manifests already filled in, which is why the guide points at it. Copying and editing beats writing from memory.
- Three things must line up before a PVC binds to a PV: the `storageClassName` strings are equal, the PV offers the access mode the claim asks for, and the PV capacity is at least the requested size. A 1Gi request against a 2Gi PV binds, and the Pod then gets the whole 2Gi.
- Access modes are about **nodes**, not Pods, with one exception. `ReadWriteOnce` is one node read-write, `ReadOnlyMany` is many nodes read-only, `ReadWriteMany` is many nodes read-write, and `ReadWriteOncePod` is the one that means a single Pod. A task saying "a single Node" means `ReadWriteOnce`.
- The Pod refers to the claim, never to the volume. `persistentVolumeClaim.claimName` is the only link.
- Field names come from `kubectl explain persistentvolume.spec` and `kubectl explain persistentvolumeclaim.spec` when the docs page is not enough.
- Check with `kubectl -n qq3 get pv,pvc`. Both must read `Bound`, and the PVC's VOLUME column must name the PV.

Review: [labs/09-persistent-data-with-pods.md](../labs/09-persistent-data-with-pods.md) covers the PVC side with dynamic provisioning on EBS, where the PV is created for you. Writing the PV by hand, as this check needs, is not in the labs.

Docs: [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) · [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

</details>

**Differences that matter**

- `namespace: qq3` on the PersistentVolume is wrong, though it cost nothing. A PV is cluster scoped and belongs to no namespace, so the API accepts the field and drops it. The official solution shows the split clearly: the PV goes in with a plain `kubectl apply`, the PVC and the Pod with `kubectl -n qq3 apply`. The task wording "create a PersistentVolume in the qq3 Namespace" is loose, and only the PVC and the Pod can honour it. `kubectl api-resources --namespaced=false` is how to settle this question in the exam.
- The `labels` plus `selector` pairing does work. Labelling the PV `app: pv` and adding `selector.matchLabels` to the claim pins that claim to that exact volume, which is a real technique when a cluster holds several PVs. Here one PV existed with a matching `storageClassName`, so the binding would have happened anyway, and the pairing is two extra places to make a typo. `spec.volumeName: pv` on the claim does the same pinning in one line.
- `volumeMode: Filesystem` is the default value, so it changes nothing.

---

## Task 2 - Volume mounts

Namespace `blah` · Skill: one hostPath volume mounted into two containers

**Requirement**

Take the manifest below and, before applying it, add a `hostPath` volume named `vol1` pointing at the host path `/tmp/vol`, then mount that volume into both `c1` and `c2` at `/var/log/blah`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    env: prod
  name: logger
  namespace: blah
spec:
  containers:
  - image: bash
    name: c1
    command: ["/usr/local/bin/bash", "-c"]
    args:
    - ifconfig > /var/log/blah/data;
      sleep 3600;
  - image: bash
    name: c2
    command: ["/usr/local/bin/bash", "-c"]
    args:
    - sleep 3600;
```

<details><summary><b>My solution</b></summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    env: prod
  name: logger
  namespace: blah
spec:
  containers:
  - image: bash
    name: c1
    command: ["/usr/local/bin/bash", "-c"]
    args:
    - ifconfig > /var/log/blah/data;
      sleep 3600;
    volumeMounts:
    - name: vol1
      mountPath: /var/log/blah
  - image: bash
    name: c2
    command: ["/usr/local/bin/bash", "-c"]
    args:
    - sleep 3600;
    volumeMounts:
    - name: vol1
      mountPath: /var/log/blah
  volumes:
  - name: vol1
    hostPath:
      path: /tmp/vol
      type: Directory
EOF

kubectl -n blah get pod
```

</details>

<details><summary><b>Official solution</b></summary>

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
 labels:
   env: prod
 name: logger
 namespace: blah
spec:
 containers:
 - image: bash
   name: c1
   command: ["/usr/local/bin/bash", "-c"]
   args:
    - ifconfig > /var/log/blah/data;
      sleep 3600;
   volumeMounts:
   - mountPath: /var/log/blah
     name: vol1
 - image: bash
   name: c2
   command: ["/usr/local/bin/bash", "-c"]
   args:
    - sleep 3600;
   volumeMounts:
   - mountPath: /var/log/blah
     name: vol1
 volumes:
 - name: vol1
   hostPath:
     path: /tmp/vol
EOF
```

</details>

<details><summary><b>Technique and what to review</b></summary>

- Sharing a volume is always the same shape. One entry under `spec.volumes` declares it, one entry under each container's `volumeMounts` consumes it, and the `name` is what ties the three together. The `mountPath` may differ per container.
- `hostPath` mounts a directory from the node's own filesystem. Both containers see the same files because they are in one Pod and therefore on one node.
- The manifest is handed to you, so the whole task is adding two `volumeMounts` blocks and one `volumes` block. Read the given YAML for the indentation style before editing.
- `kubectl explain pod.spec.volumes` and `kubectl explain pod.spec.containers.volumeMounts` give the field names.
- hostPath is a node-local path, so anything written there stays on that node. It is fine for a lab and wrong for real workloads that can reschedule.

Review: [labs/12-ephemeral-volume-types.md](../labs/12-ephemeral-volume-types.md) shares an `emptyDir` between containers with exactly this shape, and [exams/03-multi-container-pods.md](03-multi-container-pods.md) task 1 does the same. Neither uses `hostPath`, which appears nowhere else in these notes.

Docs: [Configure a Pod to Use a Volume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/) · [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

</details>

**Difference that matters**

The one difference is `type: Directory`, and it is a risk rather than an improvement.

That type makes the kubelet check the node before mounting and **require** that `/tmp/vol` already exists as a directory. If it does not, the Pod never starts. It sits in `ContainerCreating` and `kubectl describe pod logger` shows a `hostPath type check failed` event, which fails the check even though every other field is right.

The official leaves `type` out. The default is the empty string, which performs no checks at all, so the Pod starts whether or not the path exists. When you do want the directory created for you, the explicit value is `DirectoryOrCreate`, which makes it with permission 0755.

Rule for the exam: leave `type` out unless the task names it, or use `DirectoryOrCreate` when you need the path to exist.

---

## What to take into the next exam

1. State Persistence is a YAML domain. No imperative command creates a PV, a PVC, or a volume mount, so speed comes from the two documentation task pages.
2. PersistentVolume is cluster scoped. PersistentVolumeClaim and Pod are namespaced. Settle it with `kubectl api-resources --namespaced=false`.
3. A PVC binds when three things line up: equal `storageClassName`, an access mode the PV offers, and PV capacity at least the requested size.
4. `ReadWriteOnce` is one **node**. `ReadWriteOncePod` is one **Pod**. Read which word the task uses.
5. Pin a claim to one specific PV with `spec.volumeName`, or with labels on the PV plus `spec.selector` on the claim. Neither is needed when only one PV can match.
6. Sharing a volume between containers: one `spec.volumes` entry, one `volumeMounts` entry per container, same `name`.
7. hostPath `type: Directory` demands an existing directory on the node. Leave `type` out, or use `DirectoryOrCreate`.
8. Adding fields the task never asked for is unpaid risk. `volumeMode: Filesystem` and a namespace on a PV both did nothing here, and one of them was simply wrong.
