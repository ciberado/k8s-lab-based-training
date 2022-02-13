# Traefik Ingress Controller

## Introduction

* Traefik is a reverse proxy (edge router) that intercepts and routes every incoming request based on rules, easy right?
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
* We will be seeing and playing with some of the key components of Traefik like the EntryPoints, the Routers, or the Middlewares.

## Traefik Dataflow Diagram

![Traefik Architecture](https://doc.traefik.io/traefik/assets/img/architecture-overview.png)

## Preparation

* Clone the lab repository locally and `cd` to the lab directory

```bash
$ git clone git@github.com:ciberado/ntt-k8s-censored-training.git

$ cd ntt-k8s-censored-training/0500-NETWORKING/076-ingress-traefik
```

* Install `kustomize` package, we will be doing it from binaries as the lab environment has no access to NTT but keep in mind that we have this tool packaged and shipped to be used ^^.

```bash
# Kustomize NTT packaged version
$ apt install ntt-kustomize

# From official binaries
$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

* Set up the env variable MY_TRAEFIK_NAMESPACE with a unique name for your traefik namespace and then run the following command to change all the placeholders on the yaml files.

```bash
$ MY_TRAEFIK_NAMESPACE=traefik-yunes

$ find . -type f \( -name '*.yaml' \) | xargs sed -i "s/_NAMESPACE_PLACEHOLDER_/$MY_TRAEFIK_NAMESPACE/g"
```

* Open the `kustomization.yaml` file you should see something like this

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: traefik-yunes

resources:
  - resources/crds.yaml
  - resources/install.yaml
```

## Deploying

* There are two ways of deploying the manifests with kustomize.
* First one, the recommended one, is using the `kustomize` tool and pipe it to `kubectl`.
* The second one, not recommended for production, but for this lab is also an option if you were not able to install kustomize correctly, is using directly `kubectl` with the `-k` flag (kustomize module), this method is not recommended because currently `kubectl` has an older version of kustomize integrated.

```bash
# Recommended way
$ kustomize build . | kubectl apply -f -

# Not recommended way but also possible specially if you were not able to install kustomize
$ kubectl apply -k .
```

* Traefik has CRDs (Custom Resource Definitions) so the first time we deploy it, we may see some errors, just wait for the Traefik pods to be ready and running and run the deploy command again.
* Check the status of your namespace, you should see something similar to this, two pods ready and running, a service with an external ALB and a deployment, replicaset and hpa.

```bash
$ kubectl -n traefik-yunes get all
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
* The `IngressRoutes` define the listening entry point and to which service is forwarded based on a set of rules, also defines how the request should be processed in between, like changing the scheme from http to https, rewriting the host or the headers and so.
* The first `IngressRoute` to create will be the one to expose the Traefik UI, open the file `resources/ingressroutes.yaml` and check if the hostname inside the Rule section has something custom like your namespace name.
* In the spec you will see two clear sections, the entrypoint that refers to where the request should come from and the routes where the request should go based on rules. In our case the rule will send to the Traefik service `TraefikService` (CRD too) any request coming from the `traefik` entrypoint based on the hostname `traefik-yunes-test.local`.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-api
  namespace: traefik-yunes
spec:
  entryPoints:
  - traefik
  routes:
  - kind: Rule
    match: Host(`traefik-yunes-test.local`)
    services:
    - kind: TraefikService
      name: api@internal
    # Above definition is equal to below definition
    # - name: traefik-external
    #   port: 8080
    #   namespace: traefik-yunes
```

* So now, let us deploy it.

```bash
$ kubectl apply -f resources/ingressroutes.yaml
```

* Now you should see that an ingressroute has just popped up on your namespace, To check it just get the ingressroutes in your namespace.

```bash
$ kubectl -n traefik-yunes get ingressroutes
NAME          AGE
traefik-api   13s
```

* To access the Traefik web UI we would need to configure the hostname you configured on the ingressroute to resolve the IP behind the Traefik service, as we can't have that right now, we will simulate it by setting it up on our local `/etc/hosts`, don't do this at work, use Route53!
* Get the Traefik service external IP, you will see that this external IP is an ALB hostname, just resolve it to get one of its public IPs.

```
$ kubectl -n traefik-yunes get svc
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
traefik-external   LoadBalancer   10.100.134.225   a8bf0f20bebc24bfe9fdfe19dd4860f9-1788432504.eu-west-1.elb.amazonaws.com   80:30001/TCP   3h39m

$ nslookup a8bf0f20bebc24bfe9fdfe19dd4860f9-1788432504.eu-west-1.elb.amazonaws.com
Server:                172.23.80.1
Address:        172.23.80.1#53

Non-authoritative answer:
Name:   a8bf0f20bebc24bfe9fdfe19dd4860f9-1788432504.eu-west-1.elb.amazonaws.com
Address: 34.241.165.50
Name:   a8bf0f20bebc24bfe9fdfe19dd4860f9-1788432504.eu-west-1.elb.amazonaws.com
Address: 52.48.167.165
Name:   a8bf0f20bebc24bfe9fdfe19dd4860f9-1788432504.eu-west-1.elb.amazonaws.com
Address: 52.209.66.36
```

* Open your `/etc/hosts` and add an entry like `34.241.165.50     traefik-yunes-test.local`. On Windows the hosts file can be found in `C:\Windows\System32\drivers\etc\hosts` on Linux you already know (vim /etc/hosts) and for macOS I don't care, just joking, same as Linux ^^.
* Browse your custom domain like `http://traefik-yunes-test.local/` in your preferred browser and see if it's working. Is it? I guess not right? Lets troubleshoot this!
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

* Also, each "containerPort" must be exposed by a Service. If you check the Service spec for the Traefik you will see that the type is *LoadBalancer*, so behind the scene the cluster will provision an external ALB to be used.
* For Traefik we just need one Service type LoadBalancer for all the ingresses we need to open, this reduces the cost as you will not be having one ALB per Ingress as we used to have with the native Ingress object.
* So, which port is it being used for the Service to expose the traefik entrypoint? 

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

* The `8080` right? So test now the same URL on the browser but changing the port like `http://traefik-yunes-test.local:8080/`. Is it working now?

* Now, I have placed a file `resources/pokemon-nodejs.yaml` containing all the resources needed to deploy **Ciberado's** Pokemon NodeJS application, your task here is to create the `IngressRoute` needed to expose it on HTTP port 80. May the force be with you.
* To apply the resources just proceed with kubectl no need to use kustomize, **important** thing, we will be using a different namespace than the one for the Traefik to show that each application can be isolated, Traefik is needed just once.
* Once you have the IngressRoute, place it at the bottom of the file as a new resource and apply it all together. Have fun!

```bash
$ MY_APP_NAMESPACE=pokemon-nodejs-yunes

$ find ./resources -type f \( -name '*.yaml' \) | xargs sed -i "s/_APP_NAMESPACE_PLACEHOLDER_/$MY_APP_NAMESPACE/g"

$ kubectl apply -f resources/pokemon-nodejs.yaml
```

## Middlewares

* Middlewares are attached to routers and basically are a means of tweaking the requests before they are sent to your service or before the answer from the services are sent back to the clients.
* There are several available middleware in Traefik, some can modify the request, the headers, some are in charge of redirections, some add authentication, and so on.
* Middlewares should be deployed on the application namespace to keep them isolated from others.
* In this section we will be configuring a middleware to modify request headers as well as response headers as an example.
* We will be using the Pokemon application to test it, for that make sure you have the `IngressRoute` requested above working and your application is accessible. You can find the solution below collapsed.

<details><summary>Solution</summary>
<p>

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: pokemon-nodejs-ingress
  namespace: pokemon-nodejs-yunes
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    match: Host(`pokemon-nodejs-yunes-test.local`)
    services:
    - name: pokemon-nodejs
      port: 80
      namespace: pokemon-nodejs-yunes
```
</p>
</details>

* To check the headers we can use `curl` or the inspect option directly on the browser itself so use whatever you prefer, you can use curl from windows too through the CMD.
* So before configuring any kind of middleware this is what we have. You can also try to access through the browser to actually see your Pokemon!

```bash
$ curl -SsIv http://pokemon-nodejs-yunes-test.local/
*   Trying 52.209.66.36...
* TCP_NODELAY set
* Connected to pokemon-nodejs-yunes-test.local (52.209.66.36) port 80 (#0)
> HEAD / HTTP/1.1
> Host: pokemon-nodejs-yunes-test.local
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

* So, on the request side we have the typhical ones and on the response side we see the `Etag` used by the CDNs as well as the `X-Powered-By` header coming from the Node Js Express Framework. Now we will be changing this.
* Then, the `resources/middleware.yaml` has the following definition, inside the spec you can see how we are adding the "Hello" header to the request as well as the "Accept" header and overwritting the current "X-Powered-By" because we don't like NodeJs.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rewrite-headers
  namespace: pokemon-nodejs-yunes
spec:
  headers:
    customRequestHeaders:
      Hello : "World!"
      Accept : "application/json"
    customResponseHeaders:
      X-Powered-By: "Django"
```

* Let us apply it!

```bash
$ kubectl apply -f resources/middleware.yaml
```

* Now that it is created we must tell the router inside the `IngressRoute` we created for the application to use it before delivering the request to the service as well as back to the client.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: pokemon-nodejs-ingress
  namespace: pokemon-nodejs-yunes
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    middlewares:
      - name: rewrite-headers
        namespace: pokemon-nodejs-yunes
    match: Host(`pokemon-nodejs-yunes-test.local`)
    services:
    - name: pokemon-nodejs
      port: 80
      namespace: pokemon-nodejs-yunes
```

* **Javi Moreno's** application is coded to render the web page with the pokemon you got, but if the `application/json` is provided on the request headers the application understands that and will not render the web page but to just render the output in json format.
* Apply the changes and see what happens. You can see that now the same `curl` command is showing the "X-Powered-By" header modified in the response but we don't see any change on the request headers. This is because the middleware will change the request headers once it reaches the Traefik and the router delivers it to the middleware.

```bash
$ curl -SsIv http://pokemon-nodejs-yunes-test.local/*   Trying 52.209.66.36...
* TCP_NODELAY set
* Connected to pokemon-nodejs-yunes-test.local (52.209.66.36) port 80 (#0)
> HEAD / HTTP/1.1
> Host: pokemon-nodejs-yunes-test.local
> User-Agent: curl/7.58.0
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Content-Type: application/json
Content-Type: application/json
< Date: Sat, 05 Feb 2022 01:42:33 GMT
Date: Sat, 05 Feb 2022 01:42:33 GMT
< X-Powered-By: Django
X-Powered-By: Django
```

* I know you don't belive in anything without proper evidence as good engineers, so running `curl` again without any option this time though, we can see that we are getting the json output not the HTML, the only way to get the json output without the middleware in place is by passing the headers directly with curl but in this case we are not providing any, it's the middleware indeed.

```bash
$ curl http://pokemon-nodejs-yunes-test.local/
{"pokemon":{"id":"305","name":"Lairon"},"hostname":"pokemon-nodejs-7875df558c-7lhj4","friends":[]}
```

* Not enough? We have added a console logging of the headers in Json to the application code so we are able to see the request headers the application receive. For that let's stream the application logs and refresh our browser or run a curl again and see.

```bash
$ kubectl -n pokemon-nodejs-yunes logs -f pokemon-nodejs-7875df558c-7lhj4
```
* You will be seeing something like this.
```
Rendering json
{
  host: 'pokemon-nodejs-yunes-test.local',
  'user-agent': 'curl/7.58.0',
  accept: 'application/json',
  hello: 'World!',
  'x-forwarded-for': '192.168.14.34',
  'x-forwarded-host': 'pokemon-nodejs-yunes-test.local',
  'x-forwarded-port': '80',
  'x-forwarded-proto': 'http',
  'x-forwarded-server': 'traefik-6bf654c4c6-v66l5',
  'x-real-ip': '192.168.14.34'
}

```

## Cleanup

* Delete all the created resources.

```
$ kubectl delete -f resources/ingressroutes.yaml
$ kubectl delete -f resources/middleware.yaml
$ kubectl delete -f resources/pokemon-nodejs.yaml
$ kubectl delete -k .
```
