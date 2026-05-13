---
icon: kubernetes
---

# Pod Security

In this lesson, we will look at a few technologies that help reduce the risk of elevation of privilege attacks against Pods and containers. We’ll look at the following:

* Preventing processes from running as root
* Dropping capabilities
* Filtering syscalls
* Preventing privilege escalation

As we proceed through these sections, it’s important to remember that a Pod is just an execution environment for one or more containers. Some of the terminology used will refer to Pods and containers interchangeably, but usually we will mean container.

### Do not run processes as root <a href="#do-not-run-processes-as-root" id="do-not-run-processes-as-root"></a>

The _root_ user is the most powerful user on a Linux system and is always User ID 0 (UID 0). This means running application processes as root is almost always a bad idea as it grants the application process full access to the container. This is made even worse by the fact that the root user of a container sometimes has unrestricted root access to the host system as well. If that doesn’t make us afraid, nothing will!

Fortunately, Kubernetes allows us to force container processes to run as unprivileged non-root users. The following Pod manifest configures all containers that are part of this Pod to run processes as UID 1000. If the Pod has multiple containers, all container processes will run as UID 1000.

```yaml
// index.yaml

apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:      <<==== Applies to all containers in this Pod
    runAsUser: 1000     <<==== Non-root user
  containers:
  - name: demo
    image: example.io/simple:1.0
    
// YAML defining a Pod
```

The `runAsUser` property is one of many settings that fall under the category of _PodSecurityContext_ (`spec.securityContext`).

It’s possible for two or more Pods to be configured with the same `runAsUser` UID. When this happens, the containers from both Pods will run with the same security context and potentially have access to the same resources. This _might_ be fine if they are replicas of the same Pod. However, there’s a high chance this will cause problems if they’re not replicas. For example, two different containers with R/W access to the same volume can cause data corruption (both writing to the same dataset without coordinating write operations). Shared security contexts also increase the possibility of a compromised container tampering with a dataset it shouldn’t have access to.

With this in mind, it is possible to use the `securityContext.runAsUser` property at the container level instead of at the Pod level:

```yaml
// index.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:      <<==== Applies to all containers in this Pod
    runAsUser: 1000     <<==== Non-root user
  containers:
  - name: demo
    image: example.io/simple:1.0
    securityContext:
      runAsUser: 2000   <<==== Overrides the Pod-level setting
      
// YAML defining a Pod
```

This example sets the UID to 1000 at the Pod level but overrides it at the container level so that processes in the `demo` container run as UID 2000. Unless otherwise specified, all other containers in the Pod will use UID 1000.

A couple of other things that might help get around the issue of multiple Pods and containers using the same UID include:

* User namespaces
* Maintaining a map of UID usage

User namespaces is a Linux kernel technology that allows a process to run as root within a container but run as a different user outside the container. For example, a process can run as UID 0 (the root user) inside the container but get mapped to UID 1000 on the host. This can be a good solution for processes that need to run as root inside the container. However, we should check if it is fully supported by our version of Kubernetes and our container runtime.

### Capability dropping <a href="#capability-dropping" id="capability-dropping"></a>

While most applications don’t need the complete set of root capabilities, they usually require more capabilities than a typical non-root user.

What we need is a way to grant the exact set of privileges a process requires in order to run. Time for a quick bit of background!

We’ve already said that the root user is the most powerful user on a Linux system. However, its power is a combination of lots of small privileges that we call _capabilities_. For example, the `SYS_TIME` capability allows a user to set the system clock, whereas the `NET_ADMIN` capability allows a user to perform network-related operations such as modifying the local routing table and configuring local interfaces. The root user holds every _capability_ and is, therefore, extremely powerful.

Having a modular set of capabilities allows us to be extremely granular when granting permissions. Instead of an all-or-nothing (root --vs-- non-root) approach, we can grant a process the exact set of capabilities required.

There are currently over 30 capabilities, and choosing the right ones can be daunting. With this in mind, many container runtimes implement a set of sensible defaults that allow most processes to run without _leaving all the doors open_. While sensible defaults like these are better than nothing, they’re often not good enough for production environments.

A common way to find the absolute minimum set of capabilities an application requires, is to run it in a test environment with all capabilities dropped. This causes the application to fail and log messages about the missing permissions. We map those permissions to capabilities, add them to the application’s Pod spec, and run the application again. We rinse and repeat this process until the application runs properly with the minimum set of capabilities.

As good as this is, there are a few things to consider. Firstly, we _must_ perform extensive testing of each application. The last thing we want is a production edge case that we hadn’t accounted for in our test environment. Such occurrences can crash our application in production! Secondly, every application revision requires the same extensive testing against the capability set.

With these considerations in mind, it is vital that we have testing procedures and production release processes that can handle all of this. By default, Kubernetes implements our chosen container runtime’s default set of capabilities (e.g., containerd). However, we can override this as part of a container’s `securityContext` field.

The following Pod manifest shows how to add the `NET_ADMIN` and `CHOWN` capabilities to a container.

```yaml
// index.yaml
apiVersion: v1
kind: Pod
metadata:
  name: capability-test
spec:
  containers:
  - name: demo
    image: example.io/simple:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "CHOWN"]. # <<==== This line
        
// YAML snippet defining a Pod that adds the NET_ADMIN and CHOWN capabilities to a container
```

### Filter syscalls <a href="#filter-syscalls" id="filter-syscalls"></a>

**Seccomp**, short for secure computing, is similar in concept to capabilities but works by filtering syscalls rather than capabilities.

An application asks the Linux kernel to perform an operation by issuing a _syscall_. Seccomp lets us control which syscalls a particular container can make to the host kernel. As with capabilities, we should implement a least privilege model where the only syscalls a container can make are the ones it needs to run.

### Securing Pods with seccomp profiles <a href="#securing-pods-with-seccomp-profiles" id="securing-pods-with-seccomp-profiles"></a>

Seccomp went GA in Kubernetes v1.19, and we can use it in different ways based on the following seccomp profiles:

1. **Non-blocking:** This allows a Pod to run, but records every syscall to an audit log we can use to create a custom profile. The idea is to extensively test our application Pod in a dev/test environment. After that, we’ll have a log file listing every syscall the Pod needs to run. We then use this to create a custom profile that only allows those syscalls (least privilege).
2. **Blocking:** This blocks all syscalls. It’s extremely secure but prevents a Pod from doing anything useful.
3. **Runtime Default:** This forces a Pod to use the seccomp profile defined by its container runtime. This is a common place to start if we still need to create a custom profile. Profiles that ship with container runtimes are designed to be a balance of _usable_ and _secure_. They’re also thoroughly tested.
4. **Custom:** This is a profile that only allows the syscalls our application needs to run. Everything else is blocked. It’s common to extensively test our application in dev/test environment with a non-blocking profile that records all syscalls to an audit log. We then use this log to identify our app’s syscalls and build the customized profile. The danger with this approach is that our app has some edge cases we miss during testing. If this happens, our application can fail in production when it hits an edge case and uses a syscall not captured during testing.

Custom profiles operate the _least privilege_ model and are the preferred approach from a security perspective.

### Prevent privilege escalation by containers <a href="#prevent-privilege-escalation-by-containers" id="prevent-privilege-escalation-by-containers"></a>

The only way to create a new process in Linux is for one process to clone itself and then load new instructions onto the new process. We’re oversimplifying, but the original process is called the _parent_ process, and the copy is called the _child_ process.

By default, Linux allows a _child_ process to claim more privileges than its _parent_. This is usually a bad idea. In fact, we’ll often want a child process to have the same or fewer privileges than its parent. This is especially true for containers, as their security configurations are defined against their initial configuration and not against potentially escalated privileges.

Fortunately, it’s possible to prevent privilege escalation through the `securityContext` property of individual containers, as shown.

```yaml
// index.yaml

apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - name: demo
    image: example.io/simple:1.0
    securityContext:
      allowPrivilegeEscalation: false    # <<==== This line

// YAML snippet defining a Pod
```

### Standardizing Pod Security with PSS and PSA <a href="#standardizing-pod-security-with-pss-and-psa" id="standardizing-pod-security-with-pss-and-psa"></a>

Modern Kubernetes clusters implement two technologies to help enforce Pod security settings:

* **Pod Security Standards (PSS)** are policies that specify required Pod security settings.
* **Pod Security Admission (PSA)** enforces one or more PSS policies when Pods are created.

Both work together to effectively centralize the enforcement of Pod security—we choose which PSS policies to apply, and PSA enforces them.

### Pod Security Standards (PSS) <a href="#pod-security-standards-pss" id="pod-security-standards-pss"></a>

Every Kubernetes cluster gets the following three PSS _policies_ that are maintained and kept up-to-date by the community:

* `Privileged`: It is a wide-open allow-all policy.
* `Baseline`: It implements sensible defaults. It’s more secure than the _privileged_ policy but less secure than restricted.
* `Restricted`: It is the gold standard that implements the current Pod security best practices. Be warned though, it’s highly restricted, and many Pods will fail to meet its strict requirements.

At the time of writing, we cannot tweak or modify any of these policies. Also, we cannot import others or create our own.

***

## Pod Security Admission <a href="#page-title" id="page-title"></a>

Pod Security Admission (PSA) enforces our desired PSS policies. It works at the Namespace level and is implemented as a _validating admission controller_.

PSA offers three enforcement modes:

* `Warn`: Allows violating Pods to be created but issues a user-facing warning
* `Audit`: Allows violating Pods to be created but logs an audit event
* `Enforce`: Rejects Pods if they violate the policy

It’s a good practice to configure every Namespace with at least the `baseline` policy configured to either `warn` or `audit`. This allows us to start gathering data on which Pods are failing the policy and why. The next step is to enforce the `baseline` policy and start warning and auditing on the `restricted` policy.

Any Namespaces without a Pod Security configuration are a gap in our security configuration, and we should attach a policy as soon as possible, even if it’s only warning and auditing.

Applying the following label to a Namespace will apply the `baseline` policy to it. It will allow violating Pods to run but will generate a user-facing warning.

```cpp
// main.sh

pod-security.kubernetes.io/warn: baseline
```

The format of the label is `<prefix>/<mode>: <policy>` with the following options:

* Prefix is always `pod-security.kubernetes.io`.
* Mode is one of `warn`, `audit`, or `enforce`.
* Policy is always one of `privileged`, `baseline` or `restricted`.

PSAs operate as validating admission controllers, meaning they cannot modify Pods. They also cannot have any impact on running Pods.

***
