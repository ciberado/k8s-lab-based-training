# Health checks


## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Key concepts

* **startupProbe** helps to wait until the app is up and running
* **readynessProbe** is used by services to send traffic to the pod
* **livenessProbe** is used by Kubernetes to restart the pod in case of failure
* A pod that fails gracefully when it is not able to achieve its purpose does not necessary need a *liveness probe*: it will be rescheduled if `restartPolicy` is set to `Always` or `OnFailure`
* `restartPolicy` is applied to all the containers in the pod

## Pod configuration

* Use this manifest as the base for defining your own:

```yaml
cat << 'EOF' > pokemon.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pokemon
spec:
  terminationGracePeriodSeconds: 60 # <-- extra time if livenessProbe is ok
  containers:
  - name: pokemon-container
    image: ciberado/pokemon-nodejs:v1.0.3
    ports:
    - name: server-port         # <-- use named ports for clarity 
      containerPort: 80
EOF
```

* Deploy the application and see how quickly the pod becomes `ready` because there is no probe to check (press `ctrl+c` to exit the watch)

```bash
kubectl apply -f pokemon.yaml && kubectl get pods --watch
```

* Once the behaviour is clear, delete the `pod` with

```bash
kubectl delete -f pokemon.yaml
```

## Probes

* Knowing that the application has a `/health` endpoint and it is a `node` application, let's add the following probes to the manifest: `spec.containers[].startupProbe`, `spec.containers[].readinessProbe` and `spec.containers[].livenessProbe`.

```yaml
cat << 'EOF' > pokemon.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pokemon
spec:
  terminationGracePeriodSeconds: 60 # <-- extra time if livenessProbe is ok
  containers:
  - name: pokemon-container
    image: ciberado/pokemon-nodejs:v1.0.3
    ports:
    - name: server-port         # <-- use named ports for clarity 
      containerPort: 80
    startupProbe:               # <-- Wait before starting other probes
      httpGet:
        port: server-port
        path: "/health"
      failureThreshold: 5
      periodSeconds: 15         # <-- wait up to 5*15 seconds
    readinessProbe:             # <-- send me traffic if it is ok
      httpGet:
        port: server-port
        path: "/health"
      initialDelaySeconds: 0    # <-- Not needed, already startuped
      periodSeconds: 20         # <-- Check three times per minute
      timeoutSeconds: 5         # <-- Fail if it takes more than this
      successThreshold: 1       # <-- First green means it's ok
      failureThreshold: 3       # <-- Two consecutive ko stops traffic to the pod
    livenessProbe:
      exec:
        command: ["pgrep", "-x", "node"]  # <-- This is a js application for node
      initialDelaySeconds: 1
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 2
      terminationGracePeriodSeconds: 10  # <-- Quick kill if liveness probe is not ok
EOF
```

* Check how the container is not ready until it passes the seconds specified by `spec.containers[].startupProbe` (type `ctrl+c` to exit the `--watch` command)

```
kubectl apply -f pokemon.yaml && kubectl get pod --watch
```


* Forward the service port to access the application:

```bash
PORT=$(( ( RANDOM % 100 )  + 8000 ))
kubectl port-forward pokemon --address 0.0.0.0 -n demo-$USER $PORT:80 &
PID=$!
echo The tunnel PID is $PID and the endpoint address is http://localhost:$PORT
```

* Check the existing health endpoint

```bash
curl localhost:$PORT/health; echo
```

<details>
<summary>
Use `kubectl exec` against the `pokemon` pod to check if the main process (`node`) is being ran (see [pgrep](https://dashdash.io/1/pgrep)).
</summary>

```bash
kubectl exec -it pokemon -- pgrep node
```
</details>


## Unexpected fails

* Set the application to *unhealthy* state 

```bash
curl localhost:$PORT/health && echo
curl -X DELETE http://localhost:$PORT
curl localhost:$PORT/health && echo
```

* Look for the *readiness* of the pods for some minutes to see it evolving, until it reaches the *unready* state.

```bash
kubectl get pods -owide --watch
```

* Troubleshot the pod (check the events to find the unhealthy message)

```bash
kubectl describe pod pokemon
```

## Cleanup


* Stop the tunnel

```
kill -9 $PID
```

* Delete the resources

```bash
kubectl delete ns demo-$USER
```
