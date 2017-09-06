# README
This README was copied from https://github.com/ktgit/continuous-deployment which was the original 
repository setup by Peter Garbers. This contains significant changes from the original which may be
useful if we ever need to rebuild Jenkins from scratch.

# Build a Continuous Deployment Pipeline with Jenkins and Kubernetes

## Introduction
This guide will take you through the steps necessary to continuously deliver your software to end users by leveraging [Google Container Engine](https://cloud.google.com/container-engine/) and [Jenkins](https://jenkins.io) to orchestrate the software delivery pipeline.
If you are not familiar with basic Kubernetes concepts, have a look at [Kubernetes 101](http://kubernetes.io/docs/user-guide/walkthrough/).

In order to accomplish this goal you will use the following Jenkins plugins:
  - [Jenkins Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) - start Jenkins build executor containers in the Kubernetes cluster when builds are requested, terminate those containers when builds complete, freeing resources up for the rest of the cluster
  - [Jenkins Pipelines](https://jenkins.io/solutions/pipeline/) - define our build pipeline declaratively and keep it checked into source code management alongside our application code
  - [Google Oauth Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+OAuth+Plugin) - allows you to add your google oauth credentials to jenkins

In order to deploy the application with [Kubernetes](http://kubernetes.io/) you will use the following resources:
  - [Deployments](http://kubernetes.io/docs/user-guide/deployments/) - replicates our application across our kubernetes nodes and allows us to do a controlled rolling update of our software across the fleet of application instances
  - [Services](http://kubernetes.io/docs/user-guide/services/) - load balancing and service discovery for our internal services
  - [Ingress](http://kubernetes.io/docs/user-guide/ingress/) - external load balancing and SSL termination for our external service
  - [Secrets](http://kubernetes.io/docs/user-guide/secrets/) - secure storage of non public configuration information, SSL certs specifically in our case

## Prerequisites
1. A Google Cloud Platform Account
1. [Enable the Google Compute Engine and Google Container Engine APIs](https://console.cloud.google.com/flows/enableapi?apiid=compute_component,container)

## Do this first
In this section you will start your [Google Cloud Shell](https://cloud.google.com/cloud-shell/docs/) and clone this repository.

1. Create a new Google Cloud Platform project: [https://console.developers.google.com/project](https://console.developers.google.com/project)

1. Click the Google Cloud Shell icon in the top-right and wait for your shell to open:

  ![](docs/img/cloud-shell.png)

  ![](docs/img/cloud-shell-prompt.png)

1. When the shell is open, set your default compute zone:

  ```shell
  $ gcloud config set compute/zone us-east1-d
  ```

1. Clone the lab repository in your cloud shell, then `cd` into that dir:

  ```shell
  $ git clone https://github.com/ktgit/continuous-deployment.git
  Cloning into 'continuous-deployment'...
  ...
  $ cd continuous-deployment
  ```

##  Create a Kubernetes Cluster
You'll use Google Container Engine to create and manage your Kubernetes cluster. Provision the cluster with `gcloud`:

```shell
$ gcloud container clusters create jenkins \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw,cloud-platform"
```

Once that operation completes download the credentials for your cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials jenkins
Fetching cluster endpoint and auth data.
kubeconfig entry generated for jenkins.
```

Confirm that the cluster is running and `kubectl` is working by listing pods:

```shell
$ kubectl get pods
```
You should see an empty response.

## Create namespace and quota for Jenkins

Create the `jenkins` namespace:
```shell
$ kubectl create ns jenkins
```

### Create the Jenkins Home Volume
In order to pre-populate Jenkins with the necessary plugins and configuration for the rest of the tutorial, you will create
a volume from an existing tarball of that data.

```shell
gcloud compute images create jenkins-home-image --source-uri https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home.tar.gz
gcloud compute disks create jenkins-home --image jenkins-home-image
```

If this is not a new deployment there may already be a snapshot

```shell
gcloud compute disks create jenkins-home --source-snapshot <snapshot-name>
```

### Create a Jenkins Deployment and Service
Here you'll create a Deployment running a Jenkins container with a persistent disk attached containing the Jenkins home directory.

First, set the password for the default Jenkins user. Edit the password in `jenkins/k8s/options` with the password of your choice by replacing _CHANGE_ME_. To Generate a random password and replace it in the file, you can run:

```shell
$ PASSWORD=`openssl rand -base64 15`; echo "Your password is $PASSWORD"; sed -i.bak s#CHANGE_ME#$PASSWORD# jenkins/k8s/options
```

Now create the secret using `kubectl`:
```shell
$ kubectl create secret generic jenkins --from-file=jenkins/k8s/options --namespace=jenkins
secret "jenkins" created
```

Additionally you will have a service that will route requests to the controller.

> **Note**: All of the files that define the Kubernetes resources you will be creating for Jenkins are in the `jenkins/k8s` folder. You are encouraged to take a look at them before running the create commands.

The Jenkins Deployment is defined in `kubernetes/jenkins.yaml`. Create the Deployment and confirm the pod was scheduled:

```shell
$ kubectl apply -f jenkins/k8s/
deployment "jenkins" created
service "jenkins-ui" created
service "jenkins-discovery" created
```

The Jenkins UI service was implemented with a NodePort. We will need to get the port that was exposed in order to
allow the [Ingress HTTP load balancer](https://cloud.google.com/container-engine/docs/tutorials/http-balancer) to reach the UI:

```shell
$ kubectl describe svc --namespace jenkins jenkins-ui
Name:			jenkins-ui
Namespace:		jenkins
Labels:			<none>
Selector:		app=master
Type:			NodePort
IP:			10.123.123.123
Port:			ui	8080/TCP
NodePort:		ui	30573/TCP   <----- THIS IS THE PORT NUMBER OPENED UP ON EACH CLUSTER NODE
Endpoints:		<none>
Session Affinity:	None
No events.
```

Next we will open up the firewall to allow access to those ports:

```shell
$ export NODE_PORT=$(kubectl get --namespace=jenkins -o jsonpath="{.spec.ports[0].nodePort}" services jenkins-ui)
$ gcloud compute firewall-rules create allow-130-211-0-0-22-$NODE_PORT --source-ranges 130.211.0.0/22 --allow tcp:$NODE_PORT
Created [https://www.googleapis.com/compute/v1/projects/vic-goog/global/firewalls/allow-130-211-0-0-22].
NAME                 NETWORK SRC_RANGES     RULES     SRC_TAGS TARGET_TAGS
allow-130-211-0-0-22 default 130.211.0.0/22 tcp:30573          gke-jenkins-cd-e5d0264b-node
```

Check that your master pod is in the running state

```shell
$ kubectl get pods --namespace jenkins
NAME                   READY     STATUS    RESTARTS   AGE
jenkins-master-to8xg   1/1       Running   0          30s
```

Now, check that the Jenkins Service was created properly:

```shell
$ kubectl get svc --namespace jenkins
NAME                CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
jenkins-discovery   10.79.254.142   <none>        50000/TCP   10m
jenkins-ui          10.79.242.143   nodes         8080/TCP    10m
```

We are using the [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them.
Upon completion of their work they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.
Additionally the `jenkins-ui` services is exposed using a NodePort so that our HTTP loadbalancer can reach it.

Kubernetes makes it simple to deploy an [Ingress resource](http://kubernetes.io/docs/user-guide/ingress/) to act as a public load balancer and SSL terminator.

The Ingress resource is defined in `jenkins/k8s/lb/ingress.yaml`. We used the Kubernetes `secrets` API to add our certs securely to our cluster and ready for the Ingress to use.

In order to create your own certs run:

```shell
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"
```

Now you can upload them to Kubernetes as secrets:
```shell
$ kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
```

Now that the secrets have been uploaded, create the ingress load balancer. Note that the secrets must be created before the ingress, otherwise the HTTPs endpoint will not be created.

```shell
$ kubectl apply -f jenkins/k8s/lb
```

<a name="connect-to-jenkins"></a>
### Connect to Jenkins

Now find the load balancer IP address of your Ingress service (in the `Address` field). **This field may take a few minutes to appear as the load balancer is being provisioned**:

```shell
$  kubectl get ingress --namespace jenkins
NAME      RULE      BACKEND            ADDRESS         AGE
jenkins      -         master:8080        130.X.X.X      4m
```

The loadbalancer will begin health checks against your Jenkins instance. Once the checks go to healthy you will be able to access your Jenkins instance:
```shell
$  kubectl describe ingress jenkins --namespace jenkins
Name:			jenkins
Namespace:		jenkins
Address:		130.211.14.253
Default backend:	jenkins-ui:8080 (10.76.2.3:8080)
TLS:
  tls terminates
Rules:
  Host	Path	Backends
  ----	----	--------
Annotations:
  https-forwarding-rule:	k8s-fws-jenkins-jenkins
  https-target-proxy:		k8s-tps-jenkins-jenkins
  static-ip:			k8s-fw-jenkins-jenkins
  target-proxy:			k8s-tp-jenkins-jenkins
  url-map:			k8s-um-jenkins-jenkins
  backends:			{"k8s-be-32371":"HEALTHY"}   <---- LOOK FOR THIS TO BE HEALTHY
  forwarding-rule:		k8s-fw-jenkins-jenkins
Events:
  FirstSeen	LastSeen	Count	From				SubobjectPath	Type		Reason	Message
  ---------	--------	-----	----				-------------	--------	------	-------
  2m		2m		1	{loadbalancer-controller }			Normal		ADD	jenkins/jenkins
  1m		1m		1	{loadbalancer-controller }			Normal		CREATE	ip: 130.123.123.123 <--- This is the load balancer's IP
```

Open the load balancer's IP address in your web browser, click "Log in" in the top right and sign in with the default Jenkins username `jenkins` and the password you configured when deploying Jenkins. You can find the password in the `jenkins/k8s/options` file.

> **Note**: To further secure your instance follow the steps found [here](https://wiki.jenkins-ci.org/display/JENKINS/Securing+Jenkins).


![](docs/img/jenkins-login.png)

### Your progress, and what's next
You've got a Kubernetes cluster managed by Google Container Engine. You've deployed:

* a Jenkins Deployment
* a (non-public) service that exposes Jenkins to its slave containers
* an Ingress resource that routes to the Jenkins service

You have the tools to build a continuous deployment pipeline. Now you need a sample app to deploy continuously.

## Jenkins setup

### Add your service account credentials
First we will need to configure our GCP credentials in order for Jenkins to be able to access our code repository

1. In the Jenkins UI, Click “Credentials” on the left
1. Click “Global Credentials”
1. Click “Add Credentials” on the left
1. From the “Kind” dropdown, select “Google Service Account from metadata”
1. Click “OK”

![](docs/img/jenkins-credentials.png)

### Github Credentials

1. In the Jenkins UI, Click “Credentials” on the left
1. Click “Global Credentials”
1. Click “Add Credentials” on the left. 
1. From the “Kind” dropdown, select “Username with password”
1. Enter the credentials
1. Click “OK”

### jenkins-k8s-slave
Because we currently require special ssh keys we need to use a custom built [jenkins slave](https://github.com/ktgit/jenkins-k8s-slave)

1. In the Jenkins UI, Click "Manage Jenkins"
1. Click "Configure System"
1. In the "Cloud" section modify "Docker Image" to the location in our docker repository (i.e. gcr.io/knowledgetree-gcp/jenkins-k8s-slave)

## Working with Jenkins
You'll now use Jenkins to define and run a pipeline that will test, build, and deploy.


### Phase 2: Create a job
This lab uses [Github Organizations](https://jenkins.io/solutions/pipeline/) to define builds as groovy scripts.

Navigate to your Jenkins UI and follow these steps to configure a Github Organizations job (hot tip: you can find the IP address of your Jenkins install with `kubectl get ingress --namespace jenkins`):

1. Click the “Jenkins” link in the top left of the interface

1. Click the **New Item** link in the left nav

1. Name the project **Knowledgetree**, choose the **Github Organization** option, then click `OK`

1. Under `Project Sources`->`Owner` type in the name of the github Organization (i.e. ktgit)

1. Under `Scan Credentials` select the github credentials we entered earlier

1. Click `Save`

  ![](docs/img/clone_url.png)

A job entitled "Branch indexing" was kicked off to see identify the branches in your repository. If you refresh Jenkins you should see the organization folder and the branches below it

This job identifies all branches that currently have a `Jenkinsfile` and tries to run those branches. 

To clean up, navigate to the [Google Developers Console Project List](https://console.developers.google.com/project), choose the project you created for this lab, and delete it. That's it.

## Re-attaching Volume

Delete the deployment 

```shell
kubectl delete deployment/jenkins --namespace jenkins
```

Detach the existing jenkins_home ([additional information](https://cloud.google.com/sdk/gcloud/reference/compute/instances/detach-disk))

```shell
gcloud compute instances detach-disk gke-<instance-identifier> --disk <volume-name (jenkins-home)> --zone <zone>
```

If you need to restore the volume from the [backup](#user-content-create-the-jenkins-home-volume)

Create a new deployment

```shell
kubectl apply -f jenkins/k8s/jenkins.yaml
```

## Updating the Kubernetes-dashboard

1. Find the credentials

```
kubectl config view
```

2. Create a proxy to the cluster

```
kubectl proxy
```

3. Navigate to navigate to the [Kubernetes Dashboard](http://localhost:8001/ui)
4. Login using the credentials found in 1
5. Under the namepace of `kube-system` delete the dashboard and all it's related services
6. Create a new dashboard 

```
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

For additonal information visit [Continuous deployment on kubernetes](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes)
