

![icp000](images/icp000.png)



---
# Microclimate Lab
---


![microclimate](images/mcmicroclimate.png)
This lab is compatible with ICP version 3.1

---


In this tutorial, you create, install and run a **cloud-native microservice application** on an IBM® Cloud Private platform on Kubernetes.

Microclimate will guide you thru the creation of complete project including all the directories, the manifest files, the monitoring option that you need for a perfect application.

[Link to Microclimate documentation here](https://microclimate-dev2ops.github.io/)

## Table of Contents

---

- [Task 1: Access the console and check Helm](#task-1--access-the-console-and-check-helm)
- [Task 2: Install Microclimate](#task-2--install-microclimate)
  * [1. Prepare the environment](#1-prepare-the-environment)
    + [Add persistent volumes](#add-persistent-volumes)
    + [Create a Docker registry secret for Microclimate](#create-a-docker-registry-secret-for-microclimate)
    + [Patch this secret to a service account](#patch-this-secret-to-a-service-account)
    + [Ensure target namespace for deployments](#ensure-target-namespace-for-deployments)
    + [Create a secret to use Tiller over TLS](#create-a-secret-to-use-tiller-over-tls)
  * [2. Install from the command line](#2-install-from-the-command-line)
- [Task 3: Install a simple application](#task-3--install-a-simple-application)
- [Task 4: Import and deploy an application](#task-4--import-and-deploy-an-application)
- [Task 5: Use a pipeline](#task-5--use-a-pipeline)
- [Congratulations](#congratulations)

---




# Task 1: Access the console and check Helm


From a machine that is hosting your environment, open a web browser and go to one of the following URLs to access the IBM Cloud Private management console:
  - Open a browser
  - go to https://mycluster.icp:8443
  - Login to ICP console with admin / admin

![Login to ICP console](./images/login2icp.png)

On the terminal, connect on the Ubuntu VM using SSH or Putty.

Check the helm command is working:

`helm version --tls`

If you get an **error** or you didn't get the Client and Tiller Server versions, then go back in the section 10 in the installation lab.

# Task 2: Install Microclimate


Application workloads can be deployed to run on an IBM Cloud Private cluster. The deployment of an application workload must be made available as a Helm package. Such packages must either be made available for deployment on a Helm repository, or loaded into an IBM Cloud Private internal Helm repository.

## 1. Prepare the environment

There are several prerequisites:
- create a new namespace
- add persistent volumes
- Create a Docker registry secret for Microclimate
- Patch this secret to a service account.
- Ensure that the target namespace for deployments can access the docker image registry.
- Create a secret so Microclimate can securely use Helm.
- Set the Microclimate and Jenkins hostname values

### Create a namespace

We are going to install Microclimate into a new namespace called "microclimate". For Jenkins, it will be installed in an isolated namespace.

`kubectl create namespace microclimate`

Results:

```
# kubectl create namespace microclimate
namespace/microclimate created
```

Replace the ipaddress with the one from the cluster:

`cloudctl login -a https://ipaddress:8443 -n microclimate --skip-ssl-validation`

Results:

```
# cloudctl login -a https://159.8.182.80:8443 -n microclimate --skip-ssl-validation

Username> admin

Password> 
Authenticating...
OK

Select an account:
1. mycluster Account (id-mycluster-account)
Enter a number> 1
Targeted account mycluster Account (id-mycluster-account)

Targeted namespace microclimate

Configuring kubectl ...
Property "clusters.mycluster" unset.
Property "users.mycluster-user" unset.
Property "contexts.mycluster-context" unset.
Cluster "mycluster" set.
User "mycluster-user" set.
Context "mycluster-context" created.
Switched to context "mycluster-context".
OK

Configuring helm: /root/.helm
OK
```



### Add persistent volumes

<u>Some persistent volumes have been created in the installation lab.</u>

If not, create a file with that command:

`nano pv-mc.yaml`

Copy and paste the following text in the file (this is a kubernetes document):

```console
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwo-mc1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwo-mc2
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwm-mc1
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle  
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwm-mc2
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle  
```
Save the file (ctrl O, enter, ctrl X ).

And then use the following command:

`kubectl create  -f ./pv-mc.yaml`

Results:
```console
# kubectl create  -f ./pv-mc.yaml
persistentvolume "hostpath-pv-rwo-mc1" created
persistentvolume "hostpath-pv-rwo-mc2" created
persistentvolume "hostpath-pv-rwm-mc1" created
persistentvolume "hostpath-pv-rwm-mc2" created
```

Once created, these 4 volumes (hostpath) can be listed with the following command:

`kubectl get pv` 


Results:
```console
# kubectl get pv
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                       STORAGECLASS               REASON    AGE
helm-repo-pv                  5Gi        RWO            Delete           Bound       kube-system/helm-repo-pvc                   helm-repo-storage                    3h
hostpath-pv-many-test1        50Gi       RWX            Recycle          Available                                                                                    2h
hostpath-pv-once-test1        30Gi       RWO            Recycle          Available                                                                                    2h
hostpath-pv-rwm-mc1           10Gi       RWX            Recycle          Available                                                                                    1m
hostpath-pv-rwm-mc2           10Gi       RWX            Recycle          Available                                                                                    1m
hostpath-pv-rwo-mc1           10Gi       RWO            Recycle          Available                                                                                    1m
hostpath-pv-rwo-mc2           10Gi       RWO            Recycle          Available                                                                                    1m
image-manager-5.10.96.73      20Gi       RWO            Retain           Bound       kube-system/image-manager-image-manager-0   image-manager-storage                3h
logging-datanode-5.10.96.73   20Gi       RWO            Retain           Bound       kube-system/data-logging-elk-data-0         logging-storage-datanode             3h
mongodb-5.10.96.73            20Gi       RWO            Retain           Bound       kube-system/mongodbdir-icp-mongodb-0        mongodb-storage                      3h
```

### Create a Docker registry secret for Microclimate

``` 
kubectl create secret docker-registry microclimate-registry-secret \
  --docker-server=mycluster.icp:8500 \
  --docker-username=admin \
  --docker-password=admin \
  --docker-email=null
```

Results:
```console
# kubectl create secret docker-registry microclimate-registry-secret \
>   --docker-server=mycluster.icp:8500 \
>   --docker-username=admin \
>   --docker-password=admin \
>   --docker-email=null
secret "microclimate-registry-secret" created
```

### Patch this secret to a service account

`kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "microclimate-registry-secret"}]}'`

Results:
```console
# kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "microclimate-registry-secret"}]}'
serviceaccount "default" patched
```

### Ensure target namespace for deployments

The chart parameter jenkins.Pipeline.TargetNamespace defines the namespace that the pipeline deploys to. Its default value is "microclimate-pipeline-deployments". This namespace must be created before using the pipeline. Ensure that the default service account in this namespace has an associated image pull secret that permits pods in this namespace to pull images from the ICP image registry. For example, you might create another docker-registry secret and patch the service account:

`kubectl create namespace microclimate-pipeline-deployments`

Results:

```
# kubectl create namespace microclimate-pipeline-deployments
namespace/microclimate-pipeline-deployments created
```

And then use these 2 commands:

```

kubectl create secret docker-registry microclimate-registry-secret \
  --namespace=microclimate-pipeline-deployments \
  --docker-server=mycluster.icp:8500 \
  --docker-username=admin \
  --docker-password=admin \
  --docker-email=null

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "microclimate-registry-secret"}]}' --namespace microclimate-pipeline-deployments
```

### Create a secret to use Tiller over TLS

Microclimate's pipeline deploys applications by using the Tiller at kube-system. Secure communication with this Tiller is required and must be configured by creating a Kubernetes secret that contains the required certificate files as detailed below.

To create the secret, use the following command replacing the values with where you saved your files:

```
kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=.helm/cert.pem --from-file=ca.pem=.helm/ca.pem --from-file=key.pem=.helm/key.pem
```

Results:
```console 
# kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=.helm/cert.pem --from-file=ca.pem=.helm/ca.pem --from-file=key.pem=.helm/key.pem
secret "microclimate-helm-secret" created
```


## 2. Install from the command line 

First define the **ibm-charts** helm repo (if not done in a previous exercise):
```
helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/
```

Then install Microclimate (change ipaddress with your cluster address)

```
helm install --name microclimate --namespace microclimate --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,global.ingressDomain=ipaddress.nip.io ibm-charts/ibm-microclimate --tls
```
> **VERY IMPORTANT** : Change with the **ipaddress** of the ICP Cluster.
> It can take a few minutes before you can see the following results:

```console
NAME:   microclimate
LAST DEPLOYED: Sat Dec  1 15:18:55 2018
NAMESPACE: microclimate
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                                                 DATA  AGE
microclimate-jenkins                                 7     50s
microclimate-jenkins-tests                           1     50s
microclimate-ibm-microclimate-create-mc-secret       1     50s
microclimate-ibm-microclimate-create-target-ns       1     50s
microclimate-helmtest-devops                         1     50s
microclimate-ibm-microclimate-jenkinstest            1     50s
microclimate-ibm-microclimate-fixup-jenkins-ingress  1     50s

==> v1/ServiceAccount
NAME         SECRETS  AGE
pipeline-sa  1        50s
micro-sa     1        50s

==> v1beta1/ClusterRole
NAME                                     AGE
microclimate-ibm-microclimate-cr-devops  50s
microclimate-ibm-microclimate-cr-micro   50s

==> v1beta1/Deployment
NAME                                  DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
microclimate-jenkins                  1        1        1           0          50s
microclimate-ibm-microclimate-atrium  1        1        1           1          50s
microclimate-ibm-microclimate-devops  1        1        1           0          50s
microclimate-ibm-microclimate         1        1        1           1          49s

==> v1beta1/Ingress
NAME                           HOSTS                             ADDRESS  PORTS  AGE
microclimate-jenkins           jenkins.159.8.182.80.nip.io       80, 443  49s
microclimate-ibm-microclimate  microclimate.159.8.182.80.nip.io  80, 443  49s

==> v1/Pod(related)
NAME                                                   READY  STATUS   RESTARTS  AGE
microclimate-jenkins-86f48bc947-bjbbx                  0/1    Running  0         50s
microclimate-ibm-microclimate-atrium-75d746f75c-6x6bc  1/1    Running  0         50s
microclimate-ibm-microclimate-devops-675b78d749-w42lx  0/1    Running  0         50s
microclimate-ibm-microclimate-65cb4576bc-w6xh9         1/1    Running  0         49s

==> v1/Secret
NAME                           TYPE    DATA  AGE
microclimate-jenkins           Opaque  2     50s
microclimate-ibm-microclimate  Opaque  3     47s

==> v1/PersistentVolumeClaim
NAME                           STATUS  VOLUME                  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
microclimate-jenkins           Bound   hostpath-pv-once-test1  200Gi     RWO           50s
microclimate-ibm-microclimate  Bound   hostpath-pv-many-test1  200Gi     RWX           50s

==> v1beta1/ClusterRoleBinding
NAME                                      AGE
microclimate-ibm-microclimate-crb-devops  50s
microclimate-ibm-microclimate-crb-micro   50s

==> v1/RoleBinding
NAME                  AGE
microclimate-binding  50s

==> v1/Service
NAME                                  TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)                     AGE
microclimate-jenkins-agent            ClusterIP  10.0.0.30   <none>       50000/TCP                   50s
microclimate-jenkins                  ClusterIP  10.0.0.127  <none>       8080/TCP                    50s
microclimate-ibm-microclimate-devops  ClusterIP  10.0.0.63   <none>       9191/TCP                    50s
microclimate-ibm-microclimate         ClusterIP  10.0.0.175  <none>       4191/TCP,9191/TCP,9091/TCP  50s


NOTES:
ibm-microclimate-1.8.0

1. Access the Microclimate portal at the following URL: https://microclimate.159.8.182.80.nip.io
```

Check that all deployments are ready (the available column should have a 1 on every row) :
`kubectl get deployment`

Results:
```console 
# kubectl get deployment
NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
microclimate-ibm-microclimate          1         1         1            1           5m
microclimate-ibm-microclimate-atrium   1         1         1            1           5m
microclimate-ibm-microclimate-devops   1         1         1            1           5m
microclimate-jenkins                   1         1         1            1           5m
```

Finally get access to the Microclimate portal with the following link in your browser:

`https://microclimate.<ipaddress>.nip.io`
(replace <ipaddress> with your icp ip address)

Accept the license agreement:

![Accept the license](./images/mcaccept.png)

You should click on **Done** and then you are on the main menu:
![Main](./images/mcaccess.png)

This concludes the Microclimate installation. 

> Note: this installation can also be done thru the ICP Catalog.


# Task 3: Install a simple application

You’re now ready to deploy your Kubernetes application to the IBM Cloud Private environment.  In this case, the deploy command will :


Click on **New project** button:

![project](./images/mcproject.png)

Choose Node (because we want to create a Node.js application) and type the name: **nodeone** and click **Next**

![nodeone](./images/mcnodeone.png)

On the next menu, don't choose any service (but you can notice that we can bind a service to an application), then click on **Create**

![services](./images/mcservice.png)

Be patient (it could take a few minutes). Your application should appear and the building process could still be running: 

![services](./images/mcmain.png)

After a few minutes, the application should be running (check the green light):

![image-20181130235206618](images/image-20181130235206618.png)

To access the application, click on the **Open App** button on the left pane:

![Open App](./images/mcopenapp.png)

The application should appear:

![My App](./images/mcapp.png)


Navigate on the left pane to the **Edit** button:

![services](./images/mcedit.png)

At this point, the editor can be used to edit the Node.JS application. 
Expand **nodeone>public>index.html**

![index](./images/mcindex.png)

Go to the end of the index.html file and modify the **congratulation** line :

![congratulation](./images/mcmodify.png)

Then re-build the application:

![build](./images/mcbuild.png)

Wait until the green light to see the modification:

![build](./images/mcpht.png)

From the main menu of the application, click on **App Monitor** to see some metrics:

![build](./images/mcmon.png)


# Task 4: Import and deploy an application

To import a new application from Github, click on the **Project** button. Then choose **Import project**:

![build](./images/mcimport.png)

On the Import page, type `https://github.com/microclimate-demo/node` as the Git location :

![build](./images/mcgit.png)

Enter a name : `nodetwo` and click **Finish**

![build](./images/mcname.png)

When the application is running : 

![build](./images/mcbuild2.png)

Click on **Open App**:

![build](./images/mcapp2.png)


# Task 5: Use a pipeline

Goto the pipeline, type a pipeline name `node3`and a repository to get the application  `https://github.com/microclimate-demo/node`  and click on **Create pipeline**:

![build](./images/mcpipe3.png)

Click on Open Pipeline to get access to Jenkins  (enter your credentials at some point):

![build](./images/mcseepipe.png)

Click on **master** and look at the progression:

![build](./images/mcjenkins.png)

Wait until the progression ends:

![build](./images/mcstage.png)



# Congratulations

You have successfully created and installed a microservice application with Microclimate.



![icp000](images/icp000.png)