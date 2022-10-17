# Service discovery

## Services

* May manage **several replicas of the pods**
* They are the natural k8s **load balancers between replicas**
* Pods are managed by a service using **label selectors**
* Each service has its own **cluster IP**
* Using *kube-proxy* services can provide **pod load balancing**
* It is possible to **manually set** the cluster IP of a service

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

### Understanding service discovery

Service discovery allows to find resources in the k8s network, mainly using DNS. This exercise will show you the network scope of *pods* and *services*.

* Let's create a pair consistent in a `pod` and a `service`

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

* Deploy the manifest with the two document using

```bash
kubectl apply -f nginx.yaml
```

* Now define another `pod` from which we will contact the `nginx` one

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

* And deploy it

```bash
kubectl apply -f bash.yaml
```

## EKS pod IPs and domain IPs

* Take note of the ClusterIP assigned to the bash `pod`

```bash
BASH_POD_IP=$(kubectl get pods -n demo-$USER -l app=bash-app  -ojsonpath={.items[*].status.podIP})
echo $BASH_POD_IP
BASH_POD_DNS_NAME=${BASH_POD_IP//./-}
echo $BASH_POD_DNS_NAME

NGINX_POD_IP=$(kubectl get pods -n demo-$USER -l app=nginx-app  -ojsonpath={.items[*].status.podIP})
echo $NGINX_POD_IP
NGINX_POD_DNS_NAME=${NGINX_POD_IP//./-}
echo $NGINX_POD_DNS_NAME
```

## DNS resolution mechanisms

* See how the DNS configuration of the container has been established to automatically extend any DNS name to create a *fully qualified domain name* in the [resolv.conf](https://en.wikipedia.org/wiki/Resolv.conf#Contents_and_location) file

```bash
kubectl exec -it bash -- cat /etc/resolv.conf
```

* See how the DNS name of the `pod` is not automatically searched, so directly referring to it will fail

```bash
kubectl exec -it bash -- getent hosts $NGINX_POD_DNS_NAME
```

* Instead, use the whole *FQDN* to successfully resolve the IP

```bash
kubectl exec -it bash -- getent hosts $NGINX_POD_DNS_NAME.demo-$USER.pod.cluster.local
```

* On the other hand, `service` names are automatically searched by expanding its name, so

```bash
kubectl exec -it bash -- getent hosts nginx-service
```

* Is equivalent to

```bash
kubectl exec -it bash -- getent hosts nginx-service.demo-$USER.svc.cluster.local
```

* See how, by default, there is nothing preventing you from directly reach a `pod` from another one, although as both their DNS name and IPs should be considered ephemeral (in contraposition to `service` IP and DNS name)

```bash
kubectl exec -it bash -- wget -O- -q $NGINX_POD_DNS_NAME.demo-$USER.pod.cluster.local:80
```

## Clean up

* Remove everything

```
kubectl delete ns demo-$USER
```

## Appendix

* Even if there is no DNS server up and running in the cluster, it is possible to find the IP of any `service` because they are all listed as environment variables (yes, yes, we know)

```bash
kubectl exec -it bash -- env | grep SERVICE
```
