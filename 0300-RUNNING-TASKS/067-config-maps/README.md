# External configuration

## ConfigMaps

* Are key/value pairs
* Created as objects from templates or directly from property files
* Accessible from mounted files or env variables

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## External configuration basics

* Create the following resource descriptor

```yaml
cat << EOF > project-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: project-config
data:
  debug.asserts: enabled
  data.path: /data
  data.encoding: UTF8
  motd.txt: |
    "And, for an instant, she stared directly into those soft
    blue eyes and knew, with an instinctive mammalian certainty,
    that the exceedingly rich were no longer even remotely human."

    -- William Gibson
EOF
```

* Deploy the *ConfigMap*

```
kubectl apply -f project-config.yaml
```

* Check the result with

```
kubectl d▒▒▒▒▒▒▒ configmaps project-config
```

* Even if it is a bit long, take time to create and read the following file describing a deployment with the previous configuration loaded using two different methods (environment variables and files), and how to inject Kubernetes *metadata* into the pod (using `.valueFrom.fieldRef`)

```yaml
cat << 'EOF' > config-demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: config-demo-container
    image: bash
    env:
      - name: MY_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP        
      - name: DEBUG_ASSERTS
        valueFrom:
          configMapKeyRef:
            name: project-config
            key: debug.asserts
    command: ['sh', '-c', 'echo "Config demo started (Debug: $DEBUG_ASSERTS)" && sleep 600']
    volumeMounts:
    - name: configuration
      mountPath: "/configuration"
  volumes:
  - name: configuration
    configMap:
      name: project-config
EOF
```

* Launch the deployment

```bash
kubectl apply -f config-demo-pod.yaml
```

* Look at the bootstrapping message

```
kubectl logs config-demo
```

* Access the pod and check the configuration (**don't close the session after it**)

```
kubectl exec -it config-demo -- bash -c "env | grep MY_POD_IP"
kubectl exec -it config-demo -- bash -c "env | grep DEBUG"
kubectl exec -it config-demo -- ls /configuration
kubectl exec -it config-demo -- cat /configuration/motd.txt
kubectl exec -it config-demo -- cat /configuration/debug.asserts
```

* Now let's update the configuration

```bash
sed -i 's/debug.asserts: enabled/debug.asserts: disabled/' project-config.yaml
kubectl apply -f project-config.yaml
```

* See how, after a minute, the value of the fake file changes (press `ctrl+c` to exit)

```bash
kubectl exec -it config-demo -- watch cat /configuration/debug.asserts
```
* Notice, however, how the env var is still holding the old value (`env` variables aren't affected by a `ConfigMap` update until the pod is restarted)

```bash
kubectl exec -it config-demo -- bash -c "env | grep DEBUG"
```

## Challenge

Check this configuration file for `nginx` to forward all traffic to *google*

```nginx
cat << 'EOF' > nginx.conf
events {}

http {
  server {
      location / {
          proxy_pass http://www.google.com;
      }
  }
}
EOF
```

Knowing that the default path for the `nginx` configuration file is `/etc/nginx/nginx.conf`...

* Create a `ConfigMap` with the shown content 
* Create a `pod` manifest and mount the config map property at the required path, so it will become the configuration for `nginx`
* `Expose` or `port-forward` the `pod`
* Check if it is correctly acting as a proxy (you will retrieve the home page of *Google*)

<details>
<summary>Solution:</summary>

```
cat << 'EOF' > nginx.conf
events {}

http {
  server {
      location / {
          proxy_pass http://www.google.com;
      }
  }
}
EOF
```

```bash
kubectl create configmap nginx-config-map --f▒▒▒-▒▒▒▒=nginx.conf 
```

```yaml
cat << 'EOF' > nginx-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: nginx-config
          subPath: nginx.conf
          mountPath: /etc/nginx/nginx.conf
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config-map
EOF
```

```bash
kubectl apply -f nginx-demo.yaml 
kubectl logs nginx-demo
```

```bash
PORT=$(( ( RANDOM % 1000 )  + 8000 ))
echo $PORT
kubectl p▒▒▒-f▒▒▒▒▒▒ nginx-demo -n demo-$USER $PORT:80 --address='0.0.0.0' &
PID=$!
```

```bash
curl localhost:$PORT
```

```bash
kill -9 $PID
```
</details>


## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```

