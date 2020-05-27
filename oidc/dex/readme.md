# Integrating Dex for Identity Management with PMK/PMKFT

Dex is an open source OIDC (OpenID Connect) authentication service launched *by CoreOS*. This service provides an essential abstraction layer between other *services* (e.g. an app, microservice or a Kubernetes cluster itself) and sources *of identity such as LDAP, Google, Linkedin, etc.*

Kubernetes has native support for OIDC, which allows you to fit users and their groups directly into the existing LDAP support of Kubernetes.

### Prerequisites -

1. A working Kubernetes cluster. ( This guide assumes PMK/PMKFT cluster)
2. Dex service is going to be exposed using NodePort type. A custom /etc/hosts entry is likely needed that's pointed at one of the worker nodes. (This is the simplest approach though the right approach is to go via DNS along with ingress and similar other configurations. In my setup, I added and /etc/hosts entry for dex-pf9.example.com to point to a worker node IP on all the nodes in the cluster and from the machine where I ran kubectl)
3. Clone the Dex Repo from this URL - https://github.com/dexidp/dex to a machine from where Kubernetes master server is accessible and you  can access kubectl to access the cluster.



### Certificate Generation Step for Dex

1. Browse to the location where you've cloned the git repo -

 `$ cd <git_repo_location>/examples/k8s`

2. Edit  gencert.sh by running the command -

`$ vi gencert.sh` and edit the `[alt_names]` section to add the custom DNS. In this guide, I have chosen it to be
`dex-pf9.example.com`

3. Add the following line below the line (DNS.1 = dex-pf9.example.com)  which was edited in Step 2 above.

`IP.1 = <Worker_Node IP pointing to dex-pf9.example.com>`


4. On line 24, edit -days switch to a sufficient value so that the certificate doesn't get expired in 10 days.


5. Run the bash script gencert.sh which will generate the SSL folder in the current directory

`./gencert.sh`.



6. Browse to the ssl folder that gets created after executing Step 4 and verify that following files have been created -
```bash
$ ls
ca-key.pem	ca.pem		ca.srl		cert.pem	csr.pem		key.pem		req.cnf
```

7. Ensure that `X509v3 Subject Alternative Name` is populated with the DNS as well as IP address(es) in the cert.pem by running the command -

```bash
openssl x509 -in cert.pem -noout -text | grep -A1 'X509v3 Subject Alternative Name'
```

eg.
```bash
X509v3 Subject Alternative Name:
                DNS:dex-pf9.example.com, IP Address:1.2.3.4, IP Address:4.4.4.4
```



### Create cluster secrets

Browse to the location - `$ <git_repo_location>/dex/examples/k8s` and run the following command to create the secret that dex Pod will refer for the certificates generated in the earlier step.

```bash
$ kubectl create secret tls dex-pf9.example.com.tls --cert=ssl/cert.pem --key=ssl/key.pem
```


### Configuring the Identity Provider

In this guide, we'll configure GitHub and Google as Identity providers.


#### GitHub

1. Browse to the following URL - https://github.com/settings/applications/new

2. Create an Application with the following parameters:

![GitHub_Oauth_App](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/New_Oauthapp_GitHub.png)


```Application name  → example-app

Homepage URL → https://dex-pf9.example.com:32000

Application description → <Optional>

Authorization callback URL → https://dex-pf9.example.com:32000/callback
```

3. Once the app has been created, you'll see a screen which shows Client ID and Client Secret. Keep this handy as it will be needed later.

![Google_project](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/github_app.png)

#### GoogleAuth


1. Browse to the following URL and sign in - https://console.developers.google.com/

2. Select an existing project or create a new one if you don't have an existing project

![Google_project](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/Google_project.png)

3. After clicking on the project, select Credentials pane in the left pane

![Google_credentials](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/Credentials.png)

4. Create Credentials  by Clicking on "CREATE CREDENTIALS" in the top corner and select "OAuth Client ID"

![Create_credentials](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/create_credentials.png)

5. Select the application type as "Web Application", give it an appropriate name.

![Create_oauth](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/Create_oauth.png)

Also, add the following entries -

URIs :  https://dex-pf9.example.com:32000

Authorized redirect URIs: https://dex-pf9.example.com:32000/callback


(Please note that this is the URL where the Identity Provider(Google/GitHub) will redirect to and it should be resolvable from where you're going to access kubectl)

6. After Clicking Create, you'll be provided with a Client ID and Client Secret. Keep this handy as it will be needed later.
![Client_creation](https://github.com/platform9/KoolKubernetes/blob/master/oidc/dex/images/Client_creation.png)


### Configuring the Identity provider Secrets

In this section, we'll create Identity provider Secrets for Google and/or GitHub so Dex can authenticate on behalf of these IdPs.



#### GitHub

Paste the Client ID and Client Secret generated in the above section.

Create the secret as follows-

```bash
kubectl create secret  generic google-client \
--from-literal=client-id=<unique_code>.apps.googleusercontent.com \
--from-literal=client-secret=<redacted>
```

#### GoogleAuth

Paste the Client ID and Client Secret generated in the above section.

Create the secret as follows-

```bash
kubectl create secret  generic google-client \
--from-literal=client-id=<unique_code>.apps.googleusercontent.com \
--from-literal=client-secret=<redacted>
```


### Updating the APIServer flags


Here are the API server flags that need to be added for OIDC authentication to work.

(NOTE: This procedure is necessary for all the master servers in the cluster)

SSH to the master server.

Browse to the location -

`cd /opt/pf9/pf9-kube/conf/masterconfig/base`

1. copy the CA certificate of Dex on each master that was created in the earlier steps. It's present at the location - <Dex_location>/dex/examples/k8s/ssl and the file name is `ca.pem`.  Save this CA file at - `/etc/pf9/kube.d/authn` ( Keep an additional copy of this on each masters as well)

Edit the master.yaml and add the following lines under the section command for the apiserver container.

```bash
"--oidc-issuer-url=https://<FQDN_selected>:32000"
"--oidc-client-id=example-app"
"--oidc-ca-file=/srv/kubernetes/authn/CA_dex.crt"
```


If you are following this example, it would be -

```bash
"--oidc-issuer-url=https://dex-pf9.example.com:32000"
"--oidc-client-id=example-app"
"--oidc-ca-file=/srv/kubernetes/authn/CA_dex.crt"
```


2. Stop the pf9-kube process and start it using the following commands on the master servers as it will restart the api-server component needed for the above flags to take effect.  

```bash
/etc/init.d/pf9-kube stop
```

Let the above command complete, and then issue the following command -
```bash
/etc/init.d/pf9-kube start
```


### Deploying Dex on Kubernetes

Reference the yaml files provided for GitHub and Google Auth connectors in the yaml folder.

1. Browse to the location - `<Dex_location>/examples/k8s`

2. Few changes that are necessary in the dex.yaml for it to be effective in your environment -

(Line 58) issuer: This needs to be set to the FQDN you have selected for the Dex. In this guide, its https://dex-pf9.example.com:32000

(Line 74) redirectURI: This needs to be set to the <FQDN>/callback. This should be equal to what's set in the IDP configuration in the earlier section. In this guide, its https://dex-pf9.example.com:32000/callback


```bash
kubectl apply -f dex.yaml
```

3. Ensure that Dex pods is in a Running state before proceeding further in the guide.




### Using KubeLogin as OIDC Plugin for kubectl


On a Linux machine, run the following command to install kubelogin binary.
```bash
curl -LO https://github.com/int128/kubelogin/releases/download/v1.19.0/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
ln -s kubelogin kubectl-oidc_login
 ```

On a Mac OS
```bash
# Homebrew
brew install int128/kubelogin/kubelogin
```
```bash
# Krew
kubectl krew install oidc-login
```

I’ll be proceeding with the Linux Machine example going forward.

Run the following command to ensure that OIDC authentication via KeyCloak is successful.  

```bash
./kubectl-oidc_login setup   --oidc-issuer-url=https://<keycloak-interface_IP>:8443/auth/realms/<realm_name>   --oidc-client-id=<client_ID>   --oidc-client-secret=<secret noted earlier in Keycloak>--insecure-skip-tls-verify --grant-type password
```

If you are following the values in this example, the above command would look like

```bash
./kubectl-oidc_login setup   --oidc-issuer-url=https://<keycloak-interface_IP>:8443/auth/realms/master   --oidc-client-id=kubernetes   --oidc-client-secret=<secret noted earlier in Keycloak>--insecure-skip-tls-verify --grant-type password
```

You would get a set of instructions once the authentication is successful which includes Creating a cluster role, setting up API server flags, etc.  Below is an example -

```bash
## 2. Verify authentication

You got a token with the following claims:

{
  "iss": "https://dex-pf9.example.com:32000",
  "sub": "CggzNzYwNTc1MxIGZ2l0aHVi",
  "aud": "example-app",
  "exp": 1590658246,
  "iat": 1590571846,
  "nonce": "phteNh1LZMv6KRYvpTgzan7QYtp_M_Tp1LKmNLaBjuA",
  "at_hash": "NTzX9ZjOLkbtRp9_-wqDAw"
}

## 3. Bind a cluster role

Run the following command:

	kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user='https://dex-pf9.example.com:32000#CggzNzYwNTc1MxIGZ2l0aHVi'

## 4. Set up the Kubernetes API server

Add the following options to the kube-apiserver:

	--oidc-issuer-url=https://dex-pf9.example.com:32000
	--oidc-client-id=example-app

## 5. Set up the kubeconfig

Run the following command:

	kubectl config set-credentials oidc \
	  --exec-api-version=client.authentication.k8s.io/v1beta1 \
	  --exec-command=kubectl \
	  --exec-arg=oidc-login \
	  --exec-arg=get-token \
	  --exec-arg=--oidc-issuer-url=https://dex-pf9.example.com:32000 \
	  --exec-arg=--oidc-client-id=example-app \
	  --exec-arg=--oidc-client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0 \
	  --exec-arg=--insecure-skip-tls-verify \

## 6. Verify cluster access

Make sure you can access the Kubernetes cluster.

	kubectl --user=oidc get nodes

You can switch the default context to oidc.

	kubectl config set-context --current --user=oidc

You can share the kubeconfig to your team members for on-boarding.
```

Let’s create a cluster role binding as per the instructions you get from the above command.  Here’s an example for a cluster-admin  Cluster role binding

```bash
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: oidc-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: User
  name: 'https://dex-pf9.example.com:32000#CggzNzYwNTc1MxIGZ2l0aHVi'
```

Lastly, set up the kubeconfig file to reflect the following values -

  a. Set user as oidc for the default context

  b. Set the oidc user under the users section in the kubeconfig to reflect the following values.


For Linux Users, kubeconfig should look like the below configuration and edit the relevant context to reflect the oidc user.
```bash
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - get-token
      - --oidc-issuer-url=https://dex-pf9.example.com:32000
      - --oidc-client-id=<oidc_client_id>
      - --oidc-client-secret=<secret_noted_earlier>
      - --insecure-skip-tls-verify
      - --grant-type=password
      command: <path_to_kubelogin>/kubelogin
      env: null
```
if you are following the above example, kubeconfig should look like this


```bash
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - get-token
      - --oidc-issuer-url=https://dex-pf9.example.com:32000
      - --oidc-client-id=example-app
      - --oidc-client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0
      - --insecure-skip-tls-verify
      - --grant-type=password
      command: <path_to_kubelogin>/kubelogin
      env: null
```

For Mac Users, kubeconfig should look like the below configuration and edit the relevant context to reflect the oidc user.
```bash
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://dex-pf9.example.com:32000
      - --oidc-client-id=example-app
      - --oidc-client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0
      - --insecure-skip-tls-verify
      - --grant-type=password
      command: kubectl
      env: null
```


if you are following the above example, kubeconfig should look like this


```bash
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://dex-pf9.example.com:32000
      - --oidc-client-id=example-app
      - --oidc-client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0
      - --insecure-skip-tls-verify
      - --grant-type=password
      command: kubectl
      env: null
```



### References:

`https://medium.com/@int128/kubectl-with-openid-connect-43120b451672`

`https://medium.com/@mrbobbytables/kubernetes-day-2-operations-authn-authz-with-oidc-and-a-little-help-from-keycloak-de4ea1bdbbe`

`https://github.com/dexidp/dex/blob/master/Documentation/connectors/oidc.md`
