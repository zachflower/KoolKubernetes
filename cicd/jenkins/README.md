# Jenkins kubernetes deployment on Platform9 Managed Kubernetes Freedom Plan

Jenkins is a well known, widely adopted Continuous Integration platform in enterprises. 

## Deployment of Jenkins on one of your Platform9 Managed Kubernetes clusters
Here we are going to deploy Jenkins on top of platform9 managed kubernetes freedom tier followed by running a multi-branch CI pipeline to deploy a nodeJS application on kubernetes. All plugins required to deploy the sample webapp are pre-installed on the Jenkins image used by us. The Jenkins docker image can be further customized to add or remove plugins. Jenkins running on kubernetes typically requires persistent volume to save it’s configuration. This makes it possible to test Jenkins deployment on kubernetes via multiple sources like kubernetes manifests or helm charts as long as the volume persists with kubernetes cluster.
