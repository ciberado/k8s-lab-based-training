# Argo CD

You will find in the documentation how to declare apps and projects for ArgoCD using yaml syntax instead of creating them with the web UI:

[declarative setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)


## Installation ##

We can install ArgoCD in our cluster with the following commands:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.1.3/manifests/install.yaml
```

We will need a service to access its web UI. Usually, we will do it with traefik, but for now, we will use just an LB:

```bash
cat << EOF > argocd-service.yaml
# Service used to create the load balancer for the internal endpoint
apiVersion: v1
kind: Service
metadata:
  name: argocd-external
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/backend-protocol: "http"
    alb.ingress.kubernetes.io/target-type: "instance"
    alb.ingress.kubernetes.io/healthcheck-path: "/ping"
    #alb.ingress.kubernetes.io/inbound-cidrs: "13.95.217.233/32"
  labels:
    app.kubernetes.io/name: argocd-server
spec:
  type: LoadBalancer
  ports:
  - name: web
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
EOF
```

Apply the previous yaml:

```bash
kubectl apply -f argocd-service.yaml
```

Then get the loadbalancer dns, and visit the web site:

```bash
kubectl -n argocd get svc --field-selector 'metadata.name=argocd-external'
```

<details>
<summary>
Why is it not working? Why you think you can't reach the ArgoCD login page?
</summary>
Because, by default, ArgoCD runs in secure mode, so it expects to receive requests only with HTTPS, but if you check our LoadBalancer yaml, it's only supporting HTTP and port 80.

So, we are going to deactivate the secure mode:

```bash
kubectl -n argocd edit deployment argocd-server
```

Add `--insecure` option into "Command" section and then check argocd-server pods are restarted:

```bash
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-server
```
</details>

How we access to ArgoCD web UI? By retrieving its password with the command:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo ''
```

Notice that `argocd-initial-admin-secret` must be deleted after you stored the password. It is only intended to be during ArgoCD installation, and once you retrieve the password, you  must delete that secret.

## Laboratory ##
So, now that we have our ArgoCD installed, and we can access the UI, let's start setting our app!

First, we will need a repository. Clone the repository in your student shell:

```bash
#set user and email for Git globally
git config --global user.email "$USER@example.com"
git config --global user.name "$USER"

# Add ssh credentials to access repository
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/private

# clone repository
git clone git@github.com:ciberado/k8s-training-pokemon-app.git
cd k8s-training-pokemon-app
git checkout -b my-branch-$USER
```

Create the code for your application:

```bash
cat << EOF > deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon-$USER
  namespace: demo-argocd-$USER
spec:
  selector:
    matchLabels:
      app: pokemon-$USER
  replicas: 2
  strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0    
  template:
    metadata:
      labels:
        app: pokemon-$USER
        project: projectpokemon
        version: 0.0.1
    spec:
      containers:
      - name: pokemon-container
        image: ciberado/pokemon-nodejs:0.0.1
        ports:
        - containerPort: 80
EOF

cat << EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pokemon-$USER-service
  namespace: demo-argocd-$USER
  labels:
    app: pokemon-$USER
spec:
  selector:
    app: pokemon-$USER
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

Commit your changes to the repository:

```bash
git add deployment.yaml service.yaml
git commit -m "Create my fabulous application"
git push origin my-branch-$USER
```

Then, we can create our repository. We will choose SSH option, but you can try both if you want:

<details>
<summary>
Using SSH, then forced to use an SSH key to login.
</summary>
The following public key is already set in your repository, so that we can set an SSH connection to it:

```text
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCeu8okdDA4B+Mx8GvZ76rYab6s3M8KM29kymbei+oHtjSfVBaxHyME8YwN+KtnwB5lHFTWUXfETIZ47evjay5ALNx817ukdqrNSAldMyzYf+SJ+/F7bcp9AWf4P98qqHLTD7Bjrud1Pkt/ikzkpDGoeFreCG+hmpjXqHX1LsERYnH4aTdFqZiCOBf4h4POC1R+fPyVT/9HV0tMHz8Iu0/eJeB69DpjkZivVZ4BjLIpBq7HkXJf8jQo5A3d4BhbXvLlwhaI4IsPT0Uxpjdm9v8OJ7wmRwnciDpQelFsCEunNpxGteOb3DOPi3kGnd5pGseSrUv6KzPAaMB6p2omhJO9 capside@GAMUSINO
```

Then create the secret to set up the repository, using the private key of the previous public key:

```bash
cat << EOF > argocd-repo-secret-ssh.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-private-repo-$USER
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # Setting a private repository
  url: git@github.com:ciberado/k8s-training-pokemon-app.git
  name: repo-$USER-ssh
  insecure: "true"
  type: git
  sshPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEAnrvKJHQwOAfjMfBr2e+q2Gm+rNzPCjNvZMpm3ovqB7Y0n1QW
    sR8jBPGMDfirZ8AeZRxU1lF3xEyGeO3r42suQCzcfNe7pHaqzUgJXTMs2H/kifvx
    e23KfQFn+D/fKqhy0w+wY67ndT5Lf4pM5KQxqHha3ghvoZqY16h19S7BEWJx+Gk3
    RamYgjgX+IeDzgtUfnz8lU//R1dLTB8/CLtP3iXgevQ6Y5GYr1WeAYyyKQaux5Fy
    X/I0KOQN3eAYW17y5cIWiOCLD09FMaY3Zvb/Die8JkcJ3Ig6UHpRbAhLpzacRrXj
    m9wzj4t5Bp3eaRrHkq1L+iszwGjAeqdqJoSTvQIDAQABAoIBAAtBV5EEKBMhBAwb
    dxpJ8zxLKzkIoymfAgwMigTHuP14/vw5My61/X6xPfQTqNu/dKhIvP9BYZOqtXJH
    tI04oVvtkmjLx0NfIrdRn1Bbe5eSYfsiwTm2TEBW5C9nIATfUt0CZMh8s27Nzv6p
    KNChj9/ZQOAziu7TjnjkOhD7krcPvlcVWEPuym0esLUuTqyOQsro839wFLu+Do25
    MmsnHhD8FQhztYOy+zKmtXm8Or2TYCbeEVh9CKTttW8ojThxQGUiPpqu/yq1oX5x
    wzkei7JcgTPJcubuS7wkkIsVz7O1TlssyXOoVdPKccpLljA/l9fY6vNy2Q+FUSc+
    gGYHB4ECgYEA0Z2CkptwCeQZbTT1UYvBnN7Bu7SjElau2v2uBTlVY9vUbhZJDxer
    hctVCZNBP7f/z0Sjwqf5Z1wpPAmgUTrCvWkY2QgUnR9YHGGMjloZhwmgc4DLlR0L
    QwtYWivgYIyE6t5A3sBwQhfLjJWVciEgOpKqN9uAXUCz/mMb+tWzCeECgYEAwdvh
    4uaLZmhKhxK7y/c0bwG/28RwGPdV1/EGSicEBCHuZTKNM9AV87r3Q0KD1Q/l471r
    1ERtv04PXJs7g90Zi8ppdhVzFCFkR7a+ueoBnhbqzJG5ctqi4sCudfmtWzzLQDpl
    AjA3QEsq4AF0z+RHc03dy1OehmDyCRI4UyLLnV0CgYB4iTCqiYOVzHrql4dyCwGc
    6WNSQv967inCeBn3mw6FS8YOP/ZnHV9eopwV032z3GTXlUruBpWeYBq+EXMFAts0
    /BhzxPfFml6ag2XF/f2r71c61Bc9eeQd+4ok4BI4stVEEeYPsW1cND6yatnzNSVJ
    SUlksW5RMYHPiMJwLS61QQKBgGa7SWde2Ty5w9T0voSGSkkRWkTyQp1YZSt8VOLy
    7hPqj1UdhuqQOTHiQKpqE0bTl/YqKXxhju80RLvEn7Nvddw4tc6X61Ydo/DFDSmk
    spq+dktWZjpRVsRna4yldZLGEsfEqkaQmpb9voja/LY2uQ6HkyPu+jEoKttXxnV4
    GQMZAoGBAKJ+6CSzsl2Opvyxx+sJ9fytv+0yemG66Ra2iec3+YUkSP2dhuEFBnCU
    y8GmMlyxUbmGeV4ZWnBo1QSPp1d1bxdMRWrGPDeBK0/kbf96BQqUjvtxq1sO+Vkh
    O1WxIAmqyb6wJtxG2KZammTrzkyW8kq13n/HKWXZ5FGTZFIx47ca
    -----END RSA PRIVATE KEY-----
EOF

kubectl apply -f argocd-repo-secret-ssh.yaml

```
</details>

<details>
<summary>
or using HTTPS, this way you are not forced to use a user and password, therefor, can be set this way for public repositories
</summary>

**WARNING** if you try this option, make sure you modify `Application` and `AppProject` yaml manifests later. You will need to set the URL correctly for https usage, instead of the one provided for this lab.

```bash
cat << EOF > argocd-repo-secret-https.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-public-repo-$USER
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # Setting a public repository
  url: https://github.com/ciberado/k8s-training-pokemon-app.git
  name: repo-$USER-https
  #password: my-password
  #username: my-username
  insecure: "false"
  type: https
EOF

kubectl apply -f argocd-repo-secret-https.yaml
```
</details>

Remember, by default, secrets are not stored encrypted, they are stored coded in base64 and you can easily decode them:

```bash
 kubectl -n argocd get secret my-public-repo-$USER -ojson | jq -r .data.url | base64 --decode && echo '' 
```

So that a repo works properly, we need to add the server's keys to the ssh known hosts. Check that we have currently set ones for github.com. Github publishes they're servers ssh keys [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints). Verify that they are just the same ones that they provide in their web site servers:

```bash
ssh-keyscan -t rsa github.com | ssh-keygen -lf -
```

Then check the configmap:

```bash
kubectl -n argocd get configmap argocd-ssh-known-hosts-cm -oyaml
```

We will look for the rsa key in the configmap and convert it with `ssh-keygen` to verify is the same hash as the previous ones:

```bash
kubectl -n argocd get configmap argocd-ssh-known-hosts-cm -oyaml | yq .data.ssh_known_hosts | grep 'github.com ssh-rsa' | ssh-keygen -lf -
```

<details>
<summary>
The output of the previous command should be like:
</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-ssh-known-hosts-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-ssh-known-hosts-cm
  namespace: argocd
data:
  ssh_known_hosts: |
    bitbucket.org ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAubiN81eDcafrgMeLzaFPsw2kNvEcqTKl/VqLat/MaB33pZy0y3rJZtnqwR2qOOvbwKZYKiEO1O6VqNEBxKvJJelCq0dTXWT5pbO2gDXC6h6QDXCaHo6pOHGPUy+YBaGQRGuSusMEASYiWunYN0vCAI8QaXnWMXNMdFP3jHAJH0eDsoiGnLPBlBp4TNm6rYI74nMzgz3B9IikW4WVK+dc8KZJZWYjAuORU3jc1c/NPskD2ASinf8v3xnfXeukU0sJ5N6m5E8VLjObPEO+mN2t/FZTMZLiFqPWc/ALSqnMnnhwrNi2rbfg/rd/IpL8Le3pSBne8+seeFVBoGqzHM9yXw==
    github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==
    gitlab.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFSMqzJeV9rUzU4kWitGjeR4PWSa29SPqJ1fVkhtj3Hw9xjLVXVYrU9QlYWrOLXBpQ6KWjbjTDTdDkoohFzgbEY=
    gitlab.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAfuCHKVTjquxvt6CM6tdG4SLp1Btn/nOeHHE5UOzRdf
    gitlab.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCsj2bNKTBSpIYDEGk9KxsGh3mySTRgMtXL583qmBpzeQ+jqCMRgBqB98u3z++J1sKlXHWfM9dyhSevkMwSbhoR8XIq/U0tCNyokEi/ueaBMCvbcTHhO7FcwzY92WK4Yt0aGROY5qX2UKSeOvuP4D6TPqKF1onrSzH9bx9XUf2lEdWT/ia1NEKjunUqu1xOB/StKDHMoX4/OKyIzuS0q/T1zOATthvasJFoPrAjkohTyaDUz2LN5JoH839hViyEG82yB+MjcFV5MU3N1l1QL3cVUCh93xSaua1N85qivl+siMkPGbO5xR/En4iEY6K2XPASUEMaieWVNTRCtJ4S8H+9
    ssh.dev.azure.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7Hr1oTWqNqOlzGJOfGJ4NakVyIzf1rXYd4d7wo6jBlkLvCA4odBlL0mDUyZ0/QUfTTqeu+tm22gOsv+VrVTMk6vwRU75gY/y9ut5Mb3bR5BV58dKXyq9A9UeB5Cakehn5Zgm6x1mKoVyf+FFn26iYqXJRgzIZZcZ5V6hrE0Qg39kZm4az48o0AUbf6Sp4SLdvnuMa2sVNwHBboS7EJkm57XQPVU3/QpyNLHbWDdzwtrlS+ez30S3AdYhLKEOxAG8weOnyrtLJAUen9mTkol8oII1edf7mWWbWVf0nBmly21+nZcmCTISQBtdcyPaEno7fFQMDD26/s0lfKob4Kw8H
    vs-ssh.visualstudio.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7Hr1oTWqNqOlzGJOfGJ4NakVyIzf1rXYd4d7wo6jBlkLvCA4odBlL0mDUyZ0/QUfTTqeu+tm22gOsv+VrVTMk6vwRU75gY/y9ut5Mb3bR5BV58dKXyq9A9UeB5Cakehn5Zgm6x1mKoVyf+FFn26iYqXJRgzIZZcZ5V6hrE0Qg39kZm4az48o0AUbf6Sp4SLdvnuMa2sVNwHBboS7EJkm57XQPVU3/QpyNLHbWDdzwtrlS+ez30S3AdYhLKEOxAG8weOnyrtLJAUen9mTkol8oII1edf7mWWbWVf0nBmly21+nZcmCTISQBtdcyPaEno7fFQMDD26/s0lfKob4Kw8H
    github.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmKSENjQEezOmxkZMy7opKgwFB9nkt5YRrYMjNuG5N87uRgg6CLrbo5wAdT/y6v0mKV0U2w0WZ2YB/++Tpockg=
    github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
```
</details>

Create a project:

```bash
cat << EOF > argocd-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: project-$USER
  namespace: argocd
  # Finalizer that ensures that project is not deleted until it is not referenced by any application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # Project description
  description: I am going to become rich thanks to this app.

  # Allow manifests to deploy from any Git repos
  sourceRepos:
  #- '*'
  - git@github.com:ciberado/k8s-training-pokemon-app.git
  - https://github.com/ciberado/k8s-training-pokemon-app.git

  # Only permit applications to deploy to the coolapplication namespace in the same cluster
  destinations:
  - namespace: demo-argocd-$USER
    server: https://kubernetes.default.svc

  # Deny all cluster-scoped resources from being created, except for Namespace
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace

  # Allow all namespaced-scoped resources to be created, except for ResourceQuota, LimitRange, NetworkPolicy
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
  - group: ''
    kind: NetworkPolicy
  - group: ''
    kind: Secret

  # Deny all namespaced-scoped resources from being created, except for Deployment and StatefulSet
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: StatefulSet

  # Enables namespace orphaned resource monitoring.
  orphanedResources:
    warn: true

  ## list of entities with definitions of their access to resources within the project.
  #roles:
  ## A role which provides read-only access to all applications in the project
  #- name: read-only
  #  description: Read-only privileges to my-project
  #  policies:
  #  - p, proj:my-project:read-only, applications, get, my-project/*, allow
  #  groups:
  #  - my-oidc-group
EOF

kubectl apply -f argocd-project.yaml
```

Remember, you can check check available kinds, verbs, and if they are namespaced or not with kubectl CLI:

```bash
kubectl api-resources -owide --sort-by=name
```

Now deploy de app in argocd:

```bash
cat << EOF > argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application-$USER
  # You'll usually want to add your resources to the argocd namespace.
  namespace: argocd
  # Add a this finalizer ONLY if you want these to cascade delete.
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # The project the application belongs to.
  project: project-$USER

  # Source of the application manifests
  source:
    repoURL: https://github.com/ciberado/k8s-training-pokemon-app.git
    targetRevision: my-branch-$USER
    path: ./

    # directory
    directory:
      recurse: true

  # Destination cluster and namespace to deploy the application
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-argocd-$USER

  # Sync policy
  syncPolicy:
    automated: # automated sync by default retries failed attempts 5 times with following delays between attempts ( 5s, 10s, 20s, 40s, 80s ); retry controlled using retry field.
      prune: true # Specifies if resources should be pruned during auto-syncing ( false by default ).
      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).
      allowEmpty: false # Allows deleting all application resources during automatic syncing ( false by default ).
    syncOptions:     # Sync options which modifies sync behavior
    - Validate=false # disables resource validation (equivalent to 'kubectl apply --validate=false') ( true by default ).
    - CreateNamespace=true # Namespace Auto-Creation ensures that namespace specified as the application destination exists in the destination cluster.
    - PrunePropagationPolicy=foreground # Supported policies are background, foreground and orphan.
    - PruneLast=true # Allow the ability for resource pruning to happen as a final, implicit wave of a sync operation
    # The retry feature is available since v1.7
    retry:
      limit: 5 # number of failed sync attempt retries; unlimited number of attempts if less than 0
      backoff:
        duration: 5s # the amount to back off. Default unit is seconds, but could also be a duration (e.g. "2m", "1h")
        factor: 2 # a factor to multiply the base duration after each failed retry
        maxDuration: 3m # the maximum amount of time allowed for the backoff strategy

  # Ignore differences at the specified json pointers
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
EOF

kubectl apply -f argocd-app.yaml
```

Your application is failing at this point. Can you imagine the reason?

<details>
<summary>
Find out why and how to fix it here!
</summary>
You created the project, and set `namespaceResourceWhitelist`, but you didn't specify the kind Service. Add it to your configuration using the previous yaml file:

```yaml
  - group: ''
    kind: Service
```

Apply changes and check that it works now
</details>


