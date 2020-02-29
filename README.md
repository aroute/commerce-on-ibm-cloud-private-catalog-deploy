# archive-commerce-on-ibm-cloud-private-catalog-deploy
Commerce on IBM Cloud Private - Catalog deploy

## Hands-on Recipe: Deploy IBM Cloud Private and Commerce v9 via Catalog.

**Summary**

This hands-on recipe will walk you through deploying Commerce 9 (WCSv9) on IBM Cloud Private (ICP). This deployment of WCSv9 is based on ICP's application catalog's level 2 certification. 

You will learn to deploy ICP and WebSphere Commerce store and tools. 

The instructions require a general familiarity with Linux, Docker, Kubernetes and WebSphere Commerce. 

**Content**

- Phase 1: In phase one, you will be setting up and deploying IBM Cloud Private (ICP) 3.1.0 CE
- Phase 2: In phase two, you will be setting up and deploying WebSphere Commerce 9.0.1.3 via ICP's catalog.

**Network Topology & Security**

- IBM's Hybrid cloud/Kubernetes implementation - Cloud Private (ICP) - is suitable for large scale cluster of networked resources. 
- This recipe uses a 32GB Ubuntu 16.04.5 LTS single-node ICP server which acts as an all-in-one master, worker and proxy node. 
- The recipe uses private/local IP address scheme as defined in RFC1918. This network topology could be expanded and enhanced with separate worker nodes and multiple master nodes for High Availability (only available in ICP Enterprise Edition) in multi-zone data-centers across the globe. 
- Since the server is confined locally with NAT routing, you will be using relaxed password scheme (this is not possible with public IP address and/or production environment). However, to simulate security's best practices, a local account (called 'demo') has been setup with sudo privileges for elevated execution of the commands, where needed.
- A desktop-based Ubuntu machine with small footprint has been provided for you to access ICP's dashboard via Firefox browser. You will be using this desktop to deploy WebSphere Commerce via ICP's catalog.
- WebSphere Commerce's website is setup with locally resolvable domain name (configured via host file for DNS resolution). The configured Helm Charts have been designed to adjust with any domain name, as long as the domain names are configured to resolve either via public DNS, or local host file. This write-up uses the demo URL of *.ibm.com domain in an isolated network using the alternate method of local host file resolution.

**Preparation**

- Watch proof of technology video (see below). It is a full video demonstration of the complete deployment.
When ready, access your Skytap Environment.
- From the Automation section of your Skytap's Environment, click on the Power options. Change the default 2 hours to 8 hours (you will have to do this again if you take more than one day to finish this course).
- Turn ON both of your machines at once.
- From your laptop or desktop, use Terminal of your choice. If you are on MAC, use the default terminal. If you are on Windows, use Putty, or Git Bash (install Git for Windows), or any other terminal of your choice. The course literature has been prepared on Mac using the default terminal. Adjust as necessary.
- There are some multi-line commands in the recipe. Copy/paste multi-line commands for convenience.


## Phase 1: Installation/Setup of IBM Cloud Private (ICP) 3.1.0

**On your desktop/laptop, using Terminal**

From this point forward, you will be connecting to your Skytap server from your own laptop or desktop, using the terminal of your choice. 

### 1.1 Login to server via terminal

**Explanation:**

Since Linux servers generally do not come with desktop environment; the common practice is to use Secure Shell (SSH) for the authentication. In production environments, you are expected to use private/public key pairs for the most secure connection. In this course, you will be using the combination of keys and passwords.  

The ServerAliveInterval and ServerAliveCountMax tells the server to not disconnect idle connection for certain period of time. This is optional. You can directly ssh demo@.... if you prefer.

```sh
# Use your Skytap's SSH URL and port number.
MacBook-Pro:~ username$ ssh -o "ServerAliveInterval 60" -o "ServerAliveCountMax 120" demo@services-uscentral.skytap.com -p 9019 
MacBook-Pro:~ username$ yes
demo@services-uscentral.skytap.com's password: passw0rd
demo@master:~$
```
#### 1.2 Install ICP related Utilities

**Explanation:**

There are two utilities scripts prepared in advance. Run them one after another. The first script (01utilities.sh) will install socat and python. The second script (02setupdocker.sh) is self-explanatory. 

```sh
demo@master:~$ ./01utilities.sh
```

When prompted for a password, type passw0rd.

```sh
demo@master:~$ ./02setupdocker.sh
```

Type exit to logout and log back in.

```sh
# Use your Skytap's SSH URL and port number.
MacBook-Pro:~ username$ ssh -o "ServerAliveInterval 60" -o "ServerAliveCountMax 120" demo@services-uscentral.skytap.com -p 9019 
MacBook-Pro:~ username$ yes
demo@services-uscentral.skytap.com's password: passw0rd
demo@master:~$
```

#### 1.3 Setup SSH key/pairs

**Explanation:**

ICP requires ssh communication between all users, including root. The following sets of commands sets up password-less log-in between demo user and root user. You are also setting up exchange of public keys between the two accounts. In multiple-node and production-like setup, you will exchange public keys between all nodes. 

#### 1.3.1 SSH Keys for demo user

```sh
demo@master:~$ ssh-keygen -t rsa -P ""
Enter file in which to save the key (/root/.ssh/id_rsa):
demo@master:~$ (hit Enter)
Your identification has been saved in /demo/.ssh/id_rsa.
demo@master:~$ cd .ssh
demo@master:~/.ssh$ cat id_rsa.pub >> authorized_keys
demo@master:~/.ssh$ sudo service ssh restart
[sudo] password for demo: passw0rd
demo@master:~/.ssh$ ssh localhost
Are you sure you want to continue connecting (yes/no)? yes
demo@master:~$ exit
demo@master:~/.ssh$ ssh master
Are you sure you want to continue connecting (yes/no)? yes
demo@master:~$ exit
demo@master:~/.ssh$ ssh 172.16.1.1
demo@master:~$ exit
demo@master:~/.ssh$ cd ..
```

#### 1.3.2 SSH Keys for root user

```sh
demo@master:~$ sudo su -
root@master:~# ssh-keygen -t rsa -P ""
Enter file in which to save the key (/root/.ssh/id_rsa):
root@master:~# (hit Enter)
Created directory '/root/.ssh'.
Your identification has been saved in /root/.ssh/id_rsa.
root@master:~# cd .ssh
root@master:~/.ssh# cat id_rsa.pub >> authorized_keys
root@master:~/.ssh# service ssh restart
root@master:~/.ssh# ssh localhost
Are you sure you want to continue connecting (yes/no)? yes
root@master:~# exit
root@master:~/.ssh# ssh master
Are you sure you want to continue connecting (yes/no)? yes
root@master:~# exit
root@master:~/.ssh# ssh 172.16.1.1
root@master:~# exit
root@master:~/.ssh$ cd ..
```

#### 1.3.3 User root: Copy public keys to demo user

```sh
root@master:~# ssh-copy-id demo@localhost
root@localhost's password: passw0rd
root@master:~# ssh demo@localhost
demo@master:~$ exit
root@master:~# exit
```

#### 1.3.4 User demo: Copy public keys to root user

```sh
demo@master:~$ ssh-copy-id root@localhost
root@localhost's password: passw0rd
demo@master:~/.ssh$ ssh root@localhost
root@master:~# exit
```

### 1.4 Setup ICP for install

**Explanation:**

You will be installing IBM Cloud Private Community Edition 3.1.0 by pulling its docker images from public Docker Hub. The setup of ICP is based on Ansible scripts. The setup collects the inventory of the hosts (in this case, there's only one host - master - which acts as master, worker and proxy simultaneously). The 'tee' command inserts the content of the file into the host file. The setup requires private key of the user with root access. And, it takes about 30 minutes to finish.  It is recommended that you prepare the installation environment from your command line terminal but run the installation script from the Skytap's Web Console Terminal. This will help free up your local terminal while the installation will run uninterrupted in Web Console. 

#### 1.4.1 Prepare ICP installation environment

```sh
demo@master:~$ sudo mkdir /opt/ibm-cloud-private-ce-3.1.0 && cd /opt/ibm-cloud-private-ce-3.1.0
demo@master:/opt/ibm-cloud-private-ce-3.1.0$ docker run -e LICENSE=accept -v "$(pwd)":/data ibmcom/icp-inception:3.1.0 cp -r cluster /data && cd cluster
demo@master:/opt/ibm-cloud-private-ce-3.1.0/cluster$ sudo tee /opt/ibm-cloud-private-ce-3.1.0/cluster/hosts <<EOF
[master]
172.16.1.1
[worker]
172.16.1.1
[proxy]
172.16.1.1
#[management]
# 127.0.0.1
#[va]
#5.5.5.5
[all:vars]
ansible_ssh_common_args='-o ControlMaster=auto -o ControlPersist=60s'
ansible_ssh_pipelining=true
EOF

demo@master:/opt/ibm-cloud-private-ce-3.1.0/cluster$ sudo cp ~/.ssh/id_rsa ssh_key
demo@master:/opt/ibm-cloud-private-ce-3.1.0/cluster$ sudo tee install.sh <<EOF
docker run --net=host -t -e LICENSE=accept \
-v "$(pwd)":/installer/cluster ibmcom/icp-inception:3.1.0 install
EOF

demo@master:/opt/ibm-cloud-private-ce-3.1.0/cluster$ sudo chmod +x install.sh
```

#### 1.4.2 Deploy ICP

Go to Skytap Dashboard - your environment. Click on the monitor icon of the master server which launches the remote web console in a new browser's tab. Log in with demo/passw0rd and change directory.

```sh
demo@master:~$ cd /opt/ibm-cloud-private-ce-3.1.0/cluster
demo@master:/opt/ibm-cloud-private-ce-3.1.0/cluster$ ./install.sh
```

Wait about 30 to 40 minutes and let the script finish. At the end, you will be given your ICP dashboard's URL.

### 1.5 Cloudctl, Kubectl & Helm

**Explanation:**

In the following steps, you will be logging in to the default namespace of your ICP cluster using cloudctl and kubectl commands. You will also configure (in-place edit) Cluster Image Policy; deploy a test instance of Wordpress, and remove it using the helm command. The ICP installation on the master node comes with pre-installed cloudctl, kubectl and helm command line interface utilities. 

#### 1.5.1 Cloudctl log-in to ICP

On your desktop/laptop, using Terminal

```sh
# Use your Skytap's SSH URL and port number.
MacBook-Pro:~ username$ ssh -o "ServerAliveInterval 60" -o "ServerAliveCountMax 120" demo@services-uscentral.skytap.com -p 9019 
MacBook-Pro:~ username$ yes
demo@services-uscentral.skytap.com's password: passw0rd
demo@master:~$
```
```sh
demo@master:~$ cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n default
demo@master:~$ kubectl edit clusterimagepolicy
```
Using the down arrow key, place your cursor at the end of the first line under spec: repositories: 
Click the 'a' key to append and come into insert mode. 
Hit Enter. 
On the new line, align with the other lines and type - name: docker.io/bitnami/* 
Hit the Esc key. 
Type :wq! to save.

#### 1.5.2 Helm initialization and test

```sh
demo@master:~$ helm init --client-only
demo@master:~$ helm version --tls
demo@master:~$ helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
demo@master:~$ helm install --name=my-wordpress stable/wordpress --tls
demo@master:~$ kubectl get pods
demo@master:~$ helm delete my-wordpress --purge --tls
```

## Phase 2: Installation/Setup of WebSphere Commerce 9.0.1.3 (WCSv9)

### 2.1 Commerce Namespace

**Explanation:**

Separate namespaces in Kubernetes allows you to organize multiple applications. Namespace allows large and different teams or departments to access resources with accountability. In the following steps, you will be creating a separate namespace for commerce. You will create a login script targeting the commerce namespace, for daily use.

**On your desktop/laptop, using Terminal**

```sh
# Use your Skytap's SSH URL and port number.
MacBook-Pro:~ username$ ssh -o "ServerAliveInterval 60" -o "ServerAliveCountMax 120" demo@services-uscentral.skytap.com -p 9019 
MacBook-Pro:~ username$ yes
demo@services-uscentral.skytap.com's password: passw0rd
demo@master:~$
```

#### 2.1.1 Create 'commerce' namespace

```sh
demo@master:~$ kubectl create namespace commerce
```

#### 2.1.2 Target 'commerce' namespace

```sh
demo@master:~$ cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n commerce
```

#### 2.1.3 Create login script for 'commerce' namespace

```sh
demo@master:~$ tee connecticp.sh <<EOF
cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n commerce
EOF

demo@master:~$ chmod +x connecticp.sh
demo@master:~$ ./connecticp.sh
```

### 2.2 Commerce Role Based Access Control & Role Binding

**Explanation:**

Cluster's pod security policies provide a framework to ensure that pods and containers run only with the appropriate privileges and access only a finite set of resources.

RBAC policies allow you to specify which types of actions are permitted depending on the user and their role.

#### 2.2.1 Role Based Access Control (RBAC)

Copy and paste the YAML file content into your terminal and hit Enter. This will create the `wcs-rbac.yaml` file.

```yaml
tee wcs-rbac.yaml <<EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: commerce-deploy-support-commerce
  namespace: commerce
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list","create","delete","patch","update"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "watch", "list","create","delete","patch","update"]
- apiGroups: [""]
  resources: ["pods","pods/log"]
  verbs: ["get", "watch", "list","create","delete","patch","update"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "watch", "list","create","delete","patch","update"]
---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: commerce-deploy-support-commerce
  namespace: commerce
subjects:
- kind: ServiceAccount
  name: default
  namespace: commerce
roleRef:
  kind: Role
  name: commerce-deploy-support-commerce
  apiGroup: rbac.authorization.k8s.io
EOF
```
Execute the `wcs-rbac.yaml` file.
```sh
demo@master:~$ kubectl create -f ./wcs-rbac.yaml -n commerce
```

#### 2.2.2 Pod Security Policy (PSP)

Copy and paste the YAML file content into your terminal and hit Enter. This will create the `wcs-psp.yaml` file.

```yaml
tee wcs-psp.yaml <<EOF
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: commerce-psp
spec:
  allowPrivilegeEscalation: true
  readOnlyRootFilesystem: false
  allowedCapabilities:
  - CHOWN
  - DAC_OVERRIDE
  - FOWNER
  - FSETID
  - KILL
  - SETGID
  - SETUID
  - SETPCAP
  - NET_BIND_SERVICE
  - NET_RAW
  - SYS_CHROOT
  - MKNOD
  - AUDIT_WRITE
  - SETFCAP
  - SYS_RESOURCE
  - IPC_OWNER
  - SYS_NICE
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - configMap
  - emptyDir
  - persistentVolumeClaim
  - secret
  forbiddenSysctls:
  - '*'
EOF
```
Execute the `wcs-psp.yaml` file.
```sh
demo@master:~$ kubectl create -f ./wcs-psp.yaml -n commerce
```

#### 2.2.3 Apply the policies

```sh
demo@master:~$ kubectl -n commerce create role wcs-psp --verb=use --resource=podsecuritypolicy --resource-name=commerce-psp
demo@master:~$ kubectl -n commerce create rolebinding default:wcs-psp --role=wcs-psp --serviceaccount=commerce:default
```

### 2.4 Docker Log in & Load WCSv9 archive


#### 2.4.1 Prepare docker-load script for v9 archive

```sh
demo@master:~$ cd /opt/images/9.0.1.3
demo@master:/opt/images/9.0.1.3$ sudo tee dockerload.sh <<EOF
docker login -u=admin -p=admin mycluster.icp:8500
cloudctl catalog load-archive --archive WC_ICP_9013_Ent.tgz
EOF

[sudo] password for demo: passw0rd

demo@master:/opt/images/9.0.1.3$ sudo chmod +x dockerload.sh
```

#### 2.4.2 Load v9 archive

Go to Skytap Dashboard - your environment. Click on the monitor icon of the master server which launches the remote web console in a new browser's tab. Log in with demo/passw0rd and change directory.

```sh
demo@master:~$ cd /opt/images/9.0.1.3
demo@master:/opt/images/9.0.1.3$ ./dockerload.sh
```

Wait about 40 minutes. For long period of time, your terminal will be inactive. Behind the scenes, the script is unzipping the files and uploading the images.

**Troubleshooting, if necessary:**

If you get the following message: Failed Error during 'docker push (Are you logged in to the docker registry?). 

Log in to docker once again. 
Reload by running the above cloudctl catalog load... command. The load will eventually finish and you will be back to the command prompt.

### 2.5 Edit Cluster Image Policy for WCSv9 deploy

**Explanation:**

In IBM Cloud Private, running a docker image from any external resources has been tightly controlled. ICP will validate the container images to make sure only those in the predefined whitelist are able to run. In the following steps, you will be white-listing the docker container images which are needed to run WCSv9.

```sh
demo@master:~$ kubectl edit clusterimagepolicy
```
Using the down arrow key, place your cursor at the end of the first line under spec: repositories: 
Click the 'a' key to append and insert mode. 
Hit Enter. 
On the new line, aligning with the others, type 
```
- name: docker.io/vault:*  
- name: docker.io/consul:*  
- name: docker.io/python:*
```
Hit the Esc key. 
Type :wq! to save

### 2.6 Deploy Commerce Vault-Consul via ICP Catalog

Go to Skytap Dashboard - your environment. Click on the monitor icon of the Desktop which launches the remote desktop in a new browser's tab. Log in with demo/passw0rd.

It would be convenient to launch this Box notes from the Ubuntu Desktop. Copy the Box Notes' URL and paste into Skytap Toolbar's paste icon (do this twice for the first time). Launch Firefox and paste the URL. Follow the rest of the instructions from directly within the desktop.

1. From the Ubuntu desktop, launch Firefox icon. Access ICP dashboard by browsing https://172.16.1.1:8443 
2. Accept self-signed certificate.
3. Log in with admin/admin. Save password in browser if you prefer.
4. Click on Catalog (top right), search for commerce. 
5. Click ibm-websphere-commerce-vaultconsul. 
6. Click Configure. 
7. In the Helm Release name field, type vault-consul
8. In the Target Namespace drop-down, select commerce. 
9. Accept License.
10. Click Install.
11. Wait a minute and when prompted click the hamburger menu. Select Workloads. Go to Deployments.
12. Look at the vault.consul deployment (first entry at the top). You should wait until you see all four columns display 1s (it takes only a minute or two). Move forward to the next step.

### 2.7 Deploy Commerce v9 via ICP Catalog

1. Click on Catalog (top right), search for commerce. 
2. Click ibm-websphere-commerce. 
3. Click Configure. 
4. In the Helm Release name field, type demoqaauth
5. In the Target Namespace drop-down, select commerce. 
6. Accept License.
7. Click Install.
8. When you see the "Installation Started" banner, click on View Helm Release button. It will take you to the Helm release section.
9. Scroll down to the Pod section.
10. Wait and watch for about 15 minutes. The Pods will cycle through various stages.
11. Keep an eye on Ready, Status and Age columns.
12. At about 15 minutes mark, the Ready column should have 1/1s and Running status on all pods (except for xc). This confirms that your deployment is successful.
13. From your local terminal, run kubectl get pods to verify the running pods.

### 2.8 Setup host file to access v9 store and tools

1. From the Ubuntu desktop, launch Terminal icon.
2. Type sudo gedit /etc/hosts to open the host file.
3. Type passw0rd
4. Copy/paste the below host name entries and save.
5. Verify by pinging one of the entries: ping store.demoqaauth.ibm.com
6. If the ping does not work then you have to fix the two-character empty space between the IP address and the domain name. Remove the empty space and replace it with the tab space.

```
172.16.1.1        cmc.demoqaauth.ibm.com
172.16.1.1        cmc.demoqalive.ibm.com
172.16.1.1        accelerator.demoqaauth.ibm.com
172.16.1.1        accelerator.demoqalive.ibm.com
172.16.1.1        admin.demoqaauth.ibm.com
172.16.1.1        admin.demoqalive.ibm.com
172.16.1.1        org.demoqaauth.ibm.com
172.16.1.1        org.demoqalive.ibm.com
172.16.1.1        store.demoqaauth.ibm.com
172.16.1.1        store.demoqalive.ibm.com
172.16.1.1        tsapp.demoqaauth.ibm.com
172.16.1.1        searchrepeater.demoqalive.ibm.com
172.16.1.1        search.demoqaauth.ibm.com
172.16.1.1        tsapp.demoqaauth.ibm.com 
172.16.1.1        searchrepeater.demoqalive.ibm.com 
172.16.1.1        search.demoqaauth.ibm.com
```

### 2.9 Indexing for v9 store

From the Ubuntu desktop's Terminal, run the below mentioned command to start the indexing. It will return the jobStatusID `({"jobStatusId":"1001"})`

```sh
demo@desktop:~$ curl -k -u spiuser:passw0rd -X POST https://tsapp.demoqaauth.ibm.com/wcs/resources/admin/index/dataImport/build?masterCatalogId=10001
```

### 2.10 Access Commerce Store & Tools

1. From the Ubuntu Desktop's Firefox, go to the following URL to verify the Aurora sample store (accept self-signed cert for the first time): https://store.demoqaauth.ibm.com/wcs/shop/en/auroraesite
2. Verify you can see categories menu. Do some random search to ensure you get the search results back. This verifies a successful install of WebSphere Commerce v9.
3. Management Center: https://cmc.demoqaauth.ibm.com/lobtools/cmc/ManagementCenter
Username: wcsadmin - Password: wcs1admin
4. Accelerator: https://accelerator.demoqaauth.ibm.com/webapp/wcs/tools/servlet/ToolsLogon?XMLFile=common.mcLogon
5. Administration Console: https://admin.demoqaauth.ibm.com/webapp/wcs/admin/servlet/ToolsLogon?XMLFile=adminconsole.AdminConsoleLogon
6. Organization Administration Console: https://org.demoqaauth.ibm.com/webapp/wcs/orgadmin/servlet/ToolsLogon?XMLFile=buyerconsole.BuyAdminConsoleLogon



