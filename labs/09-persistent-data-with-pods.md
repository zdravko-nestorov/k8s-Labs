# 09 - Using Persistent Data with Pods

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Mastering Kubernetes Pod Configuration: Persistent Data  
> Lab: Using Persistent Data with Pods  
> Topics: persistentvolume, persistentvolumeclaim, pvc, pv, emptyDir, ebs, storageclass, mongodb

## Goal

Use a PersistentVolumeClaim (PVC) so database data survives Pod delete and recreate. See how PVs and dynamic provisioning (EBS / `gp2`) work on this AWS cluster.

## Notes

- **PersistentVolume (PV):** cluster storage that is not tied to one Pod’s lifetime (unlike default `emptyDir` volumes).
- Storage backends can include NFS, iSCSI, and cloud volumes (for example AWS EBS).
- **PersistentVolumeClaim (PVC):** a Pod asks for size and access mode. With dynamic provisioning, the PV is created for you. Treat the PVC as “your storage”.
- This cluster auto-creates EBS volumes for PVCs. StorageClass shown: `gp2`.
- **Access mode example:** `ReadWriteOnce` - one node may mount read/write at a time.
- **Reclaim policy:** `Delete` removes the PV when the PVC is deleted. Other policies can keep the volume.
- `pvc` is a kubectl short name for `persistentvolumeclaim`.

## Steps

### Step 1 - Persist data with a PVC

1. Create a Namespace and set it as the default for the current context:

```bash
# Create namespace
kubectl create namespace persistence
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=persistence
```

2. Create a PVC (2Gi, ReadWriteOnce):

```bash
cat << 'EOF' > pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: db-data
spec:
  # Only one node can mount the volume in Read/Write
  # mode at a time
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF
kubectl create -f pvc.yaml
```

3. Show the PVC:

```bash
kubectl get pvc
```

STATUS may be `Pending` while the PV is created, then `Bound`. STORAGECLASS is `gp2` (EBS).

4. Show the PV:

```bash
kubectl get pv
```

Check RECLAIM POLICY (often `Delete`).

5. Create a MongoDB Pod that mounts the PVC at `/data/db`:

```bash
cat << 'EOF' > db.yaml
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
    - name: data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: data
    # Declare the PVC to use for the volume
    persistentVolumeClaim:
      claimName: db-data
EOF
kubectl create -f db.yaml
```

6. Insert a document and print its message:

```bash
kubectl exec db -it -- mongo testdb --quiet --eval \
  'db.messages.insert({"message": "I was here"}); db.messages.findOne().message'
```

If it errors, retry every 15 seconds until the Pod is ready.

7. Delete the Pod:

```bash
kubectl delete -f db.yaml
```

An `emptyDir` volume would be gone here. The PVC (and PV data) remain.

8. Recreate the Pod (same PVC):

```bash
kubectl create -f db.yaml
```

9. Read the message again:

```bash
kubectl exec db -it -- mongo testdb --quiet --eval 'db.messages.findOne().message'
```

Expect `I was here`. Retry every 15 seconds if needed.

## Validation

- **Pod Database Files Stored on Amazon EBS Persistent Volume:** an EBS volume exists for the PVC.

## Summary

PVCs/PVs keep data independent of Pod lifetime. `emptyDir` does not.
