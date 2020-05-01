

# Integrating KeyCloak for Identity Management with PMK/PMKFT

### Prerequisites - 

Keycloak running within a VM. 

You can deploy a Keycloak server using these set of instructions. 

Deploy a CentOS 7.x machine and install Java SDK8 as its a prerequisite for KeyCloak - 
```bash
yum -y install java-1.8.0-openjdk
yum -y install wget
```
Download the Keycloak server binary by running the following command - 
```bash
cd /opt
wget https://downloads.jboss.org/keycloak/9.0.3/keycloak-9.0.3.tar.gz
```
Extract the archive and browse to that directory `<KeyCloak_directory>/standalone/configuration`
```tar -xvf ./keycloak-9.0.3.tar.gz ```

KeyCloak can be implemented in a standalone mode for testing. For production use-cases, a Clustered KeyCloak implementation is recommended.  
 


### CERTIFICATE GENERATION STEPS

Step 1 is common for both the self-signed certs as well as for the third-party CA certificate

SSH to the Keycloak VM 

1. Run the following command to create a Keystore - 

```bash 
keytool -genkeypair -keystore keystore.jks -dname "CN=localhost, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown" -keypass keystore -storepass keystore -keyalg RSA -alias localhost -ext SAN=ip:<IP_address_of Keycloak instance>
```
(IP address should have IP of at least one interface, can have multiple IP:<ip_address pairs>)    

Explanation of the arguments in the above command -  
alias - it’s the alias of the Java Keystore

dname - The Distinguished Name from the X.500 standard. This name will be associated with the alias for this key pair in the 

KeyStore - The name is also used as the "issuer" and "subject" fields in the self-signed certificate

validity - in days

keyalg- The name of the algorithm used to generate the key. 

keypass - The key pair password needed to access this specific key pair within the KeyStore.

storepass - The password for the whole KeyStore. Anyone who wants to open this KeyStore later will need this password


( KeyCloak Reference - https://www.keycloak.org/docs/latest/server_installation/#enabling-ssl-https-for-the-keycloak-server   KeyTool Reference - http://tutorials.jenkov.com/java-cryptography/keytool.html#keytool-arguments) 



### Self-Signed Certificate

2. Extract the certificate from the Keystore generated in the above command ( This will be passed to the  kube-apiserver in the later steps) 
       

```bash
keytool -export -alias localhost -keystore keystore.jks -rfc -file public.cert
```

3. Add the generated KeyStore to KeyCloak Configuration
   Copy the keystore.jks to the location `<KeyCloak_directory>/standalone/configuration`
NOTE:  This should already be done if you executed the above commands in the configuration dir.

Edit the standalone.xml or standalone-ha.xml depending on the type of the installation. Edit the SSL section under the security-realm sections. 

By default, the server-identities section looks like this - 

`
<server-identities>
                    <ssl>
                        <keystore path="application.keystore" relative-to="jboss.server.config.dir" keystore-password="password" alias="server" key-password="password" generate-self-signed-certificate-host="localhost"/>
                    </ssl> 
</server-identities>
`

Please change the default section to reflect the keystore.jks generated along with the password selected. 

`
<server-identities>
                <ssl>
                    <keystore path="keystore.jks" relative-to="jboss.server.config.dir" keystore-password="keystore" alias="unknown" key-password="keystore"/>
                </ssl>
</server-identities>
`
 

Update the following values depending on the values set above - 
keystore path = *name of the Java keystore in the above command* 

keystore-password = *password selected earlier*

alias= *alias selected earlier*

key-password =  *key password selected earlier*


### Third-Party CA Certificate (Optional if using Self-Signed Certificates)
 

 1. Run the following command to generate a CSR -  
```bash
keytool -certreq -alias unknown -keystore keystore.jks -ext SAN=ip:<ip_address>  > keycloak.careq
```

2. Get the CSR signed with the third-party CA and ensure you have a copy of the CA certificate as well. 

3. Add the Signed Certificate and the CA certificate to the KeyStore 

    i. Copy the signed certificate and the CA certificate to the same folder where the Keystore was generated in the first sub-section. 
    
    ii. Run the following command to import the CA certificate in the keystore - 
```bash
keytool -import -keystore keystore.jks -file <CA.crt>  -alias CAroot
```

   iii. Run the following command to import the signed certificate in the keystore 

```bash
keytool -import -alias unknown  -keystore keystore.jks -file <signed.crt>
```

( alias has to be the same that was earlier used in the first section) 


4.  Add the generated KeyStore to KeyCloak Configuration
  
  Copy the keystore.jks to the location `keycloak_download_location/standalone/configuration`


Edit the standalone.xml (standalone-mode)  or standalone-ha.xml (clustered mode) depending on the type of the installation. Edit the SSL section under the security-realm sections. 

`
      <server-identities>
                <ssl>
                    <keystore path="keystore.jks" relative-to="jboss.server.config.dir" keystore-password="keystore" alias="unknown" key-password="keystore" />
                </ssl>
            </server-identities>
`

Update the following values depending on the values set above - 

keystore path = *name of the Java keystore*. 

keystore-password= *password selecter earlier*

  alias= *alias selected earlier*

key-password= *key password selected earlier*

### Configuring  and Starting the KeyCloak server 
 

1. Add an admin user by the browsing to the location 
 `cd keycloak_download_location/bin` and running the command - 
```bash
./add-user-keycloak.sh -u pf9 
```

2. Start the KeyCloak server from the same location  -  
```bash
./standalone.sh  -b <interface_ip> &
```

3. Ensure that you are able to reach the KeyCloak server on the  interface ( or floating IP)

`https://interface_ip:8443`

 

4. Login to the management console using the pf9 user credentials created earlier.  Create a client  by Clicking on the Clients tab in the right pane  ( You can also choose to create a new realm and then create a client )

![Keycloak-client](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/keycloak-clients.png)

5. After creating the Client, configure the client by clicking on it and set the following properties as seen in the screenshot 
```Enabled → On

Client-Protocol → openid-connect

Access-Type → Confidential

Standard Flow Enabled → Enabled

Implicit Flow Enabled  → Disabled

Direct Access Grants → Enabled

Service Accounts → Enabled

Authorization → Enabled

Redirect_URI : urn:ietf:wg:oauth:2.0:oob
```
![Keycloak_clients_2](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/Keycloak_clients_2.png)


6. Go to the Credentials sub-tab and ensure that Secret value is populated. If not, click on Regenerate Secret and store it for further use. 

7. Click on Mappers sub-tab and create the following mappings. 

    a. email 

```Name: email 

Mapper Type: User Property

Property: email

Token Claim Name: email 

Claim JSON Type: String

Add to ID Token: Enabled

Add to access token: Enabled

Add to userinfo: Enabled
```
 
![mapper_email](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/email_mapper.png)
    
   b.  username


```Name: username 

Mapper Type: User Property

Property: username

Token Claim Name: preferred_username 

Claim JSON Type: String

Add to ID Token: Enabled

Add to access token: Enabled

Add to userinfo: Enabled
```
 

![mapper_username](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/username_mapper.png)

 

c. groups 

```Name: groups 

 Mapper Type: Group membership

Property: groups

Token Claim Name: groups 

Claim JSON Type: String

Add to ID Token: Enabled

 Add to access token: Enabled

Add to userinfo: Enabled

Full Group Path: Enabled
```
 
![mapper_groups](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/groups_mapper.png)



 

8. Using these mappers, attributes like Groups ( that would be mapped to cluster-role bindings in K8s RBAC) and the username and email specified during login will be verified. 

Create a group by Clicking Groups under the Manage section in the right pane. Name the group as cluster-admins. 

 
![group_creation](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/group_creation.png)

 

9. Add the pf9 user created earlier to this group by Clicking Users under the Manage section in the right pane. Click on the admin user and go to Groups section, select View all groups, then select Cluster admins and click on Join 


![pf9_user](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/pf9_user.png)

Also, ensure that email field for the pf9 user has been populated and the email_verified flag has been set. 


![email_verified](https://github.com/platform9/KoolKubernetes/blob/master/oidc/keycloak/images/email_verified.png)



 ( NOTE: You can repeat group creation steps where you create an additional group. Create an additional user and attach that user to the respective group.)   


### Changing API server flags ( For PMKFT and PMK clusters) 
 

Here are the API server flags that need to be added for OIDC authentication to work. 

(NOTE:  This procedure is necessary for all the master servers in the cluster) 

SSH to the master server. 

 Browse to the location - 

`cd /opt/pf9/pf9-kube/conf/masterconfig/base`

  1. copy the CA certificate (public.crt generated in the Cert generation step in case of Self-Signed cert) on each master and save it at - /etc/pf9/kube.d/authn ( Keep an additional copy of this on each masters as well)

  2.  Edit the master.yaml and add the following lines under the section command for the apiserver container. 
```bash
- "--oidc-issuer-url=https://<keycloak-interface_IP>:8443/auth/realms/<realm_name>"
- "--oidc-client-id=<Client Name created in keycloak>"
- "--oidc-ca-file=/srv/kubernetes/authn/<CAcert_crt>"
- "--oidc-username-claim=email"
- "--oidc-username-prefix=oidc:"
- "--oidc-groups-claim=groups"
- "--oidc-groups-prefix=oidc:"
```
In the case of Self-signed certs,  `-oidc-ca-file` would the public.crt created in the previous steps. 

 

3. Stop the pf9-kube process and start it using the following commands on the master servers as it will restart the api-server component needed for the above flags to take effect.  
```bash
/etc/init.d/pf9-kube stop 
```
Let the above command complete, and then issue the following command - 
```bash
/etc/init.d/pf9-kube start
```
 


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
You would get a set of instructions once the authentication is successful which includes Creating a cluster role, setting up API server flags, etc. 

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
- kind: Group
  name: oidc:/cluster-admins
```
( You can create additional ClusterRolebindings  associated with individual groups and replace the   subject: name field in the  above YAML) 

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
      - --oidc-issuer-url=https://<keycloak-interface_IP>:8443/auth/realms/<realm_name>
      - --oidc-client-id=<oidc_client_id>
      - --oidc-client-secret=<secret_noted_earlier>
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
      - --oidc-issuer-url=https://<keycloak-interface_IP>:8443/auth/realms/<realm_name>
      - --oidc-client-id=<oidc_client_id>
      - --oidc-client-secret=<secret_noted_earlier>
      - --insecure-skip-tls-verify
      - --grant-type=password
      command: kubectl
      env: null
```


 
### References: 

`https://medium.com/@int128/kubectl-with-openid-connect-43120b451672`

`https://medium.com/@mrbobbytables/kubernetes-day-2-operations-authn-authz-with-oidc-and-a-little-help-from-keycloak-de4ea1bdbbe`

 
