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

* Set the cluster name

```bash
CLUSTER_NAME=$(eksctl get cluster -o json | jq ".[0].metadata.name" -r)
```

### Creating the AWS resources

* Get the network configuration

```bash
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.resourcesVpcConfig.vpcId" --output text)

echo Using VPC $VPC_ID

CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[].CidrBlock" --output text)

echo Main CIDR_BLOCK: $CIDR_BLOCK
```

* Define the security group for the EFS mount points

```bash
MOUNT_TARGET_GROUP_NAME=eks-efs-$USER
MOUNT_TARGET_GROUP_DESC="NFS access to EFS from EKS worker nodes"
MOUNT_TARGET_GROUP_ID=$(aws ec2 create-security-group \
  --group-name $MOUNT_TARGET_GROUP_NAME \
  --description "$MOUNT_TARGET_GROUP_DESC" \
  --vpc-id $VPC_ID  \
  --query GroupId \
  --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $MOUNT_TARGET_GROUP_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $CIDR_BLOCK
echo New security group: $MOUNT_TARGET_GROUP_ID
```

* Create the EFS

```bash
FILE_SYSTEM_ID=$(aws efs create-file-system \
  --query FileSystemId \
  --output text)
echo New EFS: $FILE_SYSTEM_ID
```

* Wait until it is ready

```bash
SLEEP 60
aws efs describe-file-systems --file-system-id $FILE_SYSTEM_ID
```

* Create the mount points, one for each subnet

```bash
TAG1=tag:alpha.eksctl.io/cluster-name
TAG2=tag:kubernetes.io/role/elb
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=$TAG1,Values=$CLUSTER_NAME" "Name=$TAG2,Values=1" \
  --query Subnets[].SubnetId \
  --output text)
echo Affected subnets: $SUBNETS

for subnet in ${SUBNETS}
do
    echo "creating mount target in " $subnet
    aws efs create-mount-target \
      --file-system-id $FILE_SYSTEM_ID \
      --subnet-id $subnet \
      --security-groups $MOUNT_TARGET_GROUP_ID
done
```

* Wait until they are ready

```bash
SLEEP 100
aws efs describe-mount-targets \
  --file-system-id $FILE_SYSTEM_ID \
  --query "MountTargets[].LifeCycleState"
```

* Install the [EFS CSI](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html) driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/"
```

* Check every resource has been deployed

```bash
kubectl get daemonset efs-csi-node -n kube-system --watch

kubectl get deployment efs-csi-controller -n kube-system --watch
```

### Configuring the access

* See how both `StorageClass` and `PersistentVolumes` are **not** namespaced (only `PersistentVolumeClaims` is)

```bash
kubectl api-resources | awk 'NR == 1 || /(sc )|(pv )|(pvc )/'
```

* Define the manifest for declaring the `StorageClass`

```yaml
cat << EOF > efs-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
EOF
```

* Apply the resource 

```bash
kubectl apply -f efs-sc.yaml
```

## Creating the disk volume

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

## Challenge

Look for a way to get access an existing *EFS* volume in read-only mode and check it reading from an existing file or trying to create new content.

<details>
<summary>Solution:</summary>

[Relevant docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
</details>

## Cleanup

* Start by deleting the local resources

```bash
kubectl delete ns demo-$USER
```

* Remove the *CSI* driver

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
