# Traefik Ingress Controller

## Introduction

* Traefik is a reverse proxy (edge router) that intercepts and routes every incoming request based on rules.
* Dynamic, creates the routes for you based on autodiscovery.
* Fast, written in Go.
* Open-Source.
* Production proven.
* Circuit breakers, retries.
* Observavility can be easily configured for tracing and metrics.
* Can run everywhere, literally, as it's shipped as a docker image.
* We have a cool gopher logo as well.
* Check the official [Traefik](https://github.com/traefik/traefik/tree/master) documentation for more.

## Getting started

* Traefik can be configured with dynamic or static configuration, in our use case we will be going for static to make the lab less complex.
* Traefik can be installed using helm charts, we will be installing a helm chart version through kustomize to simplify the process.
* We will play with some of the key components of Traefik like the EntryPoints, the Routers, or the Middlewares.

## Traefik Dataflow Diagram

![Traefik Architecture](https://doc.traefik.io/traefik/assets/img/architecture-overview.png)

## Preparation

* Clone the lab repository locally.

```bash
$ git clone https://github.com/ciberado/k8s-censored-training.git

$ cd ntt-k8s-censored-training/0500-NETWORKING/076-ingress-traefik
```

* Set up the env variable MY_TRAEFIK_NAMESPACE and MY_APP_NAMESPACE with your desk number

```bash
# Change the XX for your desk number
$ MY_TRAEFIK_NAMESPACE=traefik-$USER 
$ MY_APP_NAMESPACE=pokemon-$USER

$ find . -type f \( -name '*.yaml' \) | xargs sed -i "s/_TRAEFIK_NAMESPACE_PLACEHOLDER_/$MY_TRAEFIK_NAMESPACE/g"
$ find ./resources -type f \( -name '*.yaml' \) | xargs sed -i "s/_APP_NAMESPACE_PLACEHOLDER_/$MY_APP_NAMESPACE/g"
```

## Deploying

* We are using kustomize to do the initial bootstraping of traefik.

```bash
$ kubectl apply -k .
```

* Traefik has CRDs (Custom Resource Definitions) so we may see some errors the first time we deploy it, just wait for the Traefik pods to be ready and run the deploy command again.
* Check the status of your namespace, you should see something similar to this, two pods ready and running, a service with an external ALB, a deployment, replicaset and hpa.

```bash
$ kubectl -n $MY_TRAEFIK_NAMESPACE get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-6bf654c4c6-tws9v   1/1     Running   0          112m
pod/traefik-6bf654c4c6-vt7j4   1/1     Running   0          112m

NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                     AGE
service/traefik-external   LoadBalancer   10.100.187.235   a2f8cbe5cd84a46a1908e7f28ed7afa3-1063470380.eu-west-1.elb.amazonaws.com   80:32570/TCP,443:30569/TCP,8080:31107/TCP   112m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   2/2     2            2           112m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-6bf654c4c6   2         2         2       112m

NAME                                          REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/traefik   Deployment/traefik   0%/75%    2         4         2          112m
```

## Entrypoints and Routes

* To be able to expose your service applications you must create a resource called `IngressRoute`. This is a custom definition that Traefik brings, similar to the native Kubernetes `Ingress` but with different spec and features, the concept is the same though.
* The `IngressRoute` is the CRD implementation of an HTTP routing, there are also other kinds for TCP and UDP routing.
* An `IngressRoute` defines the listening entry point and to which service is forwarded based on a set of rules, also defines how the request should be processed in between, like changing the scheme from http to https, rewriting the host or the headers and so.
* The first `IngressRoute` to create will be the one to expose the Traefik UI.
* In the following spec you will see two clear sections, the entrypoint where traefik listens for the request and the routes where the request should be forwarded based on rules. 

```bash
ALB_HOSTNAME=$(kubectl -n $MY_TRAEFIK_NAMESPACE get svc traefik-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

```yaml
cat <<EOF > traefik-ingressroute.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-api
  namespace: $MY_TRAEFIK_NAMESPACE
spec:
  entry▒▒▒▒▒▒:
  - traefik
  routes:
  - kind: Rule
    match: Host(\`$ALB_HOSTNAME\`) && PathPrefix(\`/\`)
    services:
    - kind: TraefikService
      name: api@internal
EOF
```

* Deploy it.

```bash
$ kubectl apply -f traefik-ingressroute.yaml
```

* Now you should see that an ingressroute has just popped up on your namespace, To check it just get the ingressroutes in your namespace.

```bash
$ kubectl -n $MY_TRAEFIK_NAMESPACE get ingressroutes
NAME          AGE
traefik-api   13s
```

* To access the Traefik web UI through our web browser we need to get the external service hostname.

```bash
$ kubectl -n $MY_TRAEFIK_NAMESPACE get svc traefik-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'; echo
```

* Browse your external IP hostname and see if it's working. Is it?


* Open the `resources/install.yaml` and scroll down to the ConfigMap and see how many entrypoints do we have. 

```yaml
  entryPoints:
    web:
      address: ":8000"
    websecure:
      address: ":8443"
    metrics:
      address: ":8082"
    traefik:
      address: ":9000"
```

* We have four "entryPoints" and each of them has an address port that will be the same "containerPort" on the Deployment object. 

```yaml
  ports:
    - containerPort: 9000
      name: traefik
      protocol: TCP
    - containerPort: 8000
      name: web
      protocol: TCP
    - containerPort: 8443
      name: websecure
      protocol: TCP
    - containerPort: 8082
      name: metrics
      protocol: TCP
```

* Each port on the container must be exposed by a Service. Which port is it being used for the Service to expose the traefik entrypoint? 

```yaml
spec:
  ports:
  - name: web
    port: 80
    protocol: TCP
    targetPort: web
  - name: websecure
    port: 443
    protocol: TCP
    targetPort: websecure
  - name: traefik
    port: 8080
    protocol: TCP
    targetPort: traefik
```

* Which entrypoint did we use on the previous `IngressRoute` created for the Traefik UI? Test the same hostname on the browser but change the port for the correct one and see.

* Now, I have placed a file `resources/pokemon-nodejs.yaml` containing all the resources needed to deploy **Ciberado's** Pokemon NodeJS application, your task here is to create the `IngressRoute` needed to expose it on HTTP port **80**. May the force be with you.

```bash
$ kubectl apply -f resources/pokemon-nodejs.yaml
```

<details><summary>Solution don't check until you try</summary>
<p>

```yaml
cat <<EOF > pokemon-ingressroute.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: pokemon-nodejs-ingress
  namespace: $MY_APP_NAMESPACE
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    match: Host(\`$ALB_HOSTNAME\`) && PathPrefix(\`/\`)
    services:
    - name: pokemon-nodejs
      port: 80
      namespace: $MY_APP_NAMESPACE
EOF
```
</p>
</details>

```bash
$ kubectl apply -f pokemon-ingressroute.yaml
```

## Cleanup

* Delete all the created resources.

```
$ kubectl delete -f resources/pokemon-nodejs.yaml
$ kubectl delete -k .
```
