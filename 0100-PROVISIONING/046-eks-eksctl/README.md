# EKS with `eksctl`

This lab will provide you with the instructions to create a managed k8s cluster on AWS using the third party tool `eksctl`.

## Dependencies

* Install the commands `kubectl`, `eksctl`, `jq` and `yq`

```bash
sudo python3 -m pip install awscli

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/eksctl

sudo apt-get install jq
sudo snap install yq
```

## Cluster creation

* Write the cluster definition

``` yaml
cat << EOF > cluster.yaml

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: training
  region: eu-west-1
  version: '1.22'
  tags:
    karpenter.sh/discovery: training
iam:
  withOIDC: true

iamIdentityMappings:
  - arn: arn:aws:iam::xxxxxxxxx:role/xxxxxxxxxxxxxxx
    groups:
      - system:masters
    username: admin
    noDuplicateARNs: true

managedNodeGroups:
  - name: main-ng
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 0
    maxSize: 10
    labels:
      creator: "$USER"    
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true

cloudWatch:
    clusterLogging:
        enableTypes: ["all"]
        logRetentionInDays: 30

EOF
```

* Apply the configuration

```bash
time eksctl create -f cluster.yaml
```

## Use the cluster

* Wait something like 20 minutes and then take a look at the launched `stacks`

```bash
aws cloudformation describe-stacks | jq -r .Stacks[].StackName | grep $USER
eksctl get clusters
```

* Check the cluster and the nodes (Note the lack of visibility over the masters):

```bash
kubectl version
kubectl get nodes
```

* See how there is plenty of pods already deployed in the cluster

```bash
kubectl get pods -n kube-system -owide
```

* It is always possible to rebuild the configuration with:

```bash
export AWS_DEFAULT_REGION=eu-west-1
CLUSTER_NAME=$(eksctl get clusters -o json | jq ".[].metadata.name" -r) && echo Your cluster is $CLUSTER_NAME.
aws eks --region eu-west-1 update-kubeconfig --name $CLUSTER_NAME
kubectl config set-context --namespace demo-$USER --current
```

## Clean up

* Delete the cluster

```bash
eksctl delete cluster --name training
```
