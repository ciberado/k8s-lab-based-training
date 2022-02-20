```
cat << EOF | kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp-restrictive
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
```

```
cat << EOF | kubectl apply -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-restrictive-role
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - psp-restrictive
  verbs:
  - use
EOF
```

```
cat << EOF | kubectl apply -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-default
subjects:
- kind: Group
  name: system:serviceaccounts
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: psp-restrictive-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.15.4
EOF
```

```
# will fail
kubectl get po,rs,deploy 
```

```
cat << EOF > Dockerfile
FROM nginx:1.15.4

USER 1001
CMD ["nginx", "-g", "daemon off;"]
EOF

docker login
docker build -t ciberado/nginx-non-root .
docker push ciberado/nginx-non-root .
```

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-non-root
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
```

```
# will start the container buuuuuuuut... "nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)"
kubectl get po,rs,deploy 
kubectl logs ...
```

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon-deployment-sc
  labels:
    app: nginx
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
```
