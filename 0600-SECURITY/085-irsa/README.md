# Associating IAM policies to Pods

## Environment configuration

* Create a namespace

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

* Get the name of the cluster

```bash
CLUSTER_ARN=$(kubectl config view --minify -o jsonpath='{.clusters[].name}')
CLUSTER_NAME=${CLUSTER_ARN#*/}
CLUSTER_NAME=${CLUSTER_NAME%%.*}
echo The name of the cluster is $CLUSTER_NAME.
```

* Stablish trust relationship between the cluster and IAM using web federation:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --approve \
  --region eu-west-1
```

* Get the `arn` of the desired policy

```bahs
S3_RO_POLICY=$(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn' --output text)
echo The ARN of the S3 read-only policy is $S3_RO_POLICY.
```

* Remember: `serviceaccounts` are namespaced resources:

```bash
kubectl api-resources | awk 'NR==1 || /(^serviceaccount)/'
```

* Create a the `serviceaccount` associated to our permission policy for our active `namespace`

```bash
eksctl create iamservice▒▒▒▒▒▒▒ \
    --name aws-s3-ro-demo-sa \
    --namespace demo-$USER \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn $S3_RO_POLICY \
    --approve \
    --override-existing-serviceaccounts \
    --region eu-west-1
```

* Check everything is in place

```bash
kubectl get sa aws-s3-ro-demo-sa
```

## Testing permissions

* Try to run a `pod` without any `serviceaccount` and see how it fails while trying to access S3 on AWS

```bash
kubectl run aws-command-with-role-$RANDOM \
  -it \
  --rm \
  --image=amazon/aws-cli:latest \
  --restart=Never \
  --namespace demo-$USER \
  --command \
  -- aws s3 ls
```

* Allow the pod to make use of the `serviceaccount` to be able to read content from S3

```bash
kubectl run aws-command-with-role-$RANDOM \
  -it \
  --rm \
  --image=amazon/aws-cli:latest \
  --overrides='{ "spec": { "serviceAccount": "aws-s3-ro-demo-sa" }}' \
  --restart=Never \
  --namespace demo-$USER \
  --command \
  -- aws s3 ls
```

* Also see how access to other resources (EC2, in this case) is forbidden:

```bash
kubectl run aws-command-with-role-$RANDOM \
  -it \
  --rm \
  --image=amazon/aws-cli:latest \
  --overrides='{ "spec": { "serviceAccount": "aws-s3-ro-demo-sa" }}' \
  --restart=Never \
  --namespace demo-$USER \
  --command \
  -- aws ec2 describe-instances
```


## Clean up

* Delete the namespace

```bash
kubectl delete ns demo-$USER
```
