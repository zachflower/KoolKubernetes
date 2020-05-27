# Step by Step instructions to deploy a GitOps Ready Stack + Demo NodeJS App
For this GitOps stack we will be leveraging a variety of tools:
 - CircleCI
 - ArgoCD
 - Platform9 Managed Kubernetes Freedom
 - Example NodeJS app to be leveraged in your demo

**CircleCI** is a modern continuous integration and continuous delivery (CI/CD) platform. You can learn more about CircleCI at: https://circleci.com/docs/2.0/about-circleci/

**ArgoCD** is a declarative, GitOps continuous delivery tool for Kubernetes. You can find more information about this project at: https://argoproj.github.io/argo-cd/

**Platform9 Managed Kubernetes Freedom** provides to you pure-play open source Kubernetes that is delivered as a SaaS managed service. The Freedom plan is a zero-cost plan that offers a capacity of up to 3 clusters and 20 nodes (or 800 vCPUs), community Slack support, and built-in critical alerting. You can sign up here: https://platform9.com/signup/

When you follow these instruction you will get something like the below flow:
![stack-overview](/images/stack-overview.jpg)

## Setup of a Platform9 Managed Kubernetes Freedom cluster
While the process is very straightforward, you can find the documentation to create a Platform9 Managed Kubernetes cluster here: https://docs.platform9.com/kubernetes/introduction/freedom-plan-faq/

## Example NodeJS app
In this setup we will be showcasing the instructions for a NodeJS application which is based upon the default react-app. This repo also contains the required files you'll use later on for CircleCI (eg. `.circleci`, `dockerfile`, etc.)
Clone the webapp01 repo on GitHub, and cd into the created directory. When you issue `npm start`, the application should open up in a browser. There are three main files: 
The index.html file is the template that will be sent to the browser, while the code for the application reside in src/App.js. In package.json you can find some parameters of the app. The most useful parameters in this file are the `version` and `name` parameters.


## CircleCI Setup
*Optionally: Sign up for a CircleCI Free plan in case you do not yet have a CircleCI account. In case you have a Free or Enterprise account feel free to leverage this one as well*
You can create a CircleCI account at https://circleci.com/signup/ where you can opt for authentication via GitHub. To make life very easy, you can give CircleCI access to work with all your repos.
The configuration of CircleCI is super intuitive and you find detailed instructions at https://circleci.com/docs/2.0/getting-started/#section=getting-started

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

### Install the Argo CD CLI on your computer.
For Mac you can use brew: 
```bash
brew tap argoproj/tap
brew install argoproj/tap/argocd
```
By default, the Argo CD API server is not exposed with an external IP. To access the API server, choose one of the following techniques to expose the Argo CD API server:

#### Option 1: Service Type Load Balancer
Change the argocd-server service type to `LoadBalancer`:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
#### Option 2: Ingress
Follow the ingress documentation on how to configure Argo CD with ingress: https://argoproj.github.io/argo-cd/operator-manual/ingress/

#### Option 3: Port Forwarding
Kubectl port-forwarding can also be used to connect to the API server
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
The API server can then be accessed using the `localhost:8080`

### Connect & Configure ArgoCD
Retrieve the password of the Argo CD API server
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
You can now logon to the Argo CD UI with the `retrieved` password and the `admin` user

![argo-login](/images/argo-login.png)

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

### *Optional: Add  your remote Kubernetes cluster(s) to ArgoCD*

#### kubectx
In the below example we're using Kubectx which is a nice addon to easily switch between clusters back and forth. You can find  [here](https://github.com/ahmetb/kubectx) more information about Kubectx

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
![argo-clusters](/images/argo-clusters.png)

![argo-status](/images/argo-status.png)

![argo-apps](/images/argo-apps.png)
