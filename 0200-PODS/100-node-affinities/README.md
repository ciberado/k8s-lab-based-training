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
* **Set-based**: `env in (production, preproduction)`, `env not in (development)`, `env`, `!env`

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Using labels on nodes

* Show information from nodes, including the labels

```bash
kubectl get nodes --show-labels
```

* Maybe we can use a level 9 invocation spell to properly format the labels:

```bash
kubectl get nodes --show-labels=true; echo ''; kubectl get nodes --show-labels=true --no-headers=true | head -n 1 | awk '{print $6}' | perl -pe 's/,/\n/g'
```

* Explore, if you want the `json` output for your cluster infrastructure

```bash
kubectl get nodes -o json | jid
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

* Create a deployment specifying labels for selecting the node:

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
        programar.cloud/$USER-owned: $USER
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

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```
