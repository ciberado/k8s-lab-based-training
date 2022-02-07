# Jobs

`Jobs` provide a simple but resilient way to execute a single or recurrent task. They launch a pod with a particular.

Keeping finishing jobs in the cluster can become problematic. It is possible to use `.spec.ttlSecondsAfterFinished` to automatically delete both the pods and the job itself after the required number of seconds.

[CronJobs](../064-cronjob/README.md) are very similar, but provide an additional field (`.spec.schedule`) to run the same pod at intervals.

Both of them should be **idempotent*, as it is not guaranteed the uniqueness of their execution. Also, **executions are not guaranteed**.

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Single shot tasks

We are going to use an external database ([keyvalue])(https://keyvalue.immanuel.co) to easily change the result of executing a pod. Basically, we will store the exit code that the main container will use, making it initially falling with an `exit 1` to see how the job is restarted and the modifying the value to `0` to allow the container to have a clean finish.

* Obtain an authorization key for `keyvalue`

```bash
AUTHKEY=$(curl https://keyvalue.immanuel.co/api/KeyVal/GetAppKey --silent); echo "Auth key: ${AUTHKEY:1:8}"
```

* Create the record `job-$USER` with the value `1`

```bash
curl -X POST -d "" "https://keyvalue.immanuel.co/api/KeyVal/UpdateValue/${AUTHKEY:1:8}/job-$USER/1"; echo
```

* Check the value stored in the external database (it should return `1`)

```bash
curl https://keyvalue.immanuel.co/api/KeyVal/GetValue/${AUTHKEY:1:8}/job-$USER; echo
```

* Create the job descriptor

```yaml
cat << EOF > job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo-job
spec:
  ttlSecondsAfterFinished: 180 # seconds
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
      - image: bash
        name: bashtask
        command: ["/usr/local/bin/bash"]
        args: ["-c", "apk add curl; CODE=\$(curl -s https://keyvalue.immanuel.co/api/KeyVal/GetValue/${AUTHKEY:1:8}/job-$USER); exit \${CODE:1:1}"]
EOF
```

## Failed execution

* Execute the job

```bash
kubectl apply -f job.yaml
```

* Get the resources created by the manifest (you should see both the `job` and the `pod`)

```bash
kubectl get all
```

* Watch what is happening with the pods (press `ctrl+C` to get back)

```bash
kubectl get pods --watch
```

### Discussion

How many attempts will make a `job` with a `.spec.backoffLimit` of `4`?

<details>
<summary>
Solution:
</summary>
Five, because it is a *backoff* limit (the initial run is not a backoff). Just count how many `pods` with `Error` status you get from the `kubectl get pods`.
</details>

</details>


## Succeed execution

* If you are using [tmux](https://github.com/tmux/tmux/wiki), you can watch the pods in another pane

```bash
tmux split-window -d "kubectl get pods --watch" 
```

* Run again the job, and wait for a few attempts

```bash
kubectl delete -f job.yaml
kubectl apply -f job.yaml
sleep 60
```

* Updated the exit value in the `keyvalue` database

```bash
curl -X POST -d "" "https://keyvalue.immanuel.co/api/KeyVal/UpdateValue/${AUTHKEY:1:8}/job-$USER/0"; echo
```

* Look at the list of pods and see how the newest one succeeds
  
* Just for fun, use the raw API call to get information about the completed job:

```bash
kubectl get --raw /apis/batch/v1/namespaces/demo-$USER/jobs  | jq '.items[] | select(.metadata.name="demo-job").status'
```

* If you are using `tmux`, delete the `--watch` pane

```bash
tmux kill-pan -t 2
```

## Cleanup

* Let's remove everything

```bash
kubectl delete ns demo-$USER
```
