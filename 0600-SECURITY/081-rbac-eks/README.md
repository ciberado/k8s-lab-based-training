# RBAC authorization on EKS


![AWS rbac integration diagram](aws-k8s-rbac.png)

## IAM Authentication configuration

As AWS administrator of the account containing the cluster

### IAM Role

* Set the region, in case it is necessary

```bash
export AWS_DEFAULT_REGION=eu-west-1
```

* Get your account ID

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo Your account is $ACCOUNT_ID
```

* Define the role the user will assume (we prefix it with `$USER` to avoid role name conflicts, but in a real environment that would not happen):

```
AWS_ROLE_TRUST_POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

echo $AWS_ROLE_TRUST_POLICY | jq

aws iam create-role \
  --role-name AWSDeveloperRole${USER} \
  --description "Kubernetes developer role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$AWS_ROLE_TRUST_POLICY" \
  --output text \
  --query 'Role.Arn'
```

As AWS administrator of the account in which de AWS user is going to be accreditet.


### IAM Group

* Create a group for users that will be able to assume the role (and list the clusters)

```bash
aws iam create-group --group-name AWSMyProjectGroup${USER}
```

* Attach a policy to it, allowing its members to assume the role `AWSDeveloperRole${USER}`

```bash
ASSUME_AWSDEVELOPERROLE_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeAWSDeveloperRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ":role/AWSDeveloperRole${USER}"; echo -n '"
    },
    {   "Sid" : "AllowClusterListing",
        "Effect": "Allow",
        "Action": [
            "eks:DescribeCluster",
            "eks:ListClusters",
            "cloudformation:ListStacks",
            "cloudformation:DescribeStacks"
        ],
        "Resource": "*"
    }    
  ]
}')
echo $ASSUME_AWSDEVELOPERROLE_POLICY | jq

aws iam put-group-policy \
--group-name AWSMyProjectGroup${USER} \
--policy-name AssumeAwsRolePolicy${USER} \
--policy-document "$ASSUME_AWSDEVELOPERROLE_POLICY"
```

* Check if the group is in place

```bash
aws iam list-groups --output table
```

### IAM User

* Create the user

```bash
aws iam create-user --user-name AWSUserAlice${USER}
```

* Enroll the user in the dev group

```bash
aws iam add-user-to-group --group-name AWSMyProjectGroup${USER} --user-name AWSUserAlice${USER}
```

* Check everything is fine

```bash
aws iam get-group --group-name AWSMyProjectGroup${USER}
```

* Generate AK/SC credentials (for later sending them to the IAM user). Alternatively, let the user generate their own AK/SK

```bash
aws iam create-access-key --user-name AWSUserAlice${USER} | tee /tmp/AWSUserAlice${USER}.json
```

## EKS Authorization Configuration

As the cluster administrator

We are going to provide permissions to manage `pods`, `deployments`, `services`, `secrets` and `events` in a particular `namespace`

* Create the ns

```bash
kubectl create ns dev-$USER
```

* Check the required `api-groups` and *verbs* for the resources we are going to use:

```bash
kubectl api-resources -owide \
  | awk 'NR==1 || /(^pods )|(deployment)|(^services )|(secrets)|(configmap)|(events)/'
```

* Define the `Role` with the list of permissions associated to the user that we will create later

```yaml
cat << EOF > dev-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: k8s-dev-role-$USER
rules:
  - apiGroups:
      - ""                     # pods
      - "apps"                 # deployments, services, secrets...
      - "events.k8s.io"        # events
    resources:
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "deployments"
      - "services"
      - "secrets"
      - "configmaps"
      - "events"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
      - "watch"
EOF
```

* Apply it, attached to our development `namespace`

```bash
kubectl apply -f dev-role.yaml -n dev-$USER
```

* Create the `rolebinding` to assign the `role` to a Kubernetes to any member of the group `k8s-myprojectgroup-$USER`

```yaml
cat << EOF > dev-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: k8s-dev-role-binding-$USER
subjects:
- kind: Group
  name: k8s-myprojectgroup-$USER
roleRef:
  kind: Role
  name: k8s-dev-role-$USER
  apiGroup: rbac.authorization.k8s.io
EOF
```

* Apply the binding

```bash
kubectl apply -f dev-role-binding.yaml -n dev-$USER
```

* Asociate `k8s-dev-user` in *kubernetes* to the `k8s-dev-role` role by updating the `aws-auth` map:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

CLUSTER_ARN=$(kubectl config view --minify -o jsonpath='{.clusters[].name}')
CLUSTER_NAME=${CLUSTER_ARN#*/}
CLUSTER_NAME=${CLUSTER_NAME%%.*}
echo The name of your cluster is $CLUSTER_NAME.

eksctl create iamidentitymapping \
  --cluster $CLUSTER_NAME \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/AWSDeveloperRole${USER} \
  --group k8s-myprojectgroup-$USER \
  --username k8s-alice-user-$USER \
  --region eu-west-1
```

* Check the whole `ConfigMap`:

```bash
kubectl get configmap -n kube-system aws-auth -o yaml
```

* Retrieve all roles defined in the cluster

```bashv
eksctl get iamidentitymapping --cluster $CLUSTER_NAME
```

## Configuring developer workstation

As a developer that wants to access to the cluster and is an AWS user

Note: remember in case of messing up the configuration you can restore it with `aws eks --region eu-west-1 update-kubeconfig --name $CLUSTER_NAME`.

* Ensure the region is set correctly

```bash
export AWS_DEFAULT_REGION=eu-west-1
```

* Configure the assume role credentials (in a real-world scenario, the administrator of the AWS infrastructure should have sent them to you)

```bash
mkdir ~/.aws

cat << EOF >> ~/.aws/credentials
[AWSUserAlice${USER}]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/AWSUserAlice${USER}.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/AWSUserAlice${USER}.json)
EOF

cat << EOF >> ~/.aws/config
[profile eksDevProfile]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/AWSDeveloperRole${USER}
source_profile=AWSUserAlice${USER}
EOF
```

* Check the AWS credentials are correctly configured:

```bash
aws sts get-caller-identity --profile AWSUserAlice${USER}
aws sts get-caller-identity --profile eksDevProfile
```

* Finally, update your `.kube/config`

```bash
eksctl utils write-kubeconfig \
  --cluster $CLUSTER_NAME\
  --authenticator-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AWSDeveloperRole${USER} \
  --region eu-west-1
```

## Testing the user

* Check you have the correct context activated

```bash
kubectl config get-contexts
```

* Try (and fail) getting the nodes of the cluster or the pods in the default namespace

```bash
kubectl get nodes -n default
kubectl get pods -n default
```

* Get the pods (zero, but that's ok) in the `dev-$USER` namespace

```bash
kubectl get pods -n dev-$USER
```

## Clean it up

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

CLUSTER_ARN=$(kubectl config view --minify -o jsonpath='{.clusters[].name}')
CLUSTER_NAME=${CLUSTER_ARN#*/}
CLUSTER_NAME=${CLUSTER_NAME%%.*}
echo The name of your cluster is $CLUSTER_NAME.

rm -fr ~/.kube
eksctl utils write-kubeconfig --cluster $CLUSTER_NAME
kubectl delete namespace dev-$USER
eksctl delete iamidentitymapping \
  --cluster $CLUSTER_NAME \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/AWSDeveloperRole${USER}
aws iam remove-user-from-group --group-name AWSMyProjectGroup${USER} --user-name AWSUserAlice${USER}
aws iam delete-group-policy --group-name AWSMyProjectGroup${USER} --policy-name AssumeAwsRolePolicy${USER}
aws iam delete-group --group-name AWSMyProjectGroup${USER}
aws iam delete-access-key --user-name AWSUserAlice${USER} --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/AWSUserAlice${USER}.json)

aws iam delete-user --user-name AWSUserAlice${USER}
aws iam delete-role --role-name AWSDeveloperRole${USER}

rm  ~/.aws/{config,credentials}

```
