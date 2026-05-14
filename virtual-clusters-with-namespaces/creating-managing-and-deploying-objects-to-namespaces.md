---
description: >-
  Learn how to create, inspect, and delete namespaces. Couple of ways to ways to
  deploy objects to namespaces.
---

# Creating, Managing and Deploying Objects to Namespaces

### Namespaces as a resource <a href="#namespaces-as-a-resource" id="namespaces-as-a-resource"></a>

Namespaces are first-class resources in the `core v1` API group. This means they’re stable, well-understood, and have been around for a long time. It also means we can work with them imperatively and declaratively. We’ll do both.

Use the following widget to execute all your commands.

```yml
// shield-ns.yml

kind: Namespace
apiVersion: v1
metadata:
  name: shield
  labels:
    env: marvel
```

Run the following imperative command to create a new Namespace called `hydra`.

```powershell
$ kubectl create ns hydra
namespace/hydra created

// Create a Namespace
```

Now, create one declaratively from the `shield-ns.yml` YAML file. It’s a simple file defining a single Namespace called `shield`.

```powershell
// index.yaml

kind: Namespace
apiVersion: v1
metadata:
  name: shield
  labels:
    env: marvel
    
// Contents of shield-ns.yml file
```

Create it with the following command.

```powershell
$ kubectl apply -f shield-ns.yml
namespace/shield created

// Create the namespace
```

List all Namespaces to see the two new ones we created.

```powershell
$ kubectl get ns
NAME         STATUS   AGE
<Snip>
hydra        Active   49s
shield       Active   3s

// Get all namespaces
```

If we know anything about the Marvel Cinematic Universe, we’ll know Shield and Hydra are bitter enemies and should never share the same cluster with only Namespaces separating them.

Delete the `hydra` Namespace.

```powershell
$ kubectl delete ns hydra
namespace "hydra" deleted

// Delete the namespace
```

### Configure `kubectl` for a specific Namespace <a href="#configure-kubectl-for-a-specific-namespace" id="configure-kubectl-for-a-specific-namespace"></a>

When working with Namespaces, we’ll quickly realize it’s painful having to add the `-n` or `--namespace` flag on all `kubectl` commands. A better way is to set our kubeconfig to automatically run commands against a specific Namespace.

Run the following command to configure the kubeconfig to run all future `kubectl` commands against the `shield` Namespace.

```powershell
$ kubectl config set-context --current --namespace shield
Context "tkb" modified.

// Change the context
```

Run a few simple `kubectl get` commands to test it works. The `shield` Namespace is empty, so the commands won’t return any objects.

***

## Deploying Objects to Namespaces <a href="#page-title" id="page-title"></a>

As previously mentioned, most objects are Namespaced, and Kubernetes deploys new objects to the `default` Namespace unless we specify otherwise.

### How to deploy objects to namespaces <a href="#how-to-deploy-objects-to-namespaces" id="how-to-deploy-objects-to-namespaces"></a>

There are two ways to deploy objects to specific Namespaces:

* Imperatively
* Declaratively

To do it imperatively, add the `-n` or `--namespace` flag to commands. To do it declaratively, we specify the Namespace in the objects YAML manifest.

Let’s deploy an app to the `shield` Namespace using the declarative method.

The application is defined in the `app.yml` file in the `namespaces` folder of the book’s GitHub repo. It defines three objects: a `ServiceAccount`, a `Service`, and a `Pod`. The following YAML extract shows all three objects targeted at the `shield` Namespace.

Don’t worry if you don’t understand everything in the YAML, you only need to know it defines three objects and targets each one at the `shield` Namespace.

```yaml
// index.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: shield     <<==== Namespace
  name: default
---
apiVersion: v1
kind: Service
metadata:
  namespace: shield     <<==== Namespace
  name: the-bus
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    env: marvel
---
apiVersion: v1
kind: Pod
metadata:
  namespace: shield     <<==== Namespace
  name: triskelion
<Snip>

// Contents of the app.yml file
```

Use the following widget to execute your commands.

```yml
// app.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: shield
  name: default

---
apiVersion: v1
kind: Service
metadata:
  namespace: shield
  name: the-bus
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    env: marvel
    
---
apiVersion: v1
kind: Pod
metadata:
  namespace: shield
  name: triskelion
  labels:
    env: marvel
spec:
  containers:
  - image: nigelpoulton/k8sbook:shield-01
    name: bus-ctr
    ports:
    - containerPort: 8080
    imagePullPolicy: Always
```

Deploy it with the following command. Don’t worry if you get a warning about a missing annotation for the `ServiceAccount`.

```powershell
$ kubectl apply -f app.yml
serviceaccount/default configured
service/the-bus configured
pod/triskelion created

// Create the objects
```

Run a few commands to verify all three objects are in the `shield` Namespace. We don’t need to add the `-n shield` flag if we configured `kubectl` to automatically target the `shield` Namespace.

```powershell
$ kubectl get pods -n shield
NAME         READY   STATUS    RESTARTS   AGE
triskelion   1/1     Running   0          48s

$ kubectl get svc -n shield
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
the-bus   LoadBalancer   10.43.30.174   localhost     8080:31112/TCP   52s

// Get the status of the Pod and Service
```

Now that the app is deployed, use `curl` or the browser to connect to it. Just point the browser or `curl` command to the value in the `EXTERNAL-IP` column on port `8080`. If it looks like the following example, we’ll connect to `localhost:8080`.

On the platform, we can connect to the application by running the following command first:

```powershell
kubectl port-forward svc/the-bus -n shield 8080:8080 --address 0.0.0.0
```

Now run the following command in a new terminal or click the link provided above the terminal. It will redirect you to a new browser tab, and you will notice that your application has been deployed.

```powershell
$ curl localhost:8080

<!DOCTYPE html>
<html>
<head>
    <title>AOS</title>
    <Snip>
    
// Get the application
```

Congratulations! We’ve created a Namespace and deployed an app to it. Connecting to the app is no different from connecting to an app in the default Namespace.

### Clean up <a href="#clean-up" id="clean-up"></a>

If you’re working on a cluster created on a cloud distribution, you might want to clean up your cluster. The following commands will clean up the cluster and revert the `kubeconfig` file to use the `default` Namespace.

Delete the `shield` Namespace. This will automatically delete the `Pod`, `Service`, and `ServiceAccount`, and it may take a few seconds to complete.

```powershell
$ kubectl delete ns shield
namespace "shield" deleted

// Delete the Namespace objects
```

Reset the kubeconfig so it uses the `default` Namespace. If we don’t do this, future commands will run against the deleted `shield` Namespace and return no results.

```powershell
$ kubectl config set-context --current --namespace default
Context "tkb" modified.

// Change the context
```
