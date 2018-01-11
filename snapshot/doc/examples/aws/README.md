## AWS EBS

[![asciicast](https://asciinema.org/a/5jfggavfbkayuf7lkpe6n7li1.png)](https://asciinema.org/a/5jfggavfbkayuf7lkpe6n7li1)

### Prerequisites
- On AWS install and run a Kubernetes cluster, see [Kubernetes on AWS Getting Started Guides](https://kubernetes.io/docs/getting-started-guides/aws/) for examples.
- Alternatively, not listed in the official getting started guides, you can also run a single node Kubernetes cluster using local-up-cluster.sh

### Start the Snapshot Controller and Provisioner 

 * Create and run the Secrets for your AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY called [secret.yaml](../../../deploy/kubernetes/aws/secret.yaml)
```bash
kubectl create -f secret.yaml
```

 * Create and run the Roles (RBAC) for the Controller and Provisioner called [snapshot-rbac.yaml](../../../deploy/kubernetes/aws/snapshot-rbac.yaml)
```bash
kubectl create -f snapshot-rbac.yaml
```

* Create and run the Deployment for the Snapshot Controller and Provisioner and call it [deployment.yaml](../../../deploy/kubernetes/aws/snapshot-rbac.yaml)
```bash
kubectl create -f deployment.yaml
```

* Validate that the Snapshot Controller and Provisioner started
```bash
kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
snapshot-controller-7bb857f5dc-s9hvk   2/2       Running   0          47s

```
###  Create a Dynamic Persistent Volume to use as a Snapshot
 * Create a work Namespace called *myns*.
```bash
kubectl create namespace myns
```

 * Create a [Storage Class](https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/persistent-volume-provisioning/aws-ebs.yaml) and run it.
```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/persistent-volume-provisioning/aws-ebs.yaml
```

 * Create the PVC [ebs-pvc.yaml](pvc.yaml) referencing the previous created Storage Class (slow) and run it.
```bash
kubectl create -f pvc.yaml

# Check Status of the PV and PVC
kubectl get pvc -n myns
NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-pvc   Bound     pvc-351f5fc3-f66f-11e7-9ee7-0eef3fbd31d8   4Gi        RWO            slow           23s

```

 * Create a [Snapshot Third Party Resource](snapshot.yaml) and run it. 
```bash
kubectl create -f snapshot.yaml
```

### Check VolumeSnapshot and VolumeSnapshotData are created

```bash
kubectl get volumesnapshot,volumesnapshotdata -o yaml --namespace=myns

apiVersion: v1
items:
- apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    clusterName: ""
    creationTimestamp: 2018-01-11T01:34:22Z
    generation: 0
    labels:
      SnapshotMetadata-PVName: pvc-351f5fc3-f66f-11e7-9ee7-0eef3fbd31d8
      SnapshotMetadata-Timestamp: "1515634526910793155"
    name: snapshot-demo
    namespace: myns
    resourceVersion: "1178"
    selfLink: /apis/volumesnapshot.external-storage.k8s.io/v1/namespaces/myns/volumesnapshots/snapshot-demo
    uid: 8d39e74c-f66f-11e7-9ee7-0eef3fbd31d8
  spec:
    persistentVolumeClaimName: ebs-pvc
    snapshotDataName: ""
  status:
    conditions: null
    creationTimestamp: null
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""

```

### Creating a New PV based on the Above Snapshot.

Unlike existing PV provisioners that provision blank volume, Snapshot based PV provisioners create volumes based on existing snapshots. Thus new provisioners are needed.

There is a special annotation given to PVCs that request snapshot based PVs. `snapshot.alpha.kubernetes.io/snapshot` must point to an existing VolumeSnapshot Object
```yaml
metadata:
  name: 
  namespace: 
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: snapshot-demo
```


#### Create Storage Class [snapshot-promoter-storage-class.yaml](class.yaml) to enable provisioner PVs based on volume snapshots

```bash
kubectl create -f snapshot-promoter-storage-class.yaml
```

#### Create a PVC called [snapshot-claim.yaml](claim.yaml) that restores from an existing volume snapshot

```bash
kubectl create -f snapshot-claim.yaml

# kubectl get pvc -n myns
NAME                            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
ebs-pvc                         Bound     pvc-351f5fc3-f66f-11e7-9ee7-0eef3fbd31d8   4Gi        RWO            slow                25m
snapshot-pv-provisioning-demo   Pending                    
```
