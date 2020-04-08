# Deployment of ArgoCD on Platform9 Managed Kubernetes Freedom Plan
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. You can find more information about this project at: https://argoproj.github.io/argo-cd/

## Deployment of ArgoCD on one of your Platform9 Managed Kubernetes clusters
*Optionally: create a dedicated namespace for ArgoCD*
```bash
kubectl create namespace argocd
```
`
namespace/argocd created
`
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
`
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-secret created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server-metrics created
service/argocd-server created
deployment.apps/argocd-application-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
`

## Install the Argo CD CLI on your computer.
For Mac you can use brew: 
```bash
brew tap argoproj/tap
brew install argoproj/tap/argocd
```
By default, the Argo CD API server is not exposed with an external IP. To access the API server, choose one of the following techniques to expose the Argo CD API server:

### 1. Service Type Load Balancer¶
Change the argocd-server service type to `LoadBalancer`:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
### 2. Ingress
Follow the ingress documentation on how to configure Argo CD with ingress: https://argoproj.github.io/argo-cd/operator-manual/ingress/

### 3. Port Forwarding
Kubectl port-forwarding can also be used to connect to the API server
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
The API server can then be accessed using the `localhost:8080`

## Connect & Configure ArgoCD
Retrieve the password of the Argo CD API server
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
You can now logon to the Argo CD UI with the `retrieved` password and the `admin` user

![argo-login](https://github.com/platform9/pmk-k8-yaml/blob/master/cicd/argo/images/argo-login.png)

You can change the auto-generated password using the argocd CLI
```bash
argocd login 127.0.0.1:8080
```
`
WARNING: server certificate had error: x509: cannot validate certificate for 127.0.0.1 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin' logged in successfully
Context '127.0.0.1:8080' updated
`
```bash
argocd account update-password   
```
```
Enter current password:
Enter new password:
Confirm new password:
Password updated      
```

## *Optional: Add  your remote Kubernetes cluster(s) to ArgoCD*

List all cluster contexts that are currently loaded
```
kubectx
```
```
cicd
edg01
edg02
```
```bash
argocd cluster add edg01
argocd cluster add edg02
```

```
INFO[0001] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0001] ClusterRole "argocd-manager-role" created
INFO[0001] ClusterRoleBinding "argocd-manager-role-binding" created
Cluster 'https://<<IP API Server>>' added
```

## Result
![argo-clusters](https://github.com/platform9/pmk-k8-yaml/blob/master/cicd/argo/images/argo-clusters.png)

![argo-login](https://github.com/platform9/pmk-k8-yaml/blob/master/cicd/argo/images/argo-login.png)

![argo-status](https://github.com/platform9/pmk-k8-yaml/blob/master/cicd/argo/images/argo-status.png)

![argo-apps](https://github.com/platform9/pmk-k8-yaml/blob/master/cicd/argo/images/argo-apps.png)
