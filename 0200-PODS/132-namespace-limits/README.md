# Namespace quotas

* A **ResourceQuota** provides a way to limit the aggregated resources available to a namespace
* A **LimitRange** sets the legal limits for every container deployed in a namespace


## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Namespace Quotas

* Define a `ResourceQuota` to set the maximum size of the `namespace`

```yaml
cat << EOF > resource-quota.yaml
apiVersion: v1
kind: ▒▒▒▒▒▒▒▒▒▒▒▒▒
metadata:
  name: standard-max-resources
spec:
  hard:
    r▒▒▒▒▒▒▒.cpu: 500m
    l▒▒▒▒▒.cpu: 1000m
    r▒▒▒▒▒▒▒.memory: 500Mi
    l▒▒▒▒▒.memory: 1Gi
EOF
```

* Apply it to your `namespace`

```bash
kubectl apply -f resource-quota.yaml -n demo-$USER
```

* Get information about the current occupation

```bash
kubectl describe quota
kubectl describe ns demo-$USER
```

* Create the deployment descriptor 

```bash
cat << EOF > deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon
spec:
  selector:
    matchLabels:
      app: pokemon
  replicas: 1
  template:
    metadata:
      labels:
        app: pokemon
    spec:
      containers:
      - name: pokemon-container
        image: ciberado/pokemon-nodejs:1.0.3
        ▒▒▒▒▒▒▒▒▒:
          ▒▒▒▒▒▒▒▒:
            memory: "100Mi"
            cpu: 100m
          ▒▒▒▒▒▒:
            memory: "200Mi"
            cpu: 200m
EOF
```

* Create the deployment and check it's status

```bash
kubectl apply -f deployment.yaml
kubectl get deployment pokemon --watch
```

* Try to scale it past the limits

```bash
kubectl ▒▒▒▒▒ deployment pokemon --replicas 20
kubectl rollout status -w deployment
```

* Press `ctrl+c` to get back to the prompt and investigate the problem

```bash
kubectl ▒▒▒▒▒▒▒▒ replicaset pokemon
```

* Delete the deployment

```bash
kubectl delete -f deployment.yaml
```

## Range limits

* Create the `LimitRange` manifest

```yaml
cat << EOF > limit-range.yaml
apiVersion: v1
kind: ▒▒▒▒▒▒▒▒▒▒
metadata:
  name: resource-limite
spec:
  ▒▒▒▒▒▒:
  - max:
      memory: 300Mi
    min:
      memory: 100Mi
    type: Container
EOF
```

* Apply it

```bsh
kubectl apply -f limit-range.yaml -n demo-$USER
```

* Define a pod, trying to make it bigger than allowed

```bash
cat << EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: bashpod
spec:
  containers:
    - name: main
      image: bash
      resources:
        limits:
          memory: 512Mi
          cpu: 1000m
EOF
```

* Apply it and see how it fails:

```bash
kubectl apply -f pod.yaml
```

## Challenge

It is possible to create a resource named `PriorityClasses`, as described in the [documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#example-priorityclass), and apply them to any `pod`.

Also, priorities can be assigned to `ResourceQuotas`, so `pods` with different priorities can have different constraints regarding the compute usage.

* Define three priority classes, `low`, `normal` and `high`
* Make the second one the assigned by default
* Create a constrained quota for the `low` and `normal` `pods`, and a more generous one for the `high` ones
* Test all of them

<details>
<summary>Solution</summary>

```yaml
cat << EOF > priorities.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000000
globalDefault: false
---

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: normal
value: 1000
globalDefault: true
---

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
globalDefault: false
EOF
```


```yaml
cat << EOF > quotas.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-resources
spec:
  hard:
    requests.cpu: 1000m
    limits.cpu: 1500m
    requests.memory: 1Gi
    limits.memory: 1.5Gi
    #requests.storage: 1Gi 
    count/services: 2
    count/pods: 10
  scopeSelector:
    matchExpressions:
    - operator : In
      scopeName: PriorityClass
      values: ["low", "normal"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-resources
spec:
  hard:
    requests.cpu: 2000m
    limits.cpu: 3000m
    requests.memory: 2Gi
    limits.memory: 3Gi
    #requests.storage: 2Gi 
    count/services: 4
    count/pods: 20
  scopeSelector:
    matchExpressions:
    - operator : In
      scopeName: PriorityClass
      values: ["high"]
EOF
```

</details>



## Cleanup

* Delete all the resources

```bash
kubectl delete ns demo-$USER
```
