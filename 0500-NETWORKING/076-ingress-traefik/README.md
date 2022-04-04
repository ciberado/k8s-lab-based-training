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
$ MY_TRAEFIK_NAMESPACE=traefik-studentXX 
$ MY_APP_NAMESPACE=pokemon-studentXX

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

## Middlewares

* Middlewares are attached to routers and basically are a means of tweaking the request before it's sent to your service or before the response from the service is sent back to the client.
* There are several available middleware in Traefik, some can modify the request, some the headers, some are in charge of redirections, some add authentication, and so on.
* Middlewares should be deployed on the application namespace to keep them isolated from others.
* In this section we will be configuring a middleware to modify request headers as well as response headers as an example.
* We will be using the Pokemon application to test it, for that make sure you have the `IngressRoute` requested above working and your application is accessible on port 80. You can find the solution below collapsed.



* To check the headers we can use `curl` or the inspect option directly on the browser itself so use whatever you prefer, you can use curl from windows too through the CMD.
* So before configuring any kind of middleware this is what we have.

```bash
$ curl -SsIv http://$ALB_HOSTNAME/
*   Trying 52.209.66.36...
* TCP_NODELAY set
* Connected to aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com (52.209.66.36) port 80 (#0)
> HEAD / HTTP/1.1
> Host: aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com
> User-Agent: curl/7.58.0
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Content-Length: 3106
Content-Length: 3106
< Content-Type: text/html; charset=utf-8
Content-Type: text/html; charset=utf-8
< Date: Fri, 04 Feb 2022 22:29:41 GMT
Date: Fri, 04 Feb 2022 22:29:41 GMT
< Etag: W/"c22-/Vl6JEJDlRfwurlKDTfteWZ+6mo"
Etag: W/"c22-/Vl6JEJDlRfwurlKDTfteWZ+6mo"
< X-Powered-By: Express
X-Powered-By: Express
```

* On the request side we have the typhical ones and on the response side we see the `Etag` used by the CDNs as well as the `X-Powered-By` header coming from the Node Js Express Framework. Now we will be changing this.
* The `Middleware` we are creating will have following definition, inside the spec you can see how we are adding the "Hello" header to the request as well as the "Accept" header and overwritting the current "X-Powered-By" because we don't like NodeJs.

```yaml
cat <<EOF > pokemon-middleware.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rewrite-headers
  namespace: $MY_APP_NAMESPACE
spec:
  headers:
    custom▒▒▒▒▒▒▒Headers:
      Hello : "World!"
      Accept : "application/json"
    customResponseHeaders:
      X-Powered-By: "Django"
EOF
```

* Let us apply it!

```bash
$ kubectl apply -f pokemon-middleware.yaml
```

* Now that it is created we must tell the router inside the `IngressRoute` we created for the application to use it before delivering the request to the service as well as back to the client.
* Update the `pokemon-ingressroute.yaml`

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
    ▒▒▒▒▒▒▒▒▒▒▒: # what goes here?
      - name: rewrite-headers 
        namespace: $MY_APP_NAMESPACE
    match: Host(\`$ALB_HOSTNAME\`) && PathPrefix(\`/\`)
    services:
    - name: pokemon-nodejs
      port: 80
      namespace: $MY_APP_NAMESPACE
EOF
```

* **Javi Moreno's** application is coded to render the web page with the pokemon you got, but if the `application/json` is provided on the request headers the application understands that and will not render the web page but to just render the output in json format.
* Apply the changes and see what happens. 

```bash
$ kubectl apply -f pokemon-ingressroute.yaml
```

* You can see that now the same `curl` command is showing the "X-Powered-By" header modified in the response but we don't see any change on the request headers. This is because the middleware will change the request headers once it reaches the Traefik and the router delivers it to the middleware.

```bash
$ curl -SsIv http://$ALB_HOSTNAME/
*   Trying 52.209.74.109...
* TCP_NODELAY set
* Connected to aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com (52.209.74.109) port 80 (#0)
> HEAD / HTTP/1.1
> Host: aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com
> User-Agent: curl/7.58.0
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Content-Type: application/json
Content-Type: application/json
< Date: Mon, 14 Feb 2022 09:45:50 GMT
Date: Mon, 14 Feb 2022 09:45:50 GMT
< X-Powered-By: Django
X-Powered-By: Django
```

* I know you don't belive in anything without proper evidence as good engineers, so running `curl` again without any option this time though, we can see that we are getting the json output not the HTML, the only way to get the json output without the middleware in place is by passing the headers directly with curl but in this case we are not providing any, it's the middleware indeed.

```bash
$ curl http://$ALB_HOSTNAME/ ; echo
{"pokemon":{"id":"478","name":"Froslass"},"hostname":"pokemon-nodejs-7875df558c-hrc7j","friends":[]}
```

* Not enough? We have added a console logging of the headers in Json to the application code so we are able to see the request headers the application receive. For that let's stream the application logs and refresh our browser or run a curl again and see.

```bash
$ $ kubectl -n $MY_APP_NAMESPACE logs -f --selector=app.kubernetes.io/name=pokemon-nodejs 
```
* You will be seeing something like this.
```
Rendering json
{
  host: 'aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com',
  'user-agent': 'curl/7.58.0',
  accept: 'application/json',
  hello: 'World!',
  'x-forwarded-for': '192.168.14.34',
  'x-forwarded-host': 'aa4ef1d2709e640f59ad5a1ef2649819-1690555703.eu-west-1.elb.amazonaws.com',
  'x-forwarded-port': '80',
  'x-forwarded-proto': 'http',
  'x-forwarded-server': 'traefik-6bf654c4c6-v66l5',
  'x-real-ip': '192.168.14.34'
}

```

## Cleanup

* Delete all the created resources.

```
$ kubectl delete -f resources/pokemon-nodejs.yaml
$ kubectl delete -k .
```
