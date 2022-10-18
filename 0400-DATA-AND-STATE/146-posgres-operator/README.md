# Zalando Postgres operator

## Preparation

* Create the `namespace` and set it as the preferred one

```bash
kubectl create ns demo-$USER
kubectl config set-context --namespace demo-$USER --current
```

## Operator configuration

* Clone the project and check the configuration file

```bash
git clone https://github.com/zalando/postgres-operator
cd postgres-operator

cat ./charts/postgres-operator/values.yaml 
```

* Use `helm` to install the *operator*

```bash
helm install postgres-operator ./charts/postgres-operator
```

* Explore the operator `pod`
  
```bash
POD_NAME=$(kubectl get pod \
  -l app.kubernetes.io/name=postgres-operator \
  -o jsonpath="{.items[0].metadata.name}")

echo The name of the operator pod is $POD_NAME.

kubectl logs $POD_NAME
```

* Create a postgresql cluster

```yaml
cat << EOF > postgresql.yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: acid-minimal-cluster
spec:
  teamId: "acid"
  volume:
    size: 1Gi
  numberOfInstances: 2
  users:
    zalando:  # database owner
    - superuser
    - createdb
    foo_user: []  # role for application foo
  databases:
    foo: zalando  # dbname: owner
  preparedDatabases:
    bar: {}
  postgresql:
    version: "14"
EOF
```
* Deploy a default cluster

```bash
kubectl apply -f postgresql.yaml
```

* Check the most important created resources

```bash
kubectl get postgresql
kubectl get pv
kubectl get pvc
kubectl get statefulsets
kubectl get pods -l application=spilo -L spilo-role
kubectl describe pdb
```

## Accesing the Postgres cluster

* Get the cluster name

```bash
PG_CLUSTER=$(kubectl get postgresql \
  -o jsonpath='{.items..metadata.name}') && \
  echo Your cluster is $PG_CLUSTER.
```

* Take note of the main node

```bash
export PGMAIN=$(kubectl get pods \
  -o jsonpath={.items..metadata.name} \
  -l application=spilo,cluster-name=$PG_CLUSTER,spilo-role=master) && \
  echo Your master node is $PGMAIN.
```

* Access the `secret` containing the username and password of the cluster

```bash
SECRET_NAME=$(kubectl get secret \
  -o json | \
  jq -r '.items[] | select(.metadata.name | test("^postgres.*credentials")).metadata.name'\
) && echo Your cluster secret name is $SECRET_NAME.

PGUSERNAME=$(kubectl get secret $SECRET_NAME \
  -o 'jsonpath={.data.username}' | \
  base64 -d) && \
  echo Your username is $PGUSERNAME.

export PGPASSWORD=$(kubectl get secret $SECRET_NAME \
  -o 'jsonpath={.data.password}' | \
  base64 -d) && \
  echo Your super secret password is $PGPASSWORD.

```

* Stablish a port forwarding to the main pod, running in the background

```bash
PORT=$(( ( RANDOM % 1000 )  + 8000 ))
echo $PORT
kubectl port-forward $PGMAIN  --address 0.0.0.0 $PORT:5432 &
PID=$!
export PGSSLMODE=require
```

* Check connectivity to your *postgres* cluster

```bash
psql -U postgres -h localhost -p $PORT  -c 'select 1+1'
```

## Challenge: Create a database

* You can get a *pokemon* database from [this sql dump](https://pastebin.com/raw/98DaLWwG)
* Create a database in your cluster named `pokemondb`
* Dump the *pokemon* data on it
* Get all the information regarding to the row with the `identifier` `'pikachu'`
* Extra: execute that query from a pod inside your cluster

<details>
<summary>Solution:</summary>

```bash
curl https://pastebin.com/raw/98DaLWwG > pokemon.sql
psql -U postgres -h localhost -p $PORT -c 'create database pokemondb;'
psql -U postgres -h localhost -p $PORT -d pokemondb < pokemon.sql 

psql -U postgres -h localhost -p $PORT -d pokemondb -c "Select * from pokemon where identifier='pikachu'"
```
</details>

## Logical and physical backups

See this [excellent post](https://vitobotta.com/2020/02/05/postgres-kubernetes-zalando-operator/) by Vito Botta.


## Clean up

* Break the tunnel

```bash
kill -9 $PID
```

* Delete the cluster

```bash
kubectl delete -f manifests/minimal-postgres-manifest.yaml
```

* Uninstall the charts

```
helm uninstall postgres-operator
helm uninstall postgres-operator-ui
```

* Delete the namespace

```bash
kubectl delete ns demo-$USER
```
