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

