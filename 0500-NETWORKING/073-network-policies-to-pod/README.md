# Networks policies

* Network policies are implemented by some CNI providers such as [Calico](https://docs.projectcalico.org/v2.0/getting-started/kubernetes/)

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Install Calico Network Policies support on EKS

* Using `helm`, deploy the *operator* in charge of creating the different resources required to use `network policies`

```bash
helm repo add projectcalico https://docs.projectcalico.org/charts
helm repo update
helm install -n default calico projectcalico/tigera-operator --version v3.21.4
```

* Check how everything is in place

```bash
helm ls
kubectl get all -n tigera-operator
kubectl get all -n calico-system
```

## Scenario definition

* Create a `pod` running a `bash` container 

```yaml
cat << EOF > bash.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bash
  labels:
    app: bash-app
spec:
  containers:
  - name: bash
    image: bash
    command: ['sh', '-c', 'echo bash pod started && sleep 60000']
EOF
```

* Execute the manifest

```bash
kubectl apply -f bash.yaml
```

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

* Deploy all the resources with

```bash
kubectl apply -f bash.yaml,nginx.yaml
```

## Customizing firewall controls between workloads

* Check how there is no firewall by default preventing reaching the `nginx` pod from anywhere

```bash
kubectl exec -it bash -- wget -O- -q nginx-service
```

* Create a simple `networkPolicy` that will select all the `pods` in the current `namespace` and block *ingress* traffic to them

```yaml
cat << EOF > deny-traffic-in-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
spec:
  podSe▒▒▒▒▒▒: {}
  policyTypes:
  - Ing▒▒▒▒
EOF
```

* Apply the policy

```bash
kubectl apply -f deny-traffic-in-namespace.yaml
```

* See how it has been created

```bash
kubectl get networkpolicies
kubectl describe networkpolicy default-deny-all-ingres
```

* Test how now it is not possible to reach the `nginx` pod from the `bash` one

```bash
kubectl exec -it bash -- wget -O- -q nginx-service --timeout 5
```

```yaml
cat << EOF > allow-ingress-from-bash.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-nginx-from-bash
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
    - Ing▒▒▒▒
  ingress:
    - from:
      - podSe▒▒▒▒▒▒:
          matchLabels:
            app: bash-app
EOF
```

* Using the command line tool, set a network policy with 

```
kubectl apply -f allow-ingress-from-bash.yaml
```

* Try again connecting to the `bash` `pod` to the `nginx` one

```bash
kubectl exec -it bash -- wget -O- -q nginx-service
```


* Launch another pod without the required `label` (`app: bash-app`) and check how the connectivity is not allowed 

```bash
kubectl run another-bash --rm -ti --image bash \
  -- wget -O- -q nginx-service --timeout 5
```

## Clean up

* Uninstall the release with calico

```bash
helm uninstall calico -n default
```

* Delete the created resources

```bash
kubectl delete ns demo-$USER
```

## Additional resources

* An [amazing collection of recipes](
https://github.com/ahmetb/kubernetes-network-policy-recipes
) created by [Ahmet Alp Balkan](https://github.com/ahmetb)
