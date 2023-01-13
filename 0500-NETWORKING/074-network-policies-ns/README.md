# Namespace isolation with network policies

## Preparation

* Install Calico Network Policies support on EKS, if it is not already present (we are using the `default` `namespace` to keep the state of the `release`, but this can't be considered a good practice)

```bash
helm repo add projectcalico https://docs.projectcalico.org/charts
helm repo update
helm install calico projectcalico/tigera-operator --version v3.21.4 -n default
```

* Define two namespaces, one for development and another one for production (see the `labels`)

```bash
cat << EOF > namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-dev-$USER
  labe▒▒:
    environment: dev

---

apiVersion: v1
kind: Namespace
metadata:
  name: demo-prod-$USER
  labe▒▒:
    environment: prod
EOF
```

* Create them

```bash
kubectl apply -f namespaces.yaml
```

## Deploy the production server

* Define another `pod` (with an `nginx` container) and publish it using a `service`

```yaml
cat << EOF > nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-app
  type: ClusterIP
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx-app
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

* Deploy all the resources in the production environment

```bash
kubectl apply -f nginx.yaml -n demo-prod-$USER
```

* See how it is possible to reach it from any `namespace`

```bash
kubectl run bash -n demo-dev-$USER --rm -ti --image bash \
  -- wget -O- -q nginx-service.demo-prod-$USER.svc.cluster.local

kubectl run bash -n demo-prod-$USER --rm -ti --image bash \
  -- wget -O- -q nginx-service
```

## Isolating namespaces

* Create a `networkpolicy` attached to the production `namespace` allowing traffic from any `pod` belonging to itself (check the explicit usage of `metadata.namespace`)

```yaml
cat << EOF > production-network-policy.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-prod-from-prod
  namespace: demo-prod-$USER
spec:
  podSe▒▒▒▒▒▒: {}
  ingress:
    - from:
      - namespaceSe▒▒▒▒▒▒:
          matchLab▒▒▒:
            environment: prod
EOF
```

* Deploy the policy

```bash
kubectl apply -f production-network-policy.yaml
```

* See how it is **NOT** possible to reach it from the development `namespace`

```bash
kubectl run another-bash -n demo-dev-$USER --rm -ti --image bash \
  -- wget -O- -q nginx-service.demo-prod-$USER.svc.cluster.local --timeout 5
```

* But can still connect to it from another `pod` created in the production `namespace`

```bash
kubectl run another-bash -n demo-prod-$USER --rm -ti --image bash \
  -- wget -O- -q nginx-service
```

## Cleanup

* Uninstall Calico, if required

```bash
helm uninstall calico -n default
```

* Delete the namespaces

```bash
kubectl delete -f namespaces.yaml
```

## Additional resources

* [Illustrated introduction to IPTables](https://iximiuz.com/en/posts/laymans-iptables-101/) is an excellent post detailing the behavior of *netfilter*.
