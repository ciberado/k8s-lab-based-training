# Pod lifecycle


This laboratory uses not only `pods`, but also `jobs` and `deployments`. You don't need to fully understand them, but make sure you come back once you are familiar with that kind of resources.

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Pod phases (status field)

* `Pending`: being scheduled or downloading image
* `Running`: at least one container is up
* `Suceeded`: `restartPolicy` was set to `never` or `onFailure`and all the containers have finished with a 0 code
* `Failed`: `restartPolicy` was set to `never` and at least one container existed with an error
* `Completed`: the pod was part of a *job* and it was successfully completed
* `CrashLoopBackOff`: repeated failure of at least one container is holding back pod recreation
* `Unknown`: something happened, smile

## Shutdown sequence

* The `pod` is set in `Terminating` state
* `preStop` hook execute
* `SIGTERM` sent to the process with pid `1`
* Kubernetes waits `spec.terminationGracePeriodSeconds` (30s by default)
* `SIGKILL` sent to proces with pid `1`

## Pod resiliency

* The pod `restartPolicy` is only applied at node level
* To survive node failure a controller (`job`, `resplicaset`, `deployment`...) should be used

## Playing with pod status

* Run this poor pod and check how it goes from *ContainerCreating* to *Running* to *Error*

```bash
kubectl run run-once-and-fail \
  --image bash \
  --restart=Never \
  -- bash -c "exit 1"

kubectl get pod run-once-and-fail -owide
```

* Check the resulting status of the pod (`Error`)

```bash
kubectl describe pod run-once-and-fail
```

<details>
<summary>
How can you know if the pod has been finished with a `Succeed` or a `Failed` exit?
</summary>

```bash
kubectl describe pod run-once-and-fail | grep Status
kubectl get pod run-once-and-fail -o jsonpath="{.status.containerStatuses[].state}" | jq
```
</details>

* Now, create a `deployment` and see how it doesn't matter how many times the pod dies it will be scheduled again

```bash
kubectl run restart-always \
  --image bash \
  --restart Always \
  -- bash -c "exit 1"

kubectl get pod -l run=restart-always -owide --watch
```

* Remove the namespace

```bash
kubectl delete ns demo-$USER
```


## Extraball

<details>
<summary>
How can you transform an imperative command to a declarative manifest using `--dry-run`?
</summary>

```bash
kubectl run run-once-and-fail \
  --image=busybox \
  --restart=Never \
  -n demo-$USER \
  --dry-run=True \
  -oyaml \
  -- sh -c "exit 1" 
```
</details>
