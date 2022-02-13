## External services

[Draft]

## Types of services

* A service can be exposed in **three different ways**
* **ClusterIP** is intended for **internal use only**
* **NodePort** will publish a **meshed port** on every worker node
  - Not ideal from the security point of view because lack of granalarity controlling them
  - Will not expose port 80, 443, or actually anything bellow 30000 and your firewall will not be happy
* **LoadBalancer** will associate a native L4 proxy to the service
  - On AWS it will be associated to an ELB ($20 per month)
  - Annotations can be use to further configure the actual resource (for example, installing a certificate)



* URLs placed outside the cluster can be accessed using services as service discovery abstraction
* The definition is quite straightforward as shown in [httpbin.yaml](httpbin.yaml)

```yaml
cat << EOF > external-yaml
kind: Service
apiVersion: v1
metadata:
 name: hb
spec:
 type: ExternalName
 externalName: httpbin.org
EOF
```

* Create the service

```bash
kubectl create ns demo-$USER
kubectl apply -f external-yaml -n demo-$USER
```

* Launch a pod and access *httpbin* through the service

```bash
kubectl run -it --image bash -n demo-$USER mybash

bash# wget http://hb/get -O- --header=Content-Type:application/json --header=Host:httpbin.org -q
bash# exit
```

* Delete the resources

```
kubectl delete ns demo-$USER
```

