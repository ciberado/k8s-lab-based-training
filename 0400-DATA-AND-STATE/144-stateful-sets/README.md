# StatefulSets

A `statefulset` provides a way to create several `pods` with predictable identities, and an easy way to attach each of them to a particular `persistentVolume`. Also, their creation and deletion follows a [LIFO](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) strategy. Those characteristics makes them suitable for implementing high available databases.

To orchestrate the identity of those `pods`, each `statefulset` requires a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/)

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Resource definition

* Define the `service` that will group the `pods`

```yaml
cat << EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: bash-service
spec:
  clusterIP: None 
  selector:
    app: bash-statefulset
EOF
```

* Create it

```bash
kubectl apply -f service.yaml
```

* Define the `statefulset` with one replica

```yaml
cat << EOF > statefulset.yaml
apiVersion: apps/v1
kind: Stateful▒▒▒
metadata:
  name: bash-statefulset
  labels:
    app: bash-statefulset
spec:
  selector:
    matchLabels:
      app: bash-statefulset
  serviceName: "bash-service" 
  replicas: 1
  template:
    metadata:
      labels:
        app: bash-statefulset
    spec:
      containers:
      - name: main
        image: bash
        command: ['sh', '-c', 'echo Waiting for you! && sleep 36000']
        volumeMo▒▒▒s:
        - name: datavolume
          mountPath: /datavolume
  volumeClaimTemplates:
  - metadata:
      name: datavolume
    spec:
      accessMo▒▒▒: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
EOF
```

* Deploy the resources

```bash
kubectl apply -f statefulset.yaml
```

* Check all the created resources

```bash
kubectl get all
kubectl get pv
kubectl get pvc
```

* See the identifiers of the AWS volumes associated to the `statefulset`

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-for/pvc/namespace,Values=demo-$USER" \
  --query "Volumes[*].VolumeId"
```

* List the directories in the root folder of the created `pod`

```bash
kubectl exec bash-statefulset-0 -- ls /
```

* Create a new file in the mounted directory

```bash
kubectl exec bash-statefulset-0 -- bash -c 'echo "Wop" > /datavolume/wop.txt'
kubectl exec  bash-statefulset-0 -- cat /datavolume/wop.txt
```

* Take note of the original pod IP

```bash
ORIGINAL_IP=$(kubectl get pod -o jsonpath="{.items[0].stat▒▒.podIP}")
```

* Force the recreation of the pod (press `ctrl+c` to exit from the `--watch`)

```bash
kubectl delete pod bash-statefulset-0
kubectl get pod --watch
```

* Check how the IP of the pod has changed during the recreation

```bash
NEW_IP=$(kubectl get pod -o jsonpath="{.items[0].stat▒▒.podIP}")
echo the original IP was $ORIGINAL_IP, but the new one is $NEW_IP.
```

* See if the previously created file is still accessible from the `pod`

```bash
kubectl exec bash-statefulset-0 -- cat /datavolume/wop.txt
```

* Add a new replica to the `statefulset`

```bash
kubectl scale statefulset bash-statefulset --replicas=2
```

* See how a new resources have been provisioned

```bash
kubectl get all
kubectl get pv,pvc
```

* Modify the disk of the second `pod` and reduce the number of replicas

```bash
kubectl exec bash-statefulset-1 -- bash -c 'echo "Wip" > /datavolume/wip.txt'
kubectl exec bash-statefulset-1 -- cat /datavolume/wip.txt
kubectl scale statefulset bash-statefulset --replicas=1
```

* See how (by design) the `persitentvolumes` remain at place, but the second `pod` is gone

```bash
kubectl get pv,pvc
kubectl get pods
```

* Reconstruct the second replica and check if the file is still there

```bash
kubectl scale statefulset bash-statefulset --replicas=2
kubectl get pod
kubectl exec  bash-statefulset-1 -- cat /datavolume/wip.txt
```

* Remove the `statefulset` and see how the `pods` are gone, but the `persistentVolumes` remains by design

```bash
kubectl delete statefulset bash-statefulset
kubectl get pods
kubectl get pv
```

## Cleanup

* Delete the resources from Kubernetes (the AWS volumes will also be removed)

```bash
kubectl delete ns demo-$USER
```

