# Kubernetes training, redacted version

This is a collection of laboratories intended to help students to understand Kubernetes. They are meant to be done as part 
of the Kubernetes Training course delivered by me. Please, feel free to contact me in case you have any question regarding it.

## Course content

* Provisioning clusters
  - Cloud Kuberentes with EKS
  - Personal cluster with Kind
* Basic workloads with pods
  - Pod lifecycle
  - Health checks
  - Configuration and Config Maps
  - Configuration and Secrets
  - Affinities: nodes
  - Affinities: pods
  - Taits and tolerations
  - Pod resource usage configuration
  - Namespace resource usage configuration
* Workload management
  - DaemonSets
  - Jobs
  - CronJobs
  - Deployments and version upgrades
* Stateful workloads
  - Volumes and other basic resources
  - Working with EBS volumes on AWS
  - Working with EFS
  - StatefulSets
  - Running Posgres on Kubernetes
* Networking in depth
  - Service discovery and DNS
  - Network policies at pod level (firewall)
  - Restricting networking between namespaces
  - Ingress with Traefik
  - Ingress with Kong
* Kubernetes authorization
  - RBAC authorization
  - Governance with OPA Gatekeeper
  - Service accounts for pods
  - AWS permissions for workloads with IRSA
* Observability
  - Prometheus, Alert Manager and Grafana
  - Integration with CloudWatch Logs
* CI/CD
  - GitOps with ArgoCD
 
