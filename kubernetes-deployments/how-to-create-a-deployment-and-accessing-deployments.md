---
description: >-
  Learn how we can create a Kubernetes Deployment using a YAML file and how to
  access Deployments.
icon: kubernetes
---

# How to Create a Deployment and Accessing Deployments

## Declaratively create a Deployment <a href="#declaratively-create-a-deployment" id="declaratively-create-a-deployment"></a>

To create a Deployment, we’ll use the `deploy.yml` file shown in the following snippet. It defines a single-container Pod wrapped in a Deployment. It’s annotated and snipped to draw attention to the parts we’ll focus on.

```yaml
// index.yaml

kind: Deployment
apiVersion: apps/v1
metadata:
  name: hello-deploy       <<==== Deployment name (must be valid DNS name)
spec:
  replicas: 10             <<==== Number of Pod replicas to deploy & manage
  selector:                
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300    
  minReadySeconds: 10
  strategy:                <<==== This block defines rolling update settings
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:                <<==== Below here is the Pod template
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
        
// Contents of the deploy.yml file
```

There’s a lot going on in the file, so let’s explain the most important bits.

The first two lines, i.e., **line 1** and **line 2**, tell Kubernetes to create a Deployment object based on the version of the Deployment resource defined in the `apps/v1` API.

* The `metadata` section, i.e., **line 3** and **line 4**, names the Deployment `hello-deploy`. We should always give objects valid DNS names. This means we should only use alphanumerics, the dot, and the dash in object names.
* The `spec` section, i.e., **lines 5–27**, is where most of the action happens.
* `spec.replicas`, i.e., **line 6**, asks for 10 Pod replicas. In this case, the ReplicaSet controller will create 10 replicas of the Pod defined in the `spec.template` section.
* `spec.selector`, i.e., **lines 7–9**, is a list of labels that Pods need to have for the Deployment and ReplicaSet controllers to manage them. This label selector has to match the Pod labels in the Pod template block (`spec.template.metadata.labels`). In this example, both specify the `app=hello-world` label.
* `spec.revisionHistoryLimit`, i.e., **line 10**, tells Kubernetes to keep the previous five ReplicaSets so we can roll back to the last five versions. Keeping more gives us more rollback options, but keeping too many can bloat the object and cause problems on large clusters with lots of releases.
* `spec.progressDeadlineSeconds`, i.e., **line 11**, tells Kubernetes to give each new replica a five-minute start window before reporting the update as stalled. The counter is reset for each replica, meaning each replica has its own five-minute window to come up properly (progress).
* `spec.strategy`, i.e., **lines 13–17,** tell the Deployment controller how to update the Pods when a rollout occurs. We’ll explain these settings later in the chapter when we perform a rollout.

Finally, everything below `spec.template` defines the Pod this Deployment will manage. This example defines a single-container Pod using the `nigelpoulton/k8sbook:1.0` image.

Use the following widget to run your commands.

```yaml
// deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: 128Mi
            cpu: 0.1
```

Run the following command to create the Deployment on our cluster.

```powershell
$ kubectl apply -f deploy.yml
deployment.apps/hello-deploy created

// Creating the Deployment object
```

> **Note:** All `kubectl` commands include the necessary authentication tokens from our kubeconfig file.

At this point, the Deployment configuration is persisted to the cluster store as a record of intent, and Kubernetes has scheduled 10 replicas to healthy worker nodes. The Deployment and ReplicaSet controllers are also running in the background, watching the state of play and eager to perform their reconciliation magic.

***

## Accessing Deployments <a href="#page-title" id="page-title"></a>

Learn how to access Deployments.

Let’s have a look at the `deploy.yml` file from the previous lesson. We’ll execute all these commands from the below terminal. Click the "Run" button to create a cluster.

```yml
// deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: 128Mi
            cpu: 0.1
```

### Inspecting Deployments <a href="#inspecting-deployments" id="inspecting-deployments"></a>

We can use the normal `kubectl get` and `kubectl describe` commands to see details of Deployments and ReplicaSets.

```powershell
$ kubectl get deploy hello-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-deploy   10/10   10           10          105s

$ kubectl describe deploy hello-deploy
Name:                   hello-deploy
Namespace:              default
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=hello-world
Replicas:               10 desired | 10 updated | 10 total | 10 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  app=hello-world
  Containers:
   hello-pod:
    Image:        nigelpoulton/k8sbook:1.0
    Port:         8080/TCP
<SNIP>
OldReplicaSets:  <none>
NewReplicaSet:   hello-deploy-54f5d46964 (10/10 replicas created)
<Snip>

// Get the attributes of the Deployment object
```

The outputs are trimmed for readability, but take a minute to examine them, as they contain a lot of information that will reinforce what we’ve learned.

As mentioned earlier, Deployments automatically create associated ReplicaSets. Verify this with the following command.

```powershell
$ kubectl get rs
NAME                      DESIRED   CURRENT   READY   AGE
hello-deploy-54f5d46964   10        10        10      3m45s

// Get all ReplicaSets
```

We only have one ReplicaSet as we’ve only performed an initial rollout. However, we can see the ReplicaSet’s name matches the Deployment’s name with a hash added to the end. This is a crypto-hash of the Pod template section of the Deployment manifest (everything below `spec.template`). We’ll see this shortly, but making changes to the Pod template section initiates a rollout and creates a new ReplicaSet with a hash of the updated Pod template.

We can get more detailed information about the ReplicaSet with a `kubectl describe` command. Our ReplicaSet will have a different name.

> **Note:** Replace the ReplicaSet name with the name obtained by running the above command.

```powershell
$ kubectl describe rs hello-deploy-54f5d46964
Name:           hello-deploy-54f5d46964
Namespace:      default
Selector:       app=hello-world,pod-template-hash=54f5d46964
Labels:         app=hello-world
                pod-template-hash=54f5d46964
Annotations:    deployment.kubernetes.io/desired-replicas: 10
                deployment.kubernetes.io/max-replicas: 11
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hello-deploy
Replicas:       10 current / 10 desired
Pods Status:    10 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hello-world
           pod-template-hash=54f5d46964
  Containers:
   hello-pod:
    Image:        nigelpoulton/k8sbook:1.0
    Port:         8080/TCP
<Snip>

// Get the attributes of the hello-deploy ReplicaSet
```

Notice how the output is similar to the Deployment output. This is because the Deployment dictates the configuration of ReplicaSets, and ReplicaSet information gets rolled up into the Deployment. The ReplicaSet’s status (observed state) also gets rolled up into the Deployment status.
