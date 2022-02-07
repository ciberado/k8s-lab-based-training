# Daemonsets

`DaemonSets` are a way to deploy pods to a subset of the available nodes (including all of them).

See [Taints and tolerations](../122-taints-tolerations/README.md) and [Affinities](../120-constraints/README.md) to better understand how to select the nodes affected by the `DaemonSet`.

Master nodes are usually marked as tainted, but `DaemonSets` can apply a toleration so the corresponding pod appears in the control plane (unless you are using a managed service).

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Deploying a DaemonSet

We are going to deploy an infrastructure `DaemonSet` that will update a file on each host.

<details>
<summary>Challenge: complete the next manifest (by replacing the asterisks) so it deploy a `DaemonSet` to each of the workers


```yaml
cat << EOF > daemonset.yaml
apiVersion: ***********
kind: ***********
metadata:
  name: loop
spec:
  selector:
    matchLabels:
        name: looper
  template:
    metadata:
      labels:
        name: looper
    spec:
      volumes:
        - name: tmphost
          hostPath:
            path: /tmp
      containers:
      - name: web
        image: bash
        command: ['sh', '-c', 'while true; do date >> /host-tmp/loop-$USER; sleep 5; done']
        volumeMounts:
          - name: tmphost
            mountPath: /host-tmp
EOF
```

* Apply the file, once is properly configured:

```bash
kubectl apply -f daemonset.yaml
```

</summary>

### Solution

```yaml
cat << EOF > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: loop
spec:
  selector:
    matchLabels:
        name: looper
  template:
    metadata:
      labels:
        name: looper
    spec:
      volumes:
        - name: tmphost
          hostPath:
            path: /tmp
      containers:
      - name: web
        image: bash
        command: ['sh', '-c', 'while true; do date >> /host-tmp/loop-$USER; sleep 5; done']
        volumeMounts:
          - name: tmphost
            mountPath: /host-tmp
EOF
```
</details>

## Exploring the created resources

* Get information about the `DaemonSet` and its `Pods`

```bash
kubectl get all -owide
```

<details>
<summary>Challenge:  Select the first deployed pod using `-jsonpath` and put its name into a variable

```bash
POD=$(kubectl get pods -ojsonpath='****************')
```
</summary>

### Solution

```bash
POD=$(kubectl get pods -ojsonpath='{.items[0].metadata.name}')
```
</details>

* Use that pod to see how the **nodes** have been affected by the `DaemonSet` (also, ask the trainers to `ssh` into them and show you those files directly)

```bash
kubectl exec -it $POD -- cat /host-tmp/loop-$USER
```

## Rolling upgrade

* Update the manifest so it points to a different version of the image:

```bash
sed -i 's/bash/bash:5.0.18/g' daemonset.yaml
```

* Update the configuration of the `DaemonSet`

```bash
kubectl apply -f daemonset.yaml
```

<details>
<summary>Challenge: find the command required to follow the deployment of the new version of the pods
</summary>

### Solution

```bash
kubectl rollout status ds loop
```
</details>


## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```