# The dangers of legacy applications

## Legacy programs and bad behavior

* Not all programs are aware of the limits enforced by c-groups
* A program trying to allocate more memory than allowed will be terminated
* Check [java-docker-cgroups/entrypoint.sh](https://github.com/ciberado/java-docker-cgroups/blob/master/entrypoint.sh) to see how a relatively old version of the JVM can be started with or without `c-groups` options

```bash
#!/bin/sh
test "$1" = "-x" && {
	echo "Enable experimental vm options"
	export JAVA_OPTS="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=1 -XX:+UseG1GC"
}
java $JAVA_OPTS Main
```

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Legacy applications unaware of the limits

* Define a `pod` with a container that will try to allocate trillions of zetabytes of memory, using a JVM that is not aware of the limits imposed by the `pod` spec.

```bash
cat << EOF > java-old-version.yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-old-version
spec:
  containers:
  - name: java1
    image: ciberado/java-docker-cgroups
    args: []
    resources:
      limits:
        memory: "100Mi"
      requests:
        memory: "100Mi"
EOF
```

* Run it 

```bash
kubectl apply -f java-old-version.yaml
```

* Check the logs to see how the container runtime killed the poor `pod`

```
kubectl logs java-old-version
```

## Legacy applications unaware of the limits

* Now define a pod with the proper configuration for the JVM, making the runtime aware of the real limits of the process

```bash
cat << EOF > java-new-version.yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-new-version
spec:
  containers:
  - name: java1
    image: ciberado/java-docker-cgroups
    args: ['-x']
    resources:
      limits:
        memory: "100Mi"
      requests:
        memory: "100Mi"
EOF
```

* Run the `pod`

```bash
kubectl apply -f java-new-version.yaml
```

* Check the logs and see how in this case the JVM reacted to the application generation an error (instead of just being killed)

```
kubectl logs java-new-version
```


## Extra ball

* In an small cluster it is possible to get information about the node utilization using bash magic

```
TIME=10; while true; do kubectl get nodes --no-headers \
 | awk '{print $1}' \
 | xargs -I {} sh -c 'echo {}; kubectl describe node {} \
   | grep Allocated -A 5 \
   | grep -ve Event -ve Allocated -ve percent -ve -- ; echo'; sleep $TIME; done
```

## Cleanup

* Delete all the resources

```bash
kubectl delete ns demo-$USER
```
