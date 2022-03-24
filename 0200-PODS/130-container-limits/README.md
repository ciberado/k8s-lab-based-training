# Limits and monitoring

## Resource limits explanation

* Both memory and cpu usage can be limited
* From v1.8 ephemeral storage performance is also considered a compute resource
* Custom compute resources can be created using [extended resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#cluster-level-extended-resources)
* The restrictions are per container: the pod limits are the sum of all its containers
* CPU usage is measured in *milicores*: 1000m is equivalent to a whole cpu
* In most situation it is better to run two replicas with one core than one pod with two cores
* A container will not be able to use more cycles per second that those provided by its quota: if it happens Kubernetes will throttle it
* A container trying to allocate more ram that specified in the quota might be restarted
* If there are no nodes with enough resources the pod will not be scheduled
* The **requests** is the minimum amount of resources the container needs
* The **limit** is the maximum amount of the resource provided by the node and can be throttled if needed


## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Request and Limit


* Take the next manifest as the base for learning about requests and limits

```yaml
cat << EOF > stress.yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-demo
spec:
  containers:
  - name: stress-demo-container
    image: containerstack/alpine-stress
    command: ['sh', '-c', 'echo I can be stressed! && sleep 36000']
EOF
```

## Applying resource constraints

How would you modify the manifest to specify resource limits? Our application usually
needs 100MB of RAM, with spikes up to 200MB.

```yaml
cat << EOF > stress.yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-demo
spec:
  containers:
  - name: stress-demo-container
    image: containerstack/alpine-stress
    command: ['sh', '-c', 'echo I can be stressed! && sleep 36000']
    resources:
      ▒▒▒▒▒▒:
        ▒▒▒▒▒▒: "200Mi"
      ▒▒▒▒▒▒▒▒:
        ▒▒▒▒▒▒: "100Mi"
EOF
```

* Deploy de demo

```
kubectl apply -f stress.yaml
```

* Connect to the container and allocate 100Mi bytes for 60 seconds and 180 for another minute

```bash
kubectl exec -it stress-demo -- stress --vm 1 --vm-bytes 200M --vm-hang 60 -t 60 -v
kubectl exec -it stress-demo -- stress --vm 1 --vm-bytes 120M --vm-hang 60 -t 60 -v
```

## Discussion

Can you please explain the difference in the behavior of both commands?

<details>
<summary>
Solution:
</summary>

▒▒▒ ▒▒▒▒▒ ▒▒▒ ▒▒▒▒ ▒▒ ▒▒▒▒▒▒▒▒, ▒▒ `▒▒▒▒▒▒` ▒▒▒▒▒▒▒▒▒▒▒ ▒▒▒▒ ▒▒ ▒▒▒▒ ▒▒▒ ▒▒▒▒ ▒▒▒▒▒▒ ▒▒▒▒
▒▒▒▒▒▒ ▒▒ ▒▒ ▒▒▒▒▒▒▒▒▒ (▒▒ ▒▒▒▒▒▒▒ ▒▒▒ ▒▒▒▒▒▒ ▒▒ ▒▒▒ ▒▒▒▒▒▒▒▒▒ `▒-▒▒▒▒▒`). ▒▒ ▒▒ ▒▒▒▒▒▒▒▒▒
▒▒ ▒▒▒ ▒▒▒ ▒▒▒ ▒▒ ▒▒ ▒▒▒▒▒▒ ▒▒▒▒ ▒▒▒▒▒▒, ▒▒ ▒▒▒▒▒ ▒▒▒'▒ ▒▒▒▒▒ ▒▒▒▒▒▒▒▒▒ ▒▒▒▒▒▒▒.

</details>

# Legacy programs and bad behavior

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

* Run a pod without those options and take note of the output

```bash
kubectl run j1 --image=ciberado/java-docker-cgroups --limits=memory=100Mi -it --restart=Never
```

* Run the same image again but including the `-x` option to activate the c-groups awareness and compare both outputs (Killed vs `java.lang.OutOfMemoryError`)

```bash
kubectl run j2 --image=ciberado/java-docker-cgroups --limits=memory=100Mi -it --restart=Never -- -x
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
