# EBS volumes


## Key concepts

* awsEBS mounts/unmounts EBS disks to containers
* Only available for EC2 instances sharing AZ with the pod
* `gp2` *StorageClass* will automatically provision the *PVC* and *PV* resources if not explicitly defined

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Creating AWS EBS Disk

* Check the existing `storageclasses` to see if your cluster can manage EBS volumes

```bash
kubectl get storageclasses
```

* Get one random node to ensure we do not deploy a disk in an Availability Zone that is not being used by the cluster

```bash
MAX=$(($(kubectl get nodes --label-columns failure-domain.beta.kubernetes.io/zone | wc -l) -1))
echo There are $MAX nodes in the cluster.
NODE_IDX=$(($RANDOM % $MAX))
echo We will work with node number $NODE_IDX.
NODE_NAME=$(kubectl get nodes -o jsonpath="{.items[$NODE_IDX].metadata.name}")
echo Your node is going to be $NODE_NAME.
```

* Take note of its AZ

```bash
AZ=$(kubectl get node $NODE_NAME -o json | \
       jq '.metadata.labels | {"topology.kubernetes.io/zone"}[]' -r)
echo Using Availability Zone $AZ.
```

* Create the volume on that AZ

```bash
VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone=$AZ \
  --size=10 --volume-type=gp2 \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=training-k8s-$USER}]" \
  --query VolumeId  \
  --output text)
echo Your new volume is $VOLUME_ID.
```

## Define the Storage Class

* Create the manifest for it

```yaml
cat << EOF > storage-class.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-$USER
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
mountOptions:
  - debug
volumeBindingMode: Immediate
EOF
```

* Apply it

```bash
kubectl apply -f storage-class.yaml
```

* See if it is created correctly and compare its configuration with the `gp2 (default)` one

```bash
kubectl get sc
```

## Declare the Persistent Volume

* See how `persistentVolumes` are not namespace-scoped

```bash
kubectl api-resources | grep persistentvolumes
```

* Yes, the manifest, here it is:

```yaml
cat << EOF > pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: datadisk-$USER
  labels:
    type: amazonEBS
spec:
  storageClassName: ebs-$USER
  capacity:
    storage: 10Gi
  accessModes:
    - ReadW▒▒▒▒On▒▒
  awsElasticBlockStore:
    volumeID: $VOLUME_ID
    fsType: ext4
    readOnly: false
EOF
```

* Apply that manifest

```bash
kubectl apply -f pv.yaml
```

* See the details of the resource

```bash
kubectl describe pv datadisk-$USER
```

* Check how there is not any attachment to the EBS volume

```
aws ec2 describe-volumes --volume-id $VOLUME_ID --query Volumes[].Attachments
```

## Establishing the volume claim

* Remember: `pvcs` are namespaced resources

```bash
kubectl api-resources | grep persistentvolumeclaim
```


* Create the manifest

```yaml
cat << EOF > claim.yaml
kind: PersistentVolumeCl▒▒▒
apiVersion: v1
metadata:
  name: datadisk-claim
spec:
  storageClassName: ebs-$USER
  volumeName: datadisk-$USER
  accessModes:
    - ReadW▒▒▒▒O▒▒▒
  resources:
    requests:
      storage: 10Gi
EOF
```

* Create the resource

```bash
kubectl apply -f claim.yaml
```

* Get the details about it

```bash
kubectl get pvc datadisk-claim
```

* Take a look at the details of the `pvc`

```bash
kubectl describe pvc datadisk-claim
```

* Check how there is not any attachment to the EBS volume

```
aws ec2 describe-volumes --volume-id $VOLUME_ID --query Volumes[].Attachments
```

* And see how, now, the EBS resource is attached to an EC2 instance

```bash
aws ec2 describe-volumes --volume-id $VOLUME_ID --query Volumes[].Attachments
```

## Mounting EBS disks on pods

* Declare a pod with with the volume attached, and the affinity to the pod

```yaml
cat << EOF > pod-with-ebs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-ebs
spec:
  nodeSelector:
    topology.kubernetes.io/zone: $AZ
  containers:
  - name: bash
    image: bash
    command: ['sh', '-c', 'echo Look! There are volumes! && sleep 36000']    
    volumeMounts:
    - name: datadisk
      mountPath: /test-ebs
  volumes:
    - name: datadisk
      persistent▒▒▒▒▒▒Claim:
        claimName: datadisk-claim
EOF
```

* Don't forget to create the `pod`

```bash
kubectl apply -f pod-with-ebs.yaml
```

* Check if everything is ok

```bash
kubectl describe pod pod-with-ebs
```

* Finally, the EBS disk is attached to an EC2 instance

```bash
aws ec2 describe-volumes \
  --volume-id $VOLUME_ID \
  --query Volumes[].Attachments[0].InstanceId

kubectl get node $NODE_NAME -o jsonpath="{.spec.providerID}" && echo
```

## Using the persistent disk

* Write a file in the EBS volume

```bash
kubectl exec -it pod-with-ebs -- bash -c \
  'for i in {1..10}; do echo "Welcome $i times" >> /test-ebs/hello.txt; done'
```

* Check its content

```bash
kubectl exec -it pod-with-ebs -- bash -c "cat /test-ebs/hello.txt"
```

## Challenge: checking the attachment

Use the [AWS CLI](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html) to check if the EBS disk is actually attached to the corresponding node by using its `nodeName`.

* Get the information regarding the disk, including the ID of the EC2 instance
* Use that ID to get the hostname of the virtual machine
* Using good old `kubectl`, get the name of the node
* Compare them, you should see the same string

<details>
<summary>Solution:</summary>

* Get the EC2 instance name from the description of the EBS volume

```bash
INSTANCE_ID=$(aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query "Volumes[0].Attachments[0].InstanceId" \
  --output text)
echo The instance for your volume is $INSTANCE_ID.
```

* From there, get the DNS name of the virtual machine

```bash
INSTANCE_NAME=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PrivateDnsName" \
  --output text)
echo The instance name is $INSTANCE_NAME
```

* And now, compare it with the node in which your pod is being run (they should match!)

```bash
NODE_ID=$(kubectl get pod pod-with-ebs -o jsonpath={.spec.nodeName}) 
echo And your node name is $NODE_ID
```
</details>

## Sharing the volume

* Deploy new pods based on the same configuration (including the `pvc`)

```bash
for i in {1..10} 
do 
  cat pod-with-ebs.yaml \
  | sed "s/pod-with-ebs/another-pod-with-ebs-${i}/" \
  | kubectl apply -f -
done
```

* Apply it and see how it is possible to get the new pod sharing the `persistentvolume` with the original one, because the `pvc` is acting as an affinity trait for launching them in the same node

```bash
kubectl get pods -owide
```

## Cleanup

* Delete the namespace

```bash
kubectl delete ns demo-$USER
```

* Wait a minute so the EBS volume can be detached from the instance, and delete it

```bash
sleep 60
aws ec2 delete-volume --volume-id $VOLUME_ID
```
