# Stateful applications

## Stateful applications

* The Pod storage is by default **ephemeral**
* It is possible to use **stateful** pods
* In many cases **it may not be the best solution**
* For slow evolving production systems on AWS **we recommend RDS**

## Volumes

* A `PersistentVolume` is a directory accessible to all the containers in a pod
* Survives the restart of a container
* Each container of the pod specifies the volume mount point
* An `StorageClass` provides the ability to create volumes
* To use a `PersistentVolume` we declare a `PersistentVolumeClaims`
* `PVC` can be ReadWriteOnce, ReadOnlyMany and ReadyWriteMany 
* AWS EBS is an example of RWO
* Google disks is an example of ROX
* AWS EFS is an example of RWM-

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Emptydirs

* Declare a pod with with two containers sharing the same `emptydir` volume

```yaml
cat << EOF > pod-with-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-volume
spec:
  volumes:
  - name: my-pod-storage
    emptyDir: {}
  containers:
  - name: alpha
    image: bash
    command: ['sh', '-c', 'echo "I am alpha." && sleep 6000']
    volumeMounts:
    - name: my-pod-storage
      mountPath: /data/alpha
  - name: beta
    image: bash
    command: ['sh', '-c', 'echo "I am beta." && sleep 6000']
    volumeMounts:
    - name: my-pod-storage
      mountPath: /data/beta
EOF
```

* Apply the manifest

```bash
kubectl apply -f pod-with-volume.yaml
```

* Use one container to create a file on the `volume`

```bash
kubectl exec -it pod-with-volume -▒ alpha -- bash -c "echo hi! > /data/alpha/hi.txt"
```

* Read the content of the file from the second container

```bash
kubectl exec -it pod-with-volume -▒ beta -- bash -c "cat /data/beta/*"
```

* Check how the volume is not a `persistentVolume` resource

```bash
kubectl get pv
```

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```