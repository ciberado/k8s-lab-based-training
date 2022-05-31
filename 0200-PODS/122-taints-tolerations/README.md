# Taints and tolerations

A *taint* allows a node to refuse a pod unless it is marked with a `toleration. They are sueful to create dedicated nodes, avoid deploying mundane pods in specialized or expensive nodes and expulse pods from existing nodes.

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Tainting nodes

* Get one node, we will use it as our *taint* target

```
export SPECIAL_NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
echo We are going to use $SPECIAL_NODE
```

* Mark it with a *taint* named `special` (this word is arbitrary, could be `potato`). Set its value to `true` and forbid any pod not having that combination from being scheduled on it thanks to the keyword `NoSchedule`. Other possible options are `NoExecute` (will evict existing pods in the node if they don't have the *toleration*) and `PreferNoSchedule`.

```
kubectl taint nodes $SPECIAL_NODE special=true:NoSchedule
kubectl describe node $SPECIAL_NODE | grep Taints
```

* Launch ten replicas of a pod, without the toleration. 

```yaml
cat << EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  selector:
    matchLabels:
      app: demo
  replicas: 10
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: main
        image: bash
        command: ['sh', '-c', 'echo "I am not going to go to the tainted node." && sleep 6000']
EOF

kubectl apply -f deployment.yaml
```

* Check they have not been placed in the special node:

```bash
echo The special node is $SPECIAL_NODE

kubectl get pod \
  -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName \
  --sort-by=.spec.nodeName

TOTAL=$(kubectl get pods -o wide \
  | tail -n "+2" \
  | wc -l)
echo Total number of pods: $TOTAL.

PODS_IN_SPECIAL=$(kubectl get pods -o wide \
  --field-selector spec.nodeName=$SPECIAL_NODE \
  | tail -n "+2" \
  | wc -l)
echo Pods in the special node: $PODS_IN_SPECIAL.
```

* Update the manifest so the `pods` can be deployed in the `tainted` nodes

```yaml
cat << EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  selector:
    matchLabels:
      app: demo
  replicas: 10
  template:
    metadata:
      labels:
        app: demo
    spec:
      tolerations:
      - key: "special"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: main
        image: bash
        command: ['sh', '-c', 'echo "I have no problems with that tainted node." && sleep 6000']
EOF

kubectl apply -f deployment.yaml
```

## Challenge

* Find a way, using `taints`, to evict all the **existing** pods from the `$SPECIAL_NODE` and denying any new scheduling on it unless an additional `toleration` is added to it

<details>
<summary>Solution:</summary>

* For creating the new `taint`

```bash
kubectl taint nodes $SPECIAL_NODE very-special=true:NoExecute
```

* For watching the `pods` transitioning to another node

```
kubectl get pods --watch
```

* Removing the `taint`

```bash
kubectl taint nodes $SPECIAL_NODE very-special=true:NoExecute-
```


</details>

## Clean up

* Remove the taint from the node

```bash
kubectl taint nodes $SPECIAL_NODE special=true:NoSchedule-
```
