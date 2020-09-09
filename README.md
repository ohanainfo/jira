# Deploy Jira Datacenter on Kubernetes.. a short history
This is a guide to deploy Jira Datacenter on Google Kubernetes Engine

This is my first post related to Infra as a code to deploy Atlassian products. Hope you enjoy and use (or part of) it to deploy your own infrastructure… by the way this is my first article on Linkedin too :-)

Let me explain why I did this…

I´m working at a customer that is evaluating some applications to move to Kubernetes  and since I am the “Atlassian guy” here I took the Jira Datacenter running on kubernetes POC challenge.

Well.. challenge accepted and now? :thinking:

I have a lot of expertise related to Servers, Collaboration, Network Infrastructure and deploying Jira Datacenter on Azure using the blueprint provided by Atlassian as example, but I didn´t touch Kubernetes previously.  So, my firts step was learn about it. I suggest you to take the Coursera course offered by Google. I made it and received a certificate (https://coursera.org/share/b0b439a528f073364a29c3ccf4ee3636).

Then, since I received some credits at GCP, I decided to use it as my Cloud Provider (you can use any other to do that, but in my example we will use everything that works and fit for GKE - Google Kubernetes Engine.

I know that this is a simple deployment with a lot of space from improvements.

Let´s start the CookBook :yum:

### PS - I assume you have some knowledge with Kubernetes, otherwise, take the free course from Google.

*Since an image spokes for thousand words... The following diagram explain how I deployed the environment.*
![Kube Diagram](https://github.com/ohanainfo/JiraDC_on_GKE/blob/master/images/Kube-diagram.jpg)
To deploy that info I made a research and read hundreds of sites and knowledge articles from Google, Kubernetes Github, Medium Articles, Kubernetes.io articles, StackOverFlow posts…  but my special thanks go to Praqma and Steven Hipwell whose I learned a lot from their articles. 

Let´s end the smooth talk and jump to the code.

Our pre-requisites are:

- [x] A previously PostgreSQL instance to adjust the parameters on values.yaml to passthrough the variables to the Jira Pods.
- [x] A FileStore Instance to be used as SharedHome (on other Cloud Providers you can create a NFS Disk directly from Helm, but not at GCP yet).
There is a order to deploy, so pay attention:

1 - Create the Shared Persistent Disk first (nfs-pv.yaml file inside templates folder). Edit with the Sharefolder and IP address of NFS Disk created previously following the steps below:
```
    #edit the nfs-pv.yaml file to adjust properties
    vi jira/templates/nfs-pv.yaml
    #deploy the NFS Persistent Volume
    kubectl apply -f jira/templates/nfs-pv.yaml
```
2 - After that, create a service account named Jira with the following command:
```
kubectl apply -f jira/templates/serviceaccount.yaml
```
3 - Take a look on the Helm file (values.yaml) and edit the necessary parameters. **IMPORTANT**
```
apiVersion: apps/v1
#Initially deploy with 1 replica and then adjust the number to your needs
Replicas: 1
ContainerName: jira

RBAC:
  Enabled: true
Datacenter:
  Enabled: true

#choose the version you want to deploy from Official repo from Atlassian
Image:
  Name: "atlassian/jira-software"
  Tag: "8.5.3"
  ImagePullPolicy: "IfNotPresent"

ContainerPort: 8080
TerminationGracePeriodSeconds: 50

EnvVars:
  X_PROXY_NAME: "jira.local" # change it later on Jira System
  X_PROXY_PORT: "80"
  X_PROXY_SCHEME: "http"
  X_CONTEXT_PATH: ""
  ADDITIONAL_CONNECTOR: "false"
  JAVA_OPTS: ""
  CATALINA_OPTS: ""
  jvmMinHeapSize: 384M
  jvmMaxHeapSize: 768M
  jvmAdditionalMemoryOptions: "-XX:MaxMetaspaceSize=512m -XX:MaxDirectMemorySize=10m"
  jvmAdditionalOptions: ""
service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx

  path: /
  hosts:
    - jira.local

SecurityContext: 1000

Persistence:
  DatacenterMountPath: /var/atlassian/jira-datacenter/
  VolumeClaimTemplates:
     AccessModes: ReadWriteOnce
     Storage: 5Gi
     StorageClassName: standard
     Selector:
       Enabled: false
       MatchLabels:
         app: jira
         system: production
         type: homefolder

PodDisruption:
  Enabled: true
  MinAvailable: 1

#Adjust the details below from the Google SQL Instance created (create a blank DB and add the jiradb user.
psql:
  HOST: "10.43.16.3"
  PORT: "5432"
  DATABASE: "jiradb"
  DBUSER: "jira-user"
  DBPASS: "Jira@123"
  ATL_JDBC_URL: "jdbc:postgresql://10.43.16.3:5432/jiradb"
```


