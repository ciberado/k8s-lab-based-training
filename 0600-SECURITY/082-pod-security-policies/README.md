# Pod security policies

## Basic concepts

## Creating the cluster (kops)

* In order to apply psp you have to enable an [admission controller](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/) in the *apiServer*
* Define a kops cluster **but don't deploy it**. Follow the instruction from [the kops lab](..\044-kops\README.md)

```bash
export AWS_PROFILE=<profile>
export AWS_DEFAULT_REGION=eu-west-1
export CLUSTER_NAME=<name>
export KOPS_STATE_STORE=s3://<bucket>
export DOMAIN=<domain>
export EDITOR=vim

kops create cluster \
  --state $KOPS_STATE_STORE \
  --name $CLUSTER_NAME.$DOMAIN \
  --master-size t2.medium \
  --master-count 1 \
  --master-zones eu-west-1c \
  --node-count 3 \
  --node-size t2.medium \
  --zones eu-west-1a,eu-west-1b,eu-west-1c \
  --networking calico  \
  --cloud-labels "Owner=kops,Project=$CLUSTER_NAME" \
  --ssh-public-key kops.pub \
  --authorization RBAC
```

* Edit the cluster

```bash
kops edit cluster --name $CLUSTER_NAME.$DOMAIN
```

* Add the admission control list

```diff
---
spec:
+++
spec:
  kubeAPIServer:
    admissionControl:
    - NamespaceLifecycle
    - LimitRanger
    - ServiceAccount
    - PersistentVolumeLabel
    - DefaultStorageClass
    - ResourceQuota
    - PodSecurityPolicy
    - DefaultTolerationSeconds
```

* Launch it

```bash
kops update cluster --name $CLUSTER_NAME.$DOMAIN --yes
kops rolling-update cluster --yes
```

* Wait until it is working 

```bash
kops validate cluster
```

## Psp behaviour

* Check *psp* are enabled

```bash
kubectl auth can-i use psp
kubectl get psp --all-namespaces
```

* Try to deploy a new pod. It will fail: a *deny all* is the default behaviour

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/controllers/nginx-deployment.yaml
```

* Check it has failed (we have no default psp)

```
kubectl get deployment nginx-deployment
kubectl get pods
```

* Get a proper explanation about why it failed

```bash
kubectl describe replicaset nginx-deployment
kubectl get events
```

* The output will contain something like this:

```yaml
status:
  conditions:
  - lastTransitionTime: 2019-04-03T14:53:28Z
    message: 'pods "nginx-deployment-67594d6bf6-" is forbidden: unable to validate
      against any pod security policy: []'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 1
  replicas: 0
```

* Delete the deployment

```bash 
kubectl delete deployment nginx-deployment
```

* System pods are attached to the `kube-system` policy. For example, check any of the `kube-proxy` pods

```bash
KUBE_PROXY_POD=$(kubectl get pod -n kube-system -l k8s-app=kube-proxy -ojsonpath="{.items[0].metadata.name}") && echo kube proxy: $KUBE_PROXY_POD
kubectl describe pod $KUBE_PROXY_POD -n kube-system | more
```

```bash
Annotations:        kubernetes.io/psp=kube-system
                    scheduler.alpha.kubernetes.io/critical-pod=
Status:             Running
```

## Configuring a new policy

* Check how the [restrive policy](restrictive-psp.yaml) disables *root* user

```yaml
cat << EOF > restrictive-psp.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restrictive-psp
spec:
  privileged: false
  hostNetwork: false
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  hostPID: false
  hostIPC: false
  runAsUser:
    rule: MustRunAsNonRoot 
  fsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'downwardAPI'
  - 'emptyDir'
  - 'persistentVolumeClaim'
  - 'secret'
  - 'projected'
  allowedCapabilities:
  - '*'
EOF

kubectl apply -f restrictive-psp.yaml
```

* It is *Kubernetes* itself who creates *pods* using *ServiceAccounts*:

```bash
kubectl get serviceaccounts --all-namespaces
```

* Create a `ClusterRole` and a `ClusterRoleBinding` to allow *ServiceAccounts* to use the *SecurityPolicy* described before:

```yaml
cat << EOF > allow-use-of-psp.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: use-restrictive-psp-role
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - restrictive-psp
  verbs:
  - use
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-default-role-binding
subjects:
- kind: Group
  name: system:serviceaccounts # All service accounts in the system (group)
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: use-restrictive-psp-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f allow-use-of-psp.yaml
```

## Running worloads under restrictive policies

* First we will try to use an image which uses the `root`user

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/controllers/nginx-deployment.yaml
```

* Describe the deployment to understand why it is failing (it uses ROOT)

```bash
kubectl get po,rs,deploy
```

* Look at this `Dockerfile` of our *nginx* image, which basically creates a non-root version of `nginx` (it is available at the [Docker hub](https://cloud.docker.com/repository/docker/ciberado/nginx-non-root))

```Dockerfile
FROM nginx:1.15.4

USER 1001
CMD ["nginx", "-g", "daemon off;"]
```

* Try to deploy a pod with that image (**fails again: access denied**)

```bash
cat << EOF > nginx-non-root-deployment-will-fail
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-non-root-deployment-will-fail
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: ciberado/nginx-non-root
EOF

kubectl apply -f nginx-non-root-deployment-will-fail.yaml
```

* Check the pod of the deployment: it has been created but the nginx cannot start because it tries to access a protected file using non-root user (`"nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)"`)

```bash
NGINX_FAILED_POD=$(kubectl get pod  -l app=nginx -ojsonpath="{.items[0].metadata.name}")
kubectl get pods $NGINX_FAILED_POD
kubectl logs $NGINX_FAILED_POD
```

* Delete the failure:

```bash
kubectl delete deployment nginx-non-root-deployment-will-fail
```

* Deploy an `nginx` using a [Bitnami image]()

```bash
cat << EOF > nginx-non-root-deployment-will-succeed.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-non-root-deployment-will-succeed
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: bitnami/nginx:latest
EOF

kubectl apply -f nginx-non-root-deployment-will-succeed.yaml
```

* Check how everythig is running smoothly now:

```bash
NGINX_SUCCEED_POD=$(kubectl get pod  -l app=nginx -ojsonpath="{.items[0].metadata.name}")
kubectl get pods $NGINX_SUCCEED_POD
kubectl logs $NGINX_SUCCEED_POD
```

* See how it is possible to set the `uid` of the container directly from the descriptor file. Remember: non-root users cannot access ports under 1024, so we need to use port 8080.

```yaml
cat << EOF > pokemon-deployment-sc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon-deployment-sc
  labels:
    app: pokemon
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pokemon
  template:
    metadata:
      labels:
        app: pokemon
    spec:
      containers:
      - name: pokemon
        image: ciberado/pokemon-nodejs:0.0.1
        env:
        - name: PORT
          value: "8080"
        securityContext:
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
EOF

kubectl apply -f pokemon-deployment-sc.yaml
```

* Check the pod is up and running by taking a look at its logs

```yaml
POKEMON_POD=$(kubectl get pod  -l app=pokemon -ojsonpath="{.items[0].metadata.name}")
kubectl get pods $POKEMON_POD
kubectl logs $POKEMON_POD
```