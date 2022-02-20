# Logging with CloudWatch (from EKS)

## Dependencies

* Install `jq`

```bash
sudo apt install jq -y
```

## Setting IAM permissions

* Get the cluster name

```bash
CLUSTER_NAME=$(eksctl get clusters -o json | jq ".[].metadata.name" -r)
echo Your cluster is $CLUSTER_NAME.
```

* With the cluster name, get the stacks that created it (one for each nodegroup)

```bash
STACK_NAMES=$(eksctl get nodegroup --cluster $CLUSTER_NAME -o json | jq -r '.[].StackName')
echo Stack names: $STACK_NAME
```

* For each stack, get the instance role and add the policy used by the [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html) to it

```bash
for CURRENT_STACK in $STACK_NAMES
do
    echo Processing stack $CURRENT_STACK
    ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $CURRENT_STACK | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')

    echo Instance role is $ROLE_NAME

    aws iam attach-role-policy \
    --role-name $ROLE_NAME \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy  

    echo Policy attached
done
```

## CloudWatch Agent deployment

* Set the configuration

```bash
REGION=$(aws configure get region)
FB_PORT=\"2020\"
FB_READ_FROM_HEAD=\"Off\"

[[ ${FB_READ_FROM_HEAD} = \"On\" ]] && FB_READ_FROM_TAIL=\"Off\"|| FB_READ_FROM_TAIL=\"On\"

[[ -z ${FB_PORT} ]] && FB_HTTP_SERVER=\"Off\" || FB_HTTP_SERVER=\"On\"
```

* Download the manifests and configure them

```bash
curl -o insights.yaml https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml 

sed -i  "s/{{cluster_name}}/${CLUSTER_NAME}/g;s/{{region_name}}/${REGION}/g;s/{{http_server_toggle}}/${FB_HTTP_SERVER}/g;s/{{http_server_port}}/${FB_PORT}/g;s/{{read_from_head}}/${FB_READ_FROM_HEAD}/g;s/{{read_from_tail}}/${FB_READ_FROM_TAIL}/g" insights.yaml
```

* Apply the new resources

```bash
kubectl apply -f insights.yaml
```

* Check everything is in place

```bash
kubectl get all -n amazon-cloudwatch
```

## Main resources exploration

* Calculate the URL for each service:

```bash
echo "
CloudWatch Container Insights:

https://${REGION}.console.aws.amazon.com/cloudwatch/home?region=eu-west-1#container-insights:"

echo "
CloudWatch Container Insights map:
https://${REGION}.console.aws.amazon.com/cloudwatch/home?region=${REGION}#container-insights:infrastructure/map?~(query~()~context~(timeRange~(delta~3600000)))"


echo "
CloudWatch Logs for the cluster $CLUSTER_NAME:

https://${REGION}.console.aws.amazon.com/cloudwatch/home?region=${REGION}#logsV2:log-groups$3FlogGroupNameFilter$3D${CLUSTER_NAME}"
```

## Challenge

<details>
<summary>
* Deploy the [pokemon]() application
* Use [siege](https://github.com/JoeDog/siege) to perform a test load

</summary>

```bash
SERVICE_NAME=<your service name>
ADDR=$(kubectl get services $SERVICE_NAME -ojsonpath='{.status.loadBalancer.ingress[0].hostname}')

sudo apt install siege -y
siege  --concurrent=100 --time=10s --delay=1 $ADDR
```
</details>

## Cleanup

* Delete the resources in your cluster

```bash
kubectl delete -f insights.yaml
```

* Remove the node policy

```bash
for CURRENT_STACK in $STACK_NAMES
do
    echo Processing stack $CURRENT_STACK
    ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $CURRENT_STACK | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')

    echo Instance role is $ROLE_NAME

    aws iam detach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    --role-name ${ROLE_NAME}

    echo Policy dettached
done
```

