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
  name: $USER-cluster
  region: eu-west-1
  tags: {
    Creator : $USER
  }
nodeGroups:
  - name: ng-unmanaged-mixed
    minSize: 1
    desiredCapacity: 4
    maxSize: 8
    instancesDistribution:
      instanceTypes: ["c3.large","c4.large","c5.large","c5d.large","c5n.large","c5a.large"]
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 25
      spotAllocationStrategy: "capacity-optimized"

managedNodeGroups:
  - name: ng-extra-capacity
    instanceTypes: ["c3.large","c4.large","c5.large","c5d.large","c5n.large","c5a.large"]
    spot: true
    minSize: 1
    desiredCapacity: 1
    maxSize: 4
    privateNetworking: true
    tags:
      Owner: $USER
  - name: ng-base-capacity
    instanceType: c5a.large
    minSize: 1
    desiredCapacity: 1
    maxSize: 8
EOF
```

* Apply the configuration

```bash
time eksctl apply -f cluster.yaml
```

## Use the cluster

* Wait something like 20 minutes and then take a look at the launched `stacks`

```bash
aws cloudformation describe-stacks | jq -r .Stacks[].StackName | grep $USER
eksctl get clusters
```

* Find the name of the attached role

```bash
INSTANCE_PROFILE_NAME=$(aws iam list-instance-profiles | jq -r '.InstanceProfiles[].InstanceProfileName' | grep ${USER})
echo $(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Roles[] | .RoleName')
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

## Challenge

<details>
<summary>
Find a way to show all nodes, reporting about their node group and spot nature. Tips: you can use `kubectl describe`
to easily analyze information about them, and `kubectl get nodes` with selectors to show the relevant data.
</summary>

```
kubectl get nodes -Lnode-lifecycle,alpha.eksctl.io/nodegroup-name | awk 'NR == 1; NR > 1 {print $0 | "sort -n -r -k4"}'
```
</details>

## Clean up

* Delete the cluster

```bash
eksctl delete cluster --name ${USER}-cluster
```
