# Secrets

* Secrets are the preferred way to expose sensible configuration to a `pod`
* If the infra is not 100% k8s it makes sense to use a centralized external repository
* Kubernetes `Secrets` can be easily linked to the *AWS Parameter store* and *AWS Secrets Manager* leveraging (the ASCP provider)[https://docs.aws.amazon.com/systems-manager/latest/userguide/integrating_csi_driver.html]
* Create a secret (**notice base64 is NOT encrypting anything**)

# Understanding base64

* `base64` is a codification, not an encryption mechanism: as you can see, messages can be easily transformed to plain text

```bash
PLAIN="hello"
BASE64=$(echo $PLAIN | base64 -w0) && echo Base65 codification is $BASE64.
echo $BASE64 | base64 --decode
```

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Creating secrects


* Define a `secret` in the manifest

```yaml
cat << EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: project-secrets
type: Opaque
data:
  password: "bXlzdGVyaW91cwo="
  apikey: "MTIzNDU2NzgK"
EOF
```

* Apply it

```
kubectl apply -f secret.yaml
```

* Get the secrets from command line

```bash
kubectl get secret project-secrets -ojsonpath='{.▒▒▒▒.▒▒▒▒▒▒▒▒}' | base64 --decode
kubectl get secret project-secrets -ojsonpath='{.▒▒▒▒.▒▒▒▒▒▒}' | base64 --decode
```

## Secret access from applications

* Create the manifest for running a `pod`

```yaml
cat << 'EOF' > pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secret
spec:
  containers:
  - name: main
    image: bash
    env:
      - name: APIKEY
        valueFrom:
          secretKeyRef:
            name: project-secrets
            key: apikey
    command: ['sh', '-c', 'echo "My (now not so) secret API KEY is $APIKEY." && sleep 600']
    volumeMounts:
    - name: configuration
      mountPath: "/configuration"
  volumes:
  - name: configuration
    secret:
      secretName: project-secrets
      items:
      - key: password
        path: db/password
        mode: 511
EOF
```

* Deploy the `pod`

```bash
kubectl apply -f pod-with-secret.yaml
```

* Check the api key secret

```bash
kubectl logs pod-with-secret
```

* Access the container and read the secrets

```bash
kubectl exec -it pod-with-secret -- cat configuration/db/password
kubectl exec -it pod-with-secret -- printenv APIKEY
```

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```