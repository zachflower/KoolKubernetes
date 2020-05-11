# Jenkins kubernetes deployment on Platform9 Managed Kubernetes Freedom Plan

Jenkins is a well known, widely adopted Continuous Integration platform in enterprises. 

## Deployment of Jenkins on one of your Platform9 Managed Kubernetes cluster
Here we are going to deploy Jenkins on top of platform9 managed kubernetes freedom tier. The Jenkins docker image provided with the deployment manifest has Openjdk8, Maven, Go and NodeJS preinstalled with some commonly used plugins. The Jenkins docker image can be further customized to add or remove plugins.

Jenkins running on kubernetes typically requires persistent volume to maintain it’s configuration across restarts.

## Jenkins configuration
Before deploying Jenkins we will label one node with a specific key value pair so that Jenkins pod gets scheduled on this node. 

Select the node with enough resources for Jenkins to run on. Label it in the following manner. 

```bash
$ kubectl label nodes <node-name> jenkins=allow
```
Now clone the Kool Kubernetes repository on any machine from where the kubectl can deploy json manifests on kubernetes cluster.
```bash
$ git clone https://github.com/platform9/KoolKubernetes.git
```
Deploy Jenkins using kubectl
```bash
$ kubectl apply -f KoolKubernetes/cicd/jenkins/ci/jenkins.yaml
```
The deployment creates a persistent volume with node affinity to schedule the claiming pod on the node with label set as jenkins. This makes sure Jenkins pod gets scheduled on the same node each time so that configuration in Jenkins home directory is automatically persisted. 

At the end of deployment a service type loadbalancer is created and Jenkins can be accessed via http://Load-Balancer-IP:8080

## Store Dockerhub credentials
In Jenkins UI store DockerHub username and password to allow Jenkins to upload the images on DockerHub. In Jenkins Home click Credentials -> Jenkins -> Global credentials (unrestricted) -> Add credentials. Fill out the username and password and ID.

![add-cred](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/add-cred.png)

## Run Pipeline
Create a multi branch pipeline by clicking on ‘New Item’ on the home page. Provide the name for the pipeline and select ‘Multibranch Pipeline’ from the available list of options. Click OK. 

![creae](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/create.png)

In Branch Source click Add source. Add the full path of the GitHub repository for Jenkins to checkout the code. Here we are using https://github.com/p9sys/webapp01.git. Click Save.

![source](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/source.png)

The Pipeline will proceed through stages namely checkout SCM, build, publish and deploy. Once the build stage is successful, the publish stage will create a docker image of the NodeJS application and push it to Dockerhub. Finally the deploy stage will use the kubernetes plugin to deploy the application image on the node. 

![run](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/run.png)

The NodeJS application by now should be accessible outside via a load balancer IP.

```bash
$ kubectl get svc p9-react-app
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
p9-react-app   LoadBalancer   10.21.245.88   10.128.231.207   80:31298/TCP   13m
```

![nodejs-app](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/nodejs-app.png)

