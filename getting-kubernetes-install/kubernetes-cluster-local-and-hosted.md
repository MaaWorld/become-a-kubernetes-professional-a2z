---
icon: kubernetes
---

# Kubernetes Cluster: Local and Hosted

## Local Kubernetes Cluster <a href="#page-title" id="page-title"></a>

This lesson will walk us through building a single-node Kubernetes cluster with Docker Desktop.

We’ll need to complete the following steps to build the cluster:

* Install Docker Desktop.
* Enable Docker Desktop’s built-in Kubernetes cluster.
* Test our cluster.

### Install Docker Desktop <a href="#install-docker-desktop" id="install-docker-desktop"></a>

Docker Desktop is the easiest way to get Docker, Kubernetes, and `kubectl` on the laptop. We also get a nice UI that makes switching between `kubectl` contexts eas&#x79;_._

`kubectl` is the Kubernetes command-line utility, and we’ll need it for all the examples in the course.

A `kubectl` context is a collection of settings telling `kubectl` which cluster to issue commands to and which credentials to authenticate with.

Complete the following simple steps to install Docker Desktop:

1. Search the web for Docker Desktop.
2. Download the installer for the system (Linux, Mac, or Windows).
3. Fire up the installer and follow the next, next, next instructions.

Windows users should install the WSL 2 subsystem when prompted.

After installation, we may need to start the app manually. Mac users get a whale icon in the menu bar at the top while running, whereas Windows users get the whale in the system tray at the bottom. Clicking the whale exposes some basic controls and shows whether Docker Desktop is running.

Open a terminal and run the following commands to ensure Docker and `kubectl` are installed and working.

```powershell
$ docker --version
Docker version 25.0.2, build 29cf629

$ kubectl version --client=true -o yaml
clientVersion:
  compiler: gc
  gitVersion: v1.29.1
  major: "1"
  minor: "29"
  platform: darwin/arm64

// Get the Docker version
```

### Enable Docker Desktop’s built-in Kubernetes cluster <a href="#enable-docker-desktops-built-in-kubernetes-cluster" id="enable-docker-desktops-built-in-kubernetes-cluster"></a>

Click the Docker whale icon in the menu bar or system tray and choose the "Settings" option.

Select "Kubernetes" from the left navigation bar, check the "Enable Kubernetes" option, and click "Apply & restart".

It’ll take a minute or two for Docker Desktop to pull the required images and start the cluster. The Kubernetes icon in the bottom left of the Docker Desktop window will turn green when the cluster is up and running.

### Test the cluster <a href="#test-the-cluster" id="test-the-cluster"></a>

Run the following command to ensure the cluster is up and running and the `kubectl` context is set.

```cpp
// main.cpp

$ kubectl get nodes
NAME              STATUS    ROLES             AGE    VERSION
docker-desktop    Ready     control-plane     93d    v1.29.1

// Get the state of the nodes
```

Congratulations! We’ve built a Kubernetes cluster on the laptop that you can use.

## Hosted Kubernetes Cluster <a href="#page-title" id="page-title"></a>

Learn how we can create a Kubernetes cluster on the cloud.

This option costs money. Be sure you understand the costs before you create this cluster. We also recommend you delete it as soon as you finish using it.

All the major cloud platforms offer a hosted Kubernetes service. This is a model where the cloud provider builds the cluster and manages things such as high availability (HA), performance, and updates.

Not all hosted Kubernetes services are equal, but they’re usually as close as you’ll get to a zero-effort production-grade Kubernetes cluster. For example, Google Kubernetes Engine (GKE) is a hosted service that creates high-performance, highly-available clusters that implement security best practices out of the box, all with just a few simple clicks and your credit card details.

Other popular hosted Kubernetes services include:

* AWS: Elastic Kubernetes Service (EKS)
* Azure: Azure Kubernetes Service (AKS)
* Civo Cloud Kubernetes
* DigitalOcean: DigitalOcean Kubernetes (DOKS)
* Google Cloud Platform: Google Kubernetes Engine (GKE)
* Linode: Linode Kubernetes Engine (LKE)

We’ll create a GKE cluster, and you’ll complete all of the following steps:

* GKE prerequisites
* Create a GKE cluster
* Test your GKE cluster

### GKE prerequisites <a href="#gke-prerequisites" id="gke-prerequisites"></a>

GKE is a hosted Kubernetes service on the Google Cloud Platform (GCP). Like most hosted Kubernetes services, it provides:

* A fast and easy way to get a production-grade cluster
* A managed control plane
* Itemized billing
* Integration with additional services such as load balancers, volumes, service meshes, and more

To build a GKE cluster, you’ll need a Google Cloud account with billing configured and a blank project. These are simple to set up; the remainder of this section assumes you already have them.

You’ll also need the `gcloud` CLI. Go to `https://cloud.google.com/sdk/`, click the "Get started" button, and follow the instructions to install the version for your platform. The installer will automatically install the `kubectl` command-line utility. As part of the installation, you’ll be prompted to run a `gcloud auth login` command to authorize access to your Google Cloud project. This will open a browser session where you need to follow and accept the prompts.

### Create a GKE cluster <a href="#create-a-gke-cluster" id="create-a-gke-cluster"></a>

Once you’ve got a new Google Cloud project and installed the `gcloud` CLI, complete the following steps to create a new GKE cluster.

1. Go to `https://console.cloud.google.com/` and select "Kubernetes Engine" > "Clusters" from the navigation pane on the left. You may need to click the three horizontal bars (hamburger) in the top left corner to make the navigation pane visible.
2. Select the option to create a cluster and then choose the option to "SWITCH TO STANDARD CLUSTER". Do not create an AutoPilot cluster, as these don’t currently work with all examples. You’ll be prompted to confirm you want to switch from autopilot to standard.
3. Give your cluster a meaningful name. The examples in the course will use `gke-tkb`.
4. Choose a "Regional" cluster in the "Location type". Some examples later in the course will only work with the regional clusters. The course assumes your cluster is located in `us-central1` region. If you wish to change the region, kindly change the region in the commands as well.
5. Select "Region" for your cluster.
6. Click "Target Release channel" and select the latest version from the "Rapid channel".
7. Click "default-pool" from the left navigation menu and set "Number of nodes (per zone)" to "1" in the "Size" section.
8. Feel free to explore other settings. However, do not change any of them as they might impact the examples later in the course.
9. Once you’re happy with your configuration and the estimated monthly cost, click "Create".

It’ll take a couple of minutes to create your cluster.

### Test your GKE cluster <a href="#test-your-gke-cluster" id="test-your-gke-cluster"></a>

The clusters page of your Google Cloud Console shows a high-level overview of the Kubernetes clusters in your project. Feel free to poke around and familiarize yourself with some of the settings.

Click the three dots to the right of your new cluster to reveal the Connect option. The command-line access section gives you a long `gcloud` command to configure `kubectl` to talk to your cluster.

Copy the command to your clipboard and run it in the terminal.

```powershell
$ gcloud container clusters get-credentials gke-tkb --region...
Fetching cluster endpoint and auth data.
kubeconfig entry generated for gke-tkb.

// Fetch cluster endpoints and generate a kubeconfig entry
```

When the command is complete, run the following `kubectl get nodes` command to list the nodes in the cluster.

```powershell
$ kubectl get nodes
NAME                            STATUS      ROLES        VERSION
gke-gke-tkb-default...h2gp      Ready       <none>       v1.29.0-gke.1381000
gke-gke-tkb-default...l29b      Ready       <none>       v1.29.0-gke.1381000
gke-gke-tkb-default...qzv6      Ready       <none>       v1.29.0-gke.1381000

// Get the list of nodes
```

The node names and Kubernetes version should relate to the GKE cluster you created.

Notice how all nodes have `<none>` under the `ROLES` column. This is because GKE is a hosted platform and only lets you see worker nodes. GKE manages the control plane nodes and hides them from you.

Congratulations! You have a production-grade Kubernetes cluster that you can use in most of the hands-on examples. You’ll build a different cluster for the WebAssembly chapter.

> **Warning:** Be sure to delete the cluster as soon as you finish using it to avoid unwanted costs. We recommend deleting the cluster daily and creating a new one each time you need a cluster. Doing this will obviously delete anything created on the cluster you delete.
