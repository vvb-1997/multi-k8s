kubectl create secret generic pgpassword --from-literal PGPASSWORD=postgres@123

[https://kubernetes.github.io/ingress-nginx/deploy/]

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/cloud/deploy.yaml
```

## Docker Desktop's Kubernetes Dashboard
This note is for students using Docker Desktop's built-in Kubernetes. If you are using Minikube, the setup here does not apply to you and can be skipped.

If you are using Docker Desktop's built-in Kubernetes, setting up the admin dashboard is going to take a little more work.

1. Install Helm

Following the instructions provided here, run the three following commands in your terminal:

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

2. Deploy Kubernetes Dashboard

Run the following two commands in your terminal:

```shell
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

3. Create a dash-admin-user.yaml file and paste the following:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

4. Apply the dash-admin-user configuration:

```shell
kubectl apply -f dash-admin-user.yaml
```

5. Create dash-clusterrole.yaml file and paste the following:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

6. Apply the ClusterRole configuration:

```shell
kubectl apply -f dash-clusterrole.yaml
```

7. In the terminal, run:

```shell
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

Important - You must keep this terminal window open and the proxy running!

8. Visit the following URL in your browser to access your Dashboard:
https://127.0.0.1:8443/#/login

Important - visiting localhost:8443 instead of 127.0.0.1:8443 will result in authentication failure!

9. Obtain the token

In your terminal, run the following command:

```shell
kubectl -n kubernetes-dashboard create token admin-user
```

10. Copy the token from the above output
Copy and paste the token into the login form of the dashboard. Be careful not to copy any extra spaces or output such as the trailing % you may see in your terminal.

11. After a successful login, you should now be redirected to the Kubernetes Dashboard.

The above steps can be found in the official documentation:

https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md