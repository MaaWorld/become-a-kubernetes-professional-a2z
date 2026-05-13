---
icon: kubernetes
---

# Introduction to Threat Modeling

Security is more important than ever, and Kubernetes is no exception. Fortunately, there's a lot we can do to secure Kubernetes, and we'll see some ways in the next chapter. However, before doing that, it's a good idea to model some of the common threats.

### Threat modeling <a href="#threat-modeling" id="threat-modeling"></a>

**Threat modeling** is the process of identifying vulnerabilities so we can put measures in place to prevent and mitigate them. This chapter introduces the popular _STRIDE_ model and shows how we can apply it to Kubernetes.

**STRIDE** defines six potential threat categories:

* Spoofing
* Tampering
* Repudiation
* Information disclosure
* Denial of service
* Elevation of privilege

While the model is good and provides a structured way to assess things, no model guarantees that it will cover all threats.

For the rest of this chapter, we’ll look at each of the six threat categories. For each one, we’ll give a quick description and then look at some of the ways it applies to Kubernetes.

{% stepper %}
{% step %}
#### Spoofing

Spoofing is pretending to be somebody else with the aim of gaining extra privileges.

Let’s look at some of the ways Kubernetes prevents different types of spoofing.

#### Securing communications with the API server <a href="#securing-communications-with-the-api-server" id="securing-communications-with-the-api-server"></a>

Kubernetes comprises lots of small components that work together. These include the API server, controller manager, scheduler, cluster store, and others. It also includes node components such as the `kubelet` and container runtime. Each has its own privileges that allow it to interact with and modify the cluster. Even though Kubernetes implements a least-privilege model, spoofing the identity of any of these can cause problems.

If we read the RBAC and API security chapter, we’ll know that Kubernetes requires all components to authenticate via cryptographically signed certificates (mTLS). This is good, and Kubernetes makes it easy by automatically rotating certificates. However, we must consider the following:

1. A typical Kubernetes installation auto-generates a self-signed certificate authority (CA) that issues certificates to all cluster components. While this is better than nothing, it’s not enough for production environments on its own.
2. Mutual TLS (mTLS) is only as secure as the CA issuing the certificates. Compromising the CA can render the entire mTLS layer ineffective. With this in mind, it’s vital we keep the CA secure!

A good practice is to ensure that certificates issued by the internal Kubernetes CA are only used and trusted within the Kubernetes cluster. This requires careful approval of certificate signing requests, as well as ensuring the Kubernetes CA doesn’t get added as a trusted CA for any systems outside the cluster.

As mentioned in previous chapters, all internal and external requests to the API server are subject to authentication and authorization checks. As a result, the API server needs a way to authenticate (trust) internal and external sources. A good way to do this is to have two trusted key pairs:

* One for authenticating internal systems
* A second for authenticating external systems

In this model, we’d use the cluster’s self-signed CA to issue keys to internal systems. We’d then configure Kubernetes to trust one or more trusted third-party CAs for external systems.

#### Securing Pod communications <a href="#securing-pod-communications" id="securing-pod-communications"></a>

In addition to spoofing access to the cluster, there’s also the threat of spoofing app-to-app communications. In Kubernetes, this can be when one Pod spoofs another. Fortunately, Pods can have certificates to authenticate their identity.

Every Pod has an associated _ServiceAccount_ that is used to provide an identity for the Pod. This is achieved by automatically mounting a service account token into every Pod as a _Secret_. Two points to note:

1. The service account token allows access to the API server.
2. Most Pods probably don’t need to access the API server.

With these two points in mind, we should set `automountServiceAccountToken` to `false` for Pods that don’t need to communicate with the API server. The following Pod manifest shows how to do this.

```html
// index.html

apiVersion: v1
kind: Pod
metadata:
  name: service-account-example-pod
spec:
  serviceAccountName: some-service-account
  automountServiceAccountToken: false       # <<==== This line
  <Snip>
  
  // YAML snippet defining a Pod
```

If the Pod does need to talk to the API server, the following non-default configurations are worth exploring:

* `expirationSeconds`
* `audience`

These let us force a time when the token will expire and restrict the entities it works with.

```yaml
// index.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: my-pod
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 3600     <<==== This line
          audience: vault             <<==== And this one

// YAML snippet defining a Pod that doesn't need to talk to the API server
```

The above example, inspired by the official Kubernetes docs, sets an expiry period of one hour and restricts it to the vault audience in a projected volume.
{% endstep %}

{% step %}
#### Tampering

Tampering is the act of changing something in a malicious way to cause one of the following:

* **Denial-of-service**: Tampering with the resource to make it unusable
* **Elevation of privilege**: Tampering with a resource to gain additional privileges

Tampering can be hard to avoid, so a common countermeasure is to make it obvious when something has been tampered with. A common non-Kubernetes example is packaging medication—most over-the-counter drugs are packaged with tamper-proof seals that make it obvious if the product has been tampered with.

#### Tampering with Kubernetes components <a href="#tampering-with-kubernetes-components" id="tampering-with-kubernetes-components"></a>

Tampering with any of the following Kubernetes components can cause problems:

* etcd
* Configuration files for the API server, controller-manager, scheduler, etcd, and kubelet
* Container runtime binaries
* Container images
* Kubernetes binaries

Generally speaking, tampering happens either _in transit_ or _at rest_. In transit refers to data while it is being transmitted over the network, whereas at rest refers to data stored in memory or on disk.

TLS is a great tool for protecting against _in-transit_ tampering as it provides built-in integrity guarantees that warn us when data has been tampered with.

#### Data security in Kubernetes <a href="#data-security-in-kubernetes" id="data-security-in-kubernetes"></a>

The following recommendations can also help prevent tampering with data when it is _at rest_ in Kubernetes:

* Restrict access to the servers that are running Kubernetes components, especially control plane components.
* Restrict access to repositories that store Kubernetes configuration files.
* Only perform remote bootstrapping over SSH (remember to keep our SSH keys safe).
* Always run SHA-2 checksums against downloads.
* Restrict access to our image registry and associated repositories.

This isn’t an exhaustive list. However, implementing it will significantly reduce the chances of our data being tampered with while at rest.

As well as the items listed, it’s good production hygiene to configure auditing and alerting for important binaries and configuration files. If configured and monitored correctly, these can help detect potential tampering attacks.

The following example uses a common Linux audit daemon to audit access to the `docker` binary. It also audits attempts to change the binary’s file attributes.

```powershell
$ auditctl -w /usr/bin/docker -p wxa -k audit-docker
```

#### Tampering with applications running on Kubernetes <a href="#tampering-with-applications-running-on-kubernetes" id="tampering-with-applications-running-on-kubernetes"></a>

Malicious actors will also target application components, as well as infrastructure components.

A good way to prevent a live Pod from being tampered with is setting its filesystems to _read-only_. This guarantees filesystem immutability and we can configure it via the `securityContext` section of a Pod manifest file.

We can make a container’s root filesystem read-only by setting the `readOnlyRootFilesystem` property to `true`. We can do the same for other container filesystems via the `allowedHostPaths` property.

```yaml
// index.yaml

apiVersion: v1
kind: Pod
metadata:
  name: readonly-test
spec:
  securityContext:
    readOnlyRootFilesystem: true     <<==== R/O root filesystem
    allowedHostPaths:                <<==== Make anything below 
      - pathPrefix: "/test"          <<==== this mount point
        readOnly: true               <<==== read-only (R/O)
<Snip>

// YAML snippet consisting of a Pod
```

The above YAML shows how to configure both settings in a Pod manifest. In the example, the `allowedHostPaths` section makes sure anything mounted beneath `/test` will be read-only.
{% endstep %}

{% step %}
#### Repudiation

At a very high level, **repudiation** creates doubt about something. Non-repudiation provides proof about something. In the context of information security, non-repudiation is _proving_ certain individuals carried out certain actions.

Digging a little deeper, non-repudiation includes the ability to prove:

* What happened?
* When it happened?
* Who made it happen?
* Where it happened?
* Why it happened?
* How it happened?

Answering the last two can be the hardest and usually requires the correlation of several events over a period of time.

#### How to acheive non-repudiation <a href="#how-to-acheive-non-repudiation" id="how-to-acheive-non-repudiation"></a>

Auditing Kubernetes API server events can help answer these questions. The following is an example of an API server audit event (we may need to enable auditing on our API server).

```javascript
// index.js

{
  "kind":"Event",
  "apiVersion":"audit.k8s.io/v1",
  "metadata":{ "creationTimestamp":"2022-11-11T10:10:00Z" },
  "level":"Metadata",
  "timestamp":"2022-11-11T10:10:00Z",
  "auditID":"7e0cbccf-8d8a-4f5f-aefb-60b8af2d2ad5",
  "stage":"RequestReceived",
  "requestURI":"/api/v1/namespaces/default/persistentvolumeclaims",
  "verb":"list",
  "user": {
    "username":"fname.lname@example.com",
    "groups":[ "system:authenticated" ]
  },
  "sourceIPs":[ "123.45.67.123" ],
  "objectRef": {
    "resource":"persistentvolumeclaims",
    "namespace":"default",
    "apiVersion":"v1"
  },
  "requestReceivedTimestamp":"2022-11-11T10:10:00.123456Z",
  "stageTimestamp":"2022-11-11T10:10:00.123456Z"
}

// API server audit event
```

The API server isn’t the only component we should audit for non-repudiation. At a minimum, we should collect audit logs from container runtimes, kubelets, and the applications running on our cluster. We should also audit non-Kubernetes infrastructure, such as network firewalls.

As soon as we start auditing multiple components, we’ll need a centralized location to store and correlate events. A common way to do this is by deploying an agent to all nodes via a DaemonSet. The agent collects logs (runtime, kubelet, application, etc) and ships them to a secure central location.

If we do this, the centralized log store must be secure. If it isn’t, we won’t be able to trust the logs, and their contents can be _repudiated_.

To provide non-repudiation relative to tampering with binaries and configuration files, it might be useful to use an audit daemon that watches for write actions on certain files and directories on our Kubernetes control plane nodes and worker nodes. For example, earlier in the chapter, we saw a way to enable auditing of changes to the `docker` binary. With this enabled, starting a new container with the `docker run` command will generate an event like this:

```powershell
type=SYSCALL msg=audit(1234567890.123:12345): arch=abc123 syscall=59 success=yes \\
exit=0 a0=12345678abca1=0 a2=abc12345678 a3=a items=1 ppid=1234 pid=12345 auid=0 \\
uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=1 comm="docker" \\
exe="/usr/bin/docker" subj=system_u:object_r:container_runtime_exec_t:s0 \\
key="audit-docker" type=CWD msg=audit(1234567890.123:12345):  cwd="/home/firstname"\\
type=PATH msg=audit(1234567890.123:12345): item=0 name="/usr/bin/docker"\\
 inode=123456 dev=fd:00 mode=0100600 ouid=0 ogid=0 rdev=00:00...
```

When combined and correlated with Kubernetes’ audit features, audit logs like this create a comprehensive and trustworthy picture that cannot be repudiated.
{% endstep %}

{% step %}
#### Information Disclosure

Information disclosure is when sensitive data is leaked. Common examples include hacked data stores and APIs that unintentionally expose sensitive data.

#### Protecting cluster data <a href="#protecting-cluster-data" id="protecting-cluster-data"></a>

The entire configuration of a Kubernetes cluster is stored in the cluster store (usually etcd). This includes network and storage configuration, passwords, the cluster CA, and more. This makes the cluster store a prime target for information disclosure attacks.

As a minimum, we should limit and audit access to the nodes hosting the cluster store. As we’ll see in the next paragraph, gaining access to a cluster node can allow the logged-on user to bypass some security layers.

Kubernetes v1.7 introduced encryption of Secrets but doesn’t enable it by default. Even when this becomes the default, the _data encryption key (DEK)_ is stored on the same node as the Secret! This means gaining access to a node lets us bypass encryption. This is especially worrying for nodes that host the cluster store (etcd nodes).

Fortunately, Kubernetes 1.11 enabled a beta feature that lets us store _key encryption keys (KEK)_ outside our Kubernetes cluster. These keys are used to encrypt and decrypt data encryption keys and should be safely guarded. We should seriously consider Hardware Security Modules (HSM) or cloud-based Key Management Stores (KMS) for storing our key encryption keys.

Keep an eye on upcoming versions of Kubernetes for further improvements to encryption of Secrets.

#### Protecting data in Pods <a href="#protecting-data-in-pods" id="protecting-data-in-pods"></a>

As previously mentioned, Kubernetes has an API resource called a Secret that is the preferred way to store and share sensitive data such as passwords. For example, a front-end container accessing an encrypted back-end database can have the key to decrypt the database mounted as a Secret. This is far better than storing the decryption key in a plain text file or environment variable.

It is also common to store data and configuration information outside Pods and containers in PersistentVolumes and ConfigMaps. If the data on these is encrypted, we should store the keys for decrypting them in Secrets.

Despite all of this, we must consider the caveats outlined in the previous section regarding Secrets and how their encryption keys are stored. We don’t want to do the hard work of locking the house but leaving the keys in the door.
{% endstep %}

{% step %}
#### Denial-of-Service (DoS)

Denial-of-service (DoS) is about making something unavailable.

There are many types of DoS attacks, but a well-known variation is overloading a system to the point it can no longer service requests. In the Kubernetes world, a potential attack might be overloading the API server so that cluster operations grind to a halt (even internal systems use the API server to communicate).

Let’s look at some potential Kubernetes systems that might be targets of DoS attacks, as well as some ways to protect and mitigate them.

#### Protecting cluster resources against DoS attacks <a href="#protecting-cluster-resources-against-dos-attacks" id="protecting-cluster-resources-against-dos-attacks"></a>

It’s a time-honored best practice to replicate essential services on multiple nodes for high availability (HA). Kubernetes is no different, and we should run multiple control plane nodes in an HA configuration for our production environments. Doing this prevents any control plane node from becoming a single point of failure. In relation to certain types of DoS attacks, an attacker may need to attack more than one control plane node to have a meaningful impact.

We should also replicate control plane nodes across availability zones. This may prevent a DoS attack on the network of a particular availability zone from taking down our entire control plane.

The same principle applies to worker nodes. Having multiple worker nodes not only allows the scheduler to spread our applications over multiple availability zones, but it may also render DoS attacks on any single node or zone ineffective (or less effective).

We should also configure appropriate limits for the following:

* Memory
* CPU
* Storage

Limits like these can help prevent essential system resources from being starved, therefore preventing potential DoS.

Limiting _Kubernetes objects_ can also be a good practice. This includes limiting things such as the number of ReplicaSets, Pods, Services, Secrets, and ConfigMaps in a particular Namespace.

Here’s an example manifest that limits the number of Pod objects in the `skippy` Namespace to 100.

```
// index.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
  namespace: skippy
spec:
  hard:
    pods: "100"

// 
```

One more feature — `podPidsLimit` — restricts the number of processes a Pod can create.

Assume a Pod is the target of a fork bomb attack where a rogue process attempts to bring the system down by creating enough processes to consume all system resources. If we’ve configured the Pod with `podPidsLimit` to restrict the number of processes the Pod can create, we’ll prevent it from exhausting the node’s resources and confine the attack’s impact to the Pod. Kubernetes will normally restart a Pod if it exhausts its `podPidsLimit`.

This also ensures a single Pod doesn’t exhaust the PID range for all the other Pods on the node, including the kubelet. However, setting the correct value requires a reasonable estimate of how many Pods will run simultaneously on each node, and we can easily over or under-allocate PIDs to each pod without a ballpark estimate.

#### Protecting the API Server against DoS attacks <a href="#protecting-the-api-server-against-dos-attacks" id="protecting-the-api-server-against-dos-attacks"></a>

The API server exposes a RESTful interface over a TCP socket. This makes it a target for botnet-based DoS attacks.

The following may be helpful in either preventing or mitigating such attacks:

* Highly available control plane nodes — multiple replicas of the API server running on multiple nodes across multiple availability zones
* Monitoring and alerting on API server requests based on some thresholds
* Using things like firewalls to limit API server exposure to the internet

As well as botnet DoS attacks, an attacker may also attempt to spoof a user or other control plane service to cause an overload. Fortunately, Kubernetes has robust authentication and authorization controls to prevent spoofing. However, even with a robust RBAC model, we must safeguard access to accounts with high privileges.

#### Protecting the cluster store against DoS attacks <a href="#protecting-the-cluster-store-against-dos-attacks" id="protecting-the-cluster-store-against-dos-attacks"></a>

Kubernetes stores cluster configuration in etcd. This makes it vital that etcd be available and secure. The following recommendations help accomplish this:

* Configure an HA etcd cluster with either three or five nodes.
* Configure monitoring and alerting of requests to etcd.
* Isolate etcd at the network level so that only members of the control plane can interact with it.

A default installation of Kubernetes installs etcd on the same servers as the rest of the control plane. This is fine for development and testing. However, large production clusters should seriously consider a dedicated etcd cluster. This will provide better performance and greater resilience.

On the performance front, etcd is the most common choking point for large Kubernetes clusters. With this in mind, we should perform testing to ensure the infrastructure it runs on is capable of sustaining performance at scale — a poorly performing etcd can be as bad as an etcd cluster under a sustained DoS attack. Operating a dedicated etcd cluster also provides additional resilience by protecting it from other parts of the control plane that might be compromised.

Monitoring and alerting of etcd should be based on some thresholds, and a good place to start is by monitoring etcd log entries.

#### Protecting application components against DoS attacks <a href="#protecting-application-components-against-dos-attacks" id="protecting-application-components-against-dos-attacks"></a>

Most Pods expose their main service on the network, and without additional controls in place, anyone with access to the network can perform a DoS attack on the Pod. Fortunately, Kubernetes provides Pod resource request limits to prevent such attacks from exhausting Pod and node resources. As well as these, the following will be helpful:

* Define Kubernetes network policies to restrict Pod-to-Pod and Pod-to-external communications.
* Utilize mutual TLS and API token-based authentication for application-level authentication (reject any unauthenticated requests).

For defense in depth, we should also implement application layer authorization policies that implement the least privilege.

<figure><img src="../.gitbook/assets/image (16).png" alt="" width="558"><figcaption><p>The process after applying network layer and application layer authorization policies</p></figcaption></figure>

The above figure shows how these can be combined to make it hard for an attacker to successfully carry out a denial-of-service (DoS) attack on an application.
{% endstep %}

{% step %}
#### Elevation of Privilege

**Privilege escalation** is gaining higher access than what is granted. The aim is to cause damage or gain unauthorized access.

Let’s look at a few ways to prevent this in a Kubernetes environment.

#### Protecting the API server <a href="#protecting-the-api-server" id="protecting-the-api-server"></a>

Kubernetes offers several authorization modes that help safeguard access to the API server. These include:

* Role-based Access Control (RBAC)
* Webhook
* Node

We should run multiple authorizers at the same time. For example, it’s common to use the _RBAC_ and _node_ authorizers.

**RBAC mode** lets us restrict API operations to subsets of users. These _users_ can be regular user accounts or system services. The idea is that all requests to the API server must be authenticated and authorized. Authentication ensures that requests come from a validated user, whereas authorization ensures the validated user can perform the requested operation. For example, can _Mia create Pods_? In this example, _Mia_ is the user, _create_ is the operation, and _Pods_ is the resource. Authentication makes sure that it really is Mia making the request, and authorization determines if she’s allowed to create Pods.

**Webhook mode** lets us offload authorization to an external REST-based policy engine. However, it requires additional effort to build and maintain the external engine. It also makes the external engine a potential single point of failure for every request to the API server. For example, if the external webhook system becomes unavailable, we may be unable to make any requests to the API server. With this in mind, we should rigorously vet and implement any webhook authorization service.

**Node authorization** is about authorizing API requests made by `kubelets` (Nodes). The requests made to the API server by `kubelets` are obviously different from those generally made by regular users, and the node authorizer is designed to help with this.
{% endstep %}
{% endstepper %}

***
