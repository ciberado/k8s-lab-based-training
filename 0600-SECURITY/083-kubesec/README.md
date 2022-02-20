# KubeSec

This tool allows to easily check some security rules, both in deployed resources and directly using their manifest. More information in [the github repo](https://github.com/controlplaneio/kubectl-kubesec).

## Installation

* Download it and install

```bash
wget https://github.com/controlplaneio/kubesec/releases/download/v2.8.0/kubesec_linux_amd64.tar.gz
tar zxvf kubesec_linux_amd64.tar.gz
sudo mv kubesec /usr/local/bin/
```

* Check if it is correctly installed

```bash
kubesec version
```

## Usage

* Create a manifest

```bash
cat << EOF > demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pokemon
  labels:
    app: pokemon
    type: webapp
spec:
  containers:
  - name: web
    image: ciberado/pokemon-nodejs:0.0.1
EOF
```

* Check the manifest file before deploying it

```bash
kubesec scan demo-pod.yaml | jq ".[].scoring.advise[].reason" -r
```


## Plugin configuration

* Install the `krew` plugin manager

```bash
(
  set -x; cd "$(mktemp -d)" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&
  "$KREW" install krew
)

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew update
```

* Install the `kubectl` plugin

```bash
kubectl krew install kubesec-scan
```

* Check it

```bash
kubectl plugin list
kubectl kubesec-scan version
```


* Deploy the application

```bash
kubectl create ns demo-$USER
kubectl apply -f demo-pod.yaml -n demo-$USER
```

* Check the deployed resource

```bash
kubectl kubesec-scan pod pokemon -n demo-$USER
```
