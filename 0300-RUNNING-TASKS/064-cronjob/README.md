# Cronjobs

`Cronjobs` are a way to execute scheduled [jobs](../063-jobs/README.md). So, by defining a `Cronjob`, you will create a `job` each *x* minutes

Remember: `spec.jobtemplate.spec.completions` is the number of `completed` pods ran by each `job` to mark it as succeeded, so it should be equal or less than `spec.jobtemplate.spec.parallelism` (which is the number of pods executed on each `job`).

Currently, there is no way to set a finish date for the resource.

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Scheduled tasks

* Create this `cronjob` definition

```yaml
cat << 'EOF' > cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: demo-cronjob
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 59
  concurrencyPolicy: "Allow"
  successfulJobsHistoryLimit: 10
  failedJobsHistoryLimit: 10
  jobTemplate:
    spec:
      backoffLimit: 5
      ttlSecondsAfterFinished: 100
      parallelism: 2
      completions: 2
      template:
        spec:
          restartPolicy: Never
          containers:
          - image: bash
            name: bashtask
            command: ["/usr/local/bin/bash"]
            args: ["-c", "STARTDATE=$(date); for i in {1..90}; do echo \"I started at $STARTDATE ($i)\"; sleep 1; done"]
EOF
```

<details>
<summary>
What does the `spec.schedule` expression means?
</summary>

*Execute a new job each minute (as explained [here](https://www.baeldung.com/cron-expressions)).*
</details>

* Execute the manifest

```bash
kubectl apply -f cronjob.yaml
```

* See how the name of the pod is being increased for each job, and how every now and then they are automatilly removed from the list as they expire.

```bash
watch kubectl get pods --sort-by=.status.startTime 
```

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER --force
```
