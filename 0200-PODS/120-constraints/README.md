# Contraints

## Labels

* **Key/Value** pairs
* **Used by selectors** in queries
* Also used by **services and replication controllers** to select managed pods
* Attached to objects
* Grouped by **prefixes** acting as namespaces
* If specified prefixes **must be subdomains** (micompany.com/labelname)
* Prefixes are separated by `/` from the name


## Selectors

* Two types of selectors
* **Equality-based**: `env=prod`, `env!=prod`
* **Set-based**: `env in (production, preproduction)`, `env not in (development), `env`, `!env`
* Used in templates:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
    - name: demo
      image: demo
  nodeSelector:
    disktype: ssd-enabled
```

* Used in queries

```
kubectl get pods --selector env=production,tier=cache
```

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Using labels on nodes

* Show information from nodes, including the labels

```
kubectl get nodes --show-labels
```

* Retrieve the name of a worker node 

```
FRONT_END_NODE="$(kubectl get nodes -o jsonpath='{▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒.name}')"
echo We are going to use $FRONT_END_NODE
```

* Tag it with the new label

```
kubectl ▒▒▒▒▒ ▒▒▒▒ "$FRONT_END_NODE" programar.cloud/$USER-owned=$USER
```

* Create a deployment specifying labels:

```yaml
cat << EOF > nginx-front-end.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      programar.cloud/tier: frontend
  template:
    metadata:
      labels:
        programar.cloud/tier: frontend
    spec:
      ▒▒▒▒▒▒▒▒▒▒▒▒:
        programar.cloud/$USER-owned=$USER
      containers:
      - name: nginx
        image: nginx
EOF
```
* Deploy all pods!

```bash
kubectl apply -f nginx-front-end.yaml
```

* Check the affinity between pods and nodes

```bash
kubectl get pods --output wide
```

* Clean it up

```
kubectl delete deployment frontend
```

## Interpod affinity

In the next scenario we plan to deploy two replicas of a heavy web server only on nodes that contains a node-wide memcached-based cache in order to take advantage of the locality,presumibly by injecting the node name into the client environment and using `spec.containers.ports.hostPort` on the memcached

```yaml
env:
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName 
```

* The *Memcached* database will publish its port **in the host**, so we should ensure two pods don't share the same vm

```yaml
cat << EOF > node-cache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-cache-deployment
spec:
  selector:
    matchLabels:
      app: node-cache
  replicas: 2
  template:
    metadata:
      labels:
        app: node-cache
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - node-cache
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: memcached-server
        image: memcached:alpine
        ports:
          - name: memcached-port
            hostPort: 8888
            containerPort: 11211
            protocol: TCP
EOF
```

* Deploy it with

```
kubectl apply -f node-cache.yaml
```

* Check the two replicas are on separated pods

```
kubectl get pods -owide --sort-by .spec.nodeName
```

* Look at the descriptor of the to learn how to enforce the deployment of the replicas in nodes that contains a `node-cache` but avoiding placing to servers on the same node (yes, yes: the following is also a simplification)

```yaml
cat << EOF > heavy-web-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-web-server-deployment
spec:
  selector:
    matchLabels:
      app: heavy-web-server
  replicas: 2
  template:
    metadata:
      labels:
        app: heavy-web-server
    spec:
      hostNetwork: true
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - node-cache
            topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - heavy-web-server
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: bash
        command: ['sh', '-c', 'echo "This is a very heavy server" && sleep 600']
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName 
EOF
```

* Deploy the webserver and check how it shares node with the cache

```
kubectl apply -f heavy-web-server.yaml
kubectl get pods -owide --sort-by=.spec.nodeName
```

* Check you can access the cache from the other pod:

```bash
HEAVY_POD=$(kubectl get pod -l app=heavy-web-server -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $HEAVY_POD -- bash -c 'echo stats | nc $NODE_NAME 8888'
 ```

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```
