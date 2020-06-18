# Implementing Linkerd as ServiceMesh for PMK/PMKFT

Installing Linkerd as a ServiceMesh in a Kubernetes cluster is fairly straightforward because of its architecture and great documentation.

### Prerequisites

Minimum Kubernetes version required is 1.13. This can be checked by running the command -

```bash
kubectl version --short
```


### Installing Linkerd CLI

While installing Linkerd for the first time, you need to download the Linkerd CLI. This CLI interacts with Linkerd, including installing the control plane onto your Kubernetes cluster.

Here's the command to run the CLI manually -

```bash
curl -sL https://run.linkerd.io/install | sh
```

You can also download a CLI directly from the [Linkerd releases](https://github.com/linkerd/linkerd2/releases/) page if you want to install a specific version.


Add the Linkerd to your path and also consider adding it to your .bashrc file too.

```bash
export PATH=$PATH:$HOME/.linkerd2/bin
```

If you are using Homebrew, the command to install it is as follows -

```bash
brew install linkerd
```

Once installed, you can verify the version of linkerd by running the command -

```bash
linkerd version
```

### Validating Linkerd CLI Installation

To ensure that Linkerd's CLI  has been successfully installed and all the pre-requisites  like namespace linkerd not being present etc., run the following command -

```bash
linkerd check --pre
```

All the checks in the above command should complete successfully, before proceeding.

![linkerd-checks](https://github.com/platform9/KoolKubernetes/blob/master/servicemesh/linkerd/images/linkerd_checks.png)

### Installing Linkerd's Control plane

Here's the simple command to install the Linkerd's Control plane with all the default configuration.


```bash
linkerd install | kubectl apply -f -
```

you can change configuration options like choosing a custom namespace for Linkerd Control plane by running the command -

```bash
linkerd install -l <custom_name> | kubectl apply -f -
```

For eg.

```bash
linkerd install -l linkerdtest | kubectl apply -f -
```
To explore all the available options while installing, run the command -

```bash
linkerd install --help
```

Once the installation is complete, you can run the following command to verify if all the Linkerd components are running successfully.

```bash
linkerd check
```


### Exploring Linkerd Dashboard

You can explore the Linkerd Dashboard by running the following command --

```bash
linkerd dashboard &
```

If you are running this command from a Linux machine and would like the dashboard to listen on a specific interface instead of localhost by default, run the following commmand -


```bash
linkerd dashboard --address <interface_ip> &
```
This command sets up a port forward from your local system to the linkerd-web pod.

(NOTE: If you cannot access browser from the machine you are running the above linkerd command, you can make Linkerd dashboard listening on a specific interface IP instead of localhost by default, you'll have to edit the enforced-host container argument and set it to an empty string. Detailed instructions are  [here](https://linkerd.io/2/tasks/exposing-dashboard/#dns-rebinding-protection) under *Tweaking Host Requirement* Section.


### Installing Demo Application

Install the emojivoto application in the corresponding namespace by running the command -

```bash
curl -sL https://run.linkerd.io/emojivoto.yml \
  | kubectl apply -f -
```

You can take a look at this application by using a port forward for the web-svc service associated with the emojivoto application by running the command  -

```bash
kubectl -n emojivoto port-forward svc/web-svc 8080:80
```
Then, access the application using -  http://localhost:8080/

(NOTE: If you cannot access browser from the machine you are running the above kubectl command, you can make use the --address switch so you can reach it from your local machine and access it via a browser. The complete command is -
```bash
 kubectl -n emojivoto port-forward svc/web-svc 8080:80 --address <IP_address>
 ```   


 You might notice that  that some parts of emojivoto app are broken. this is intentional and you can debug it further by checking out this [link](https://linkerd.io/2/debugging-an-app/).


### Injecting Linkerd into the Demo Application


You can inject Linkerd into the newly deployed Demo application by running the command -

```bash
kubectl get -n emojivoto deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```

The first part of the command gets all the manifests from the emojivoto namespace; second part injects it with linkerd related metadata and then reapplies the configuration.


There will be no downtime associated with this operation as it would be a rolling deploy.

You can check if the above operation was successful by running the following command -

```bash
linkerd -n emojivoto check --proxy
```

Demo app has an included load generator so you can observe the live metrics associated with the traffic flowing.

```bash
linkerd -n emojivoto stat deploy
```
### Monitoring

There's also a Grafana instance that comes prebuilt that shows metrics collected by Prometheus such as latency, request volume and success rate.


![Grafana-dashboard](https://github.com/platform9/KoolKubernetes/blob/master/servicemesh/linkerd/images/grafana.png)
