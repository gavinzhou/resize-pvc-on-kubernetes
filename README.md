# resize-pvc-on-kubernetes

## create storageclasses

```yaml
cat azure-sc-exp.yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: exp
provisioner: kubernetes.io/azure-disk
parameters:
  cachingmode: None
  kind: Managed
  storageaccounttype: Standard_LRS
allowVolumeExpansion: true  # enable resize pvc
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```sh
kubectl apply -f azure-sc-exp.yaml
```

```sh
kubectl get sc

NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   2d
exp                 kubernetes.io/azure-disk   2d
managed-premium     kubernetes.io/azure-disk   21d
```

## create pvc

```sh
cat pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: exp
```

```sh
kubectl apply -f pvc.yaml
```

```sh
cat nginx-pod.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
        version: v1
    spec:
      containers:
      - name: sleep
        image: busybox
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: data
          mountPath: "/data"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: myclaim
```

## resize pvc

```sh
cat pvc.yaml | sed  "s/2Gi/32Gi/" | kubectl apply -f -
```

## scale deployment pod to 0

```sh
kubectl scale deployment sleep --replicas 0
```

## check resizing pvc status

```sh
kubectl get pvc myclaim -o yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"myclaim","namespace":"default"},"spec":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"32Gi"}},"storageClassName":"exp","volumeMode":"Filesystem"}}
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/azure-disk
  creationTimestamp: 2018-08-30T06:27:51Z
  finalizers:
  - kubernetes.io/pvc-protection
  name: myclaim
  namespace: default
  resourceVersion: "2329927"
  selfLink: /api/v1/namespaces/default/persistentvolumeclaims/myclaim
  uid: d1f91902-ac1d-11e8-a8b1-0265dc9f81fe
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 32Gi
  storageClassName: exp
  volumeName: pvc-d1f91902-ac1d-11e8-a8b1-0265dc9f81fe
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4Gi
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-08-30T06:40:41Z
    status: "True"
    type: Resizing  ## need change to `FileSystemResizePending`
  phase: Bound
```

```sh
kubectl get pvc myclaim -o yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"myclaim","namespace":"default"},"spec":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"32Gi"}},"storageClassName":"exp","volumeMode":"Filesystem"}}
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/azure-disk
  creationTimestamp: 2018-08-30T06:27:51Z
  finalizers:
  - kubernetes.io/pvc-protection
  name: myclaim
  namespace: default
  resourceVersion: "2330145"
  selfLink: /api/v1/namespaces/default/persistentvolumeclaims/myclaim
  uid: d1f91902-ac1d-11e8-a8b1-0265dc9f81fe
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 32Gi
  storageClassName: exp
  volumeName: pvc-d1f91902-ac1d-11e8-a8b1-0265dc9f81fe
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4Gi
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-08-30T06:43:19Z
    message: Waiting for user to (re-)start a pod to finish file system resize of
      volume on node.
    status: "True"
    type: FileSystemResizePending
  phase: Bound
```

## Scale deployment pod to 1

```sh
kubectl scale deployment sleep --replicas 1
```

```sh
kubectl get pvc                                                                                                                       ✘ 130 master ◼
NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim   Bound     pvc-d1f91902-ac1d-11e8-a8b1-0265dc9f81fe   32Gi       RWO            exp            26m
```