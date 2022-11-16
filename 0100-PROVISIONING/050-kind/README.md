# Dev cluster with Kind

[Kind](https://kind.sigs.k8s.io/) is a tool to create simple Kubernetes clusters, uselful for learning  
and experimenting because each node is implemented inside a container in a single machine.


## Preparation

* Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

* Install Docker

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
sudo usermod -aG docker ${USER}
su - ${USER}
```

* Install Go (the instructions are written for `arm64` architecture. Adapt them if you are using `AMD64` or `Darwin`)

```bash
wget https://go.dev/dl/go1.19.3.linux-arm64.tar.gz
tar xvf go1.19.3.linux-arm64.tar.gz 
sudo mv go/ /usr/local/bin/
PATH=$PATH:/usr/local/bin/go/bin/
```

* Install Kind

```bash
go install sigs.k8s.io/kind@v0.17.0
```

## Configure the cluster

* Define the components of your cluster

```bashcat << EOF > kind-config.yaml 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```

* Create the cluster

```
time ./go/bin/kind create cluster --name local-cluster --config kind-config.yaml ^C
```

* Update the config file (optional, as it is automatically updated by the `kind create cluster` command)

```bash
./go/bin/kind export kubeconfig --name local-cluster
```

## Validate cluster configuration

* Check the *nodes* infrastructure

```bash
docker ps
```

* List the `nodes` of the cluster

```bash
kubectl get nodes
```

## Cleanup

* Delete the cluster

```bash
 ./go/bin/kind delete cluster --name local-cluster
 ```
 


