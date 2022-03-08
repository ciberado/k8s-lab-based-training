# EKS and EFS

This lab is based on the excellent [Deploying Stateful Microservices](https://www.eksworkshop.com/beginner/190_efs/) one.

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Cluster configuration

Please, **if you are part of a directed training, wait for further instructions before proceed**: the next steps should be run only once per cluster, so there is a good chance your trainer had already executed them.

### Creating the AWS resources

* Get the network configuration
* Define the security group for the EFS mount points
* Create the EFS
* Wait until it is ready
* Create the mount points, one for each subnet
* Wait until they are ready
* Install the [EFS CSI](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html) driver
* Check every resource has been deployed
* Define the manifest for declaring the `StorageClass`

## Creating the disk volume

* Describe the precreated `storageClass`

```bash
kubectl describe sc efs-sc
```

* See how both `StorageClass` and `PersistentVolumes` are **not** namespaced (only `PersistentVolumeClaims` is)

```bash
kubectl api-resources | awk 'NR == 1 || /(sc )|(pv )|(pvc )/'
```

* Create the manifest for defining the `PersistentVolume`
  
```yaml
cat << EOF > efs-persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-persistent-volume-$USER
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesyst▒▒
  accessModes:
    - ReadWriteM▒▒▒
  persistentVolumeReclaimPolicy: Re▒▒▒▒
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: $FILE_SYSTEM_ID
EOF
```

* Deploy the volume

```bash
kubectl apply -f efs-persistent-volume.yaml
```

## Registering access to EFS

* Define the manifest for the `PersistentVolumeClaim`


```yaml
cat << EOF > efs-persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-persistent-volume-claim
spec:
  storageClassName: efs-sc
  volumeName: efs-persistent-volume-$USER
  accessModes:
    - ReadWriteM▒▒▒
  resources:
    requests:
      storage: 5Gi
EOF
```

* Run the `apply` command

```bash
kubectl apply -f efs-persistent-volume-claim.yaml
```

* Check the state of everything

```bash
kubectl get sc,pv,pvc
```

## Using EFS from pods

* Create a `pod` definition

```yaml
cat << EOF > pod-with-efs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-efs
spec:
  containers:
  - name: bash
    image: bash
    command: ['sh', '-c', 'echo Look! There are volumes! && sleep 36000']    
    volumeMounts:
    - mountPath: /test-efs
      name: pod-efs-volume
  volumes:
  - name: pod-efs-volume
    persistentVolumeCl▒▒▒:
      claimName: efs-persistent-volume-▒▒▒▒▒
EOF
```

* Apply it

```bash
kubectl apply -f pod-with-efs.yaml
```

* Check it has been able to mount the `PersistentVolume`

```bash
kubectl get pod pod-with-efs --watch
```

* Write to one file in the shared disk

```bash
kubectl exec -it pod-with-efs -- bash -c \
  "for i in {1..10}; do echo \"Welcome \$i times\" >> /test-efs/hello-$USER.txt; done"
```

* No tricks here, let's remove the `pod`, and then recreate it 

```bash
kubectl delete pod pod-with-efs
kubectl apply -f pod-with-efs.yaml
```

* See how the the shared folder now contains one or more files

```bash
kubectl exec -it pod-with-efs -- ls /test-efs
kubectl exec -it pod-with-efs -- cat /test-efs/hello-$USER.txt
```

## Cleanup

* Start by deleting the local resources

```bash
kubectl delete ns demo-$USER
```

* Remove the *CSI* driver

### Do not run below commands if instructor led!

```bash
kubectl delete -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/"
```

* Unmount the target points, delete the *EFS* disk and remove the *security group*

```bash
MOUNT_TARGET_IDS=$(aws efs describe-mount-targets \
  --file-system-id $FILE_SYSTEM_ID \
  --query MountTargets[].MountTargetId \
  --output text)

for target in ${MOUNT_TARGET_IDS}
do
    echo "Deleting mount target  " $target
    aws efs delete-mount-target \
      --mount-target-id $target
done

aws efs delete-file-system --file-system-id $FILE_SYSTEM_ID

aws ec2 delete-security-group --group-id $MOUNT_TARGET_GROUP_ID
```
