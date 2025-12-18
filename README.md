kubectl create secret generic pgpassword --from-literal PGPASSWORD=postgres@123

[https://kubernetes.github.io/ingress-nginx/deploy/]

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/cloud/deploy.yaml
```

## Docker Desktop's Kubernetes Dashboard
[README](docs/docker_desktop_k9s_dashboard.md)

## Setup GCP GITHUB
[README](docs/setup_gcp_github.md)

## Setup GCP k8s Cluster
[README](docs/setup_gcp_k8s_cluster.md)