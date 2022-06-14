# Service accounts

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## API Server access with kubectl

* Execute this command to get the list of `pods` in your cluster (in all `namespaces`)

```bash
kubectl get --raw /api/v1/pods | jq
```

* Feel free to explore the output with `jid` or `jq`

```bash
kubectl get --raw /api/v1/pods | jq .items[].metadata.name
```

## Service Account

* Define the `ServiceAccount` resource

```yaml
cat << EOF > demo-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-sa
EOF
```

* Create it

```bash
kubectl apply -f demo-sa.yaml
```

## Role

* Define the permissions that we want to associate to the `ServiceAccount`

```yaml
cat << EOF > list-pods-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1 
metadata:
    name: list-pods-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

* Create the `Role`

```bash
kubectl apply -f list-pods-role.yaml
```

## RoleBinding

* Define the resource for linking the `Role` with the `ServiceAccount`

```yaml
cat << EOF > demo-sa-list-pods-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-sa-list-pods-rolebinding
subjects:
- kind: ServiceAccount
  name: demo-sa
roleRef:
  kind: Role
  name: list-pods-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

* Apply the file

```bash
kubectl apply -f demo-sa-list-pods-rolebinding.yaml
```

## Starting the pod

* Create a `pod` to play with it's permissions (check the use of `.spec.serviceAccountName`)

```yaml
cat << EOF > demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  serviceAccountName: demo-sa
  containers:
  - name: bash
    image: bash
    command: ['sh', '-c', 'echo bash pod started && sleep 60000']
EOF
```

* Launch the pod

```
kubectl apply -f demo-pod.yaml
```

* Install `curl` and `jq` in the `pod` main container

```bash
kubectl exec -it demo -- apk add curl
kubectl exec -it demo -- apk add jq
```

## Checking the permissions

* Create a script for accessing the `API server`

```bash
cat << 'EOF' > kctl.sh
#!/bin/bash

APISERVER=https://kubernetes.default.svc
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
echo Current namespace: $NAMESPACE
TOKEN=$(cat ${SERVICEACCOUNT}/token)
echo Service Account token: $TOKEN
CACERT=${SERVICEACCOUNT}/ca.crt
echo Certification Authority location: $CACERT
PATH=${APISERVER}${1}
echo API call path: $PATH

/usr/bin/curl -s --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET $PATH | /usr/bin/jq ${2}
EOF
```

* Copy the file to the `pod`

```bash
kubectl cp ./kctl.sh demo:/kctl.sh
```

* Run the script inside the `pod`, using the first argument as the `api` call path

```bash
kubectl exec -it demo -- bash kctl.sh /api/v1/namespaces/demo-$USER/pods
```

* It is possible to use the second argument to filter the output of the execution

```bash
kubectl exec -it demo -- bash kctl.sh /api/v1/namespaces/demo-$USER/pods .items[].metadata.name
```

* Try to get all the `pods` in all `namespaces` (it will fail, as the `pod` doesn't have enought permissions)

```bash
kubectl exec -it demo -- bash kctl.sh /api/v1/pods
```

## Cleanup

* Delete everything

```bash
kubectl delete ns demo-$USER
```
