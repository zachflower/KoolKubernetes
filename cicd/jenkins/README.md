# Jenkins kubernetes deployment on Platform9 Managed Kubernetes Freedom Plan

Jenkins is a well known, widely adopted Continuous Integration platform in enterprises. 

## Deployment of Jenkins on one of your Platform9 Managed Kubernetes cluster
Here we are going to deploy Jenkins on top of platform9 managed kubernetes freedom tier. The Jenkins docker image provided with the deployment manifest has Openjdk8, Maven, Go and NodeJS preinstalled with some commonly used plugins. The Jenkins docker image provided here can be further customized once Jenkins is up and running.

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
The deployment creates a persistent volume with node affinity to schedule the jenkins pod on the node with label set to 'jenkins'. This makes sure Jenkins pod gets scheduled on the same node each time so that configuration in Jenkins home directory is automatically persisted across jenkins pod restarts.

At the end of deployment a service type loadbalancer is created and Jenkins can be accessed via http://Load-Balancer-IP:8080

## Store Dockerhub and GitHub credentials
Storing DockerHub username and password of your DockerHub repository into Jenkins is required by pipeline to upload images to your repository. Also store your GitHub credentials in Jenkins as it helps pulling the KoolKuberenetes repository instantly.
On Jenkins UI on the home page click Credentials -> Jenkins -> Global credentials (unrestricted) -> Add credentials. Fill out the username, password and ID.

![add-cred-dhub](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/add_cred_dhub.png)

Similarly add credential for GitHub account. Once both dockerhub and GitHub credentials are in place they can be seen as below:

![add-cred](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/add_cred.png)


## Bring your DockerHub registry to publish image
Set your dockerhub registry location as environment variable by clicking Jenkins -> Manage Jenkins -> Configure System. Scroll down to Global Properties. Tick 'Environment Variables'. Set the variable as shown in the image below. This allows the pipeline to publish the image to your dockerhub repository mentioned in the environment variable. Use any pre-existing dockerhub repository or create a new one on hub.docker.com.

![add-env](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/dhub_loc.png)

## Run Pipeline
Now we are all set to configure pipeline to run on Jenkins. Create a multi branch pipeline by clicking on ‘New Item’ on the home page. Provide some name to this pipeline and select ‘Multibranch Pipeline’ from the available list of options. Click OK to move to next page. 

![create](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/create.png)

On this page under 'Branch Source' click Add source. Set 'GitHub' in credentials from drop-down menu. Add the full path of the KooklKubernetes GitHub repository in 'Repository HTTPS URL'. Clicking 'Validate' button will verify access on the repo.

![source](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/source.png)

Further scroll down on the page to 'Build Configuration' and set the script path to 'cicd/jenkins/webapp01/Jenkinsfile'. Finally press Save at the bottom to finish configuring. 

![jenkinsfile](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/jenkinsfile_path.png)

The Pipeline will execute immediately after you click 'Save'.  

![p-start](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/p_start.png)


It will move through stages namely checkout SCM, build, publish and deploy. Once the build stage is successful, the publish stage will create a docker image of the NodeJS application and push it to your Dockerhub repository. Finally the deploy stage will use the kubernetes plugin to pull the newly created image from your dockerhub repository and deploy it on the node with label 'jenkins'.

![p-end](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/p_finish.png)

The NodeJS application by now should be accessible outside via the load balancer IP.

```bash
$ kubectl get svc p9-react-app
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
p9-react-app   LoadBalancer   10.21.245.88   10.128.231.207   80:31298/TCP   13m
```


![nodejs-app](https://github.com/platform9/KoolKubernetes/blob/master/cicd/jenkins/images/nodejs-app.png)


