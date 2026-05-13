---
icon: kubernetes
---

# IAM & Security Monitoring

### Securing Kubernetes with RBAC <a href="#securing-kubernetes-with-rbac" id="securing-kubernetes-with-rbac"></a>

Controlling user access to Kubernetes is important in any production environment. Fortunately, Kubernetes has a robust RBAC subsystem that integrates with existing IAM providers such as Active Directory, other LDAP systems, and cloud-based IAM solutions.

Most organizations already have a centralized IAM provider integrated with company HR systems to simplify employee lifecycle management.

Fortunately, Kubernetes leverages existing IAM providers instead of implementing its own. This means new employees get an identity in the corporate IAM database, and assuming we make them members of the appropriate groups, they will automatically get permissions in Kubernetes. Likewise, when the employee leaves the organization, an HR process will automatically remove their identity from the IAM database, and their Kubernetes access will cease.

RBAC has been a stable Kubernetes feature since v1.8 and we should leverage its full capabilities.

### Managing Remote SSH access to cluster nodes <a href="#managing-remote-ssh-access-to-cluster-nodes" id="managing-remote-ssh-access-to-cluster-nodes"></a>

We’ll do almost all Kubernetes administration via REST calls to the API server. This means users should rarely need remote SSH access to Kubernetes cluster nodes. In fact, remote SSH access to cluster nodes should only be for the following types of activity:

* Node management activities that we cannot perform via the Kubernetes API
* Break the Glass activities, such as when the API server is down
* Deep troubleshooting

### Multi-factor authentication (MFA) <a href="#multi-factor-authentication-mfa" id="multi-factor-authentication-mfa"></a>

With great power comes great responsibility.

Accounts with root access to the API server and root access to cluster nodes are extremely powerful and are prime targets for attackers and disgruntled employees. As such, we should protect their use via multi-factor authentication (MFA). This is where a user has to input a username and password, followed by a second authentication stage. For example:

* Stage 1: Tests _knowledge_ of a username and password
* Stage 2: Tests _possession_ of something like a one-time password

We should also secure access to workstations and user profiles that have `kubectl` installed.

***

No system is 100% secure, and we should always plan for the eventuality that our systems will be breached. When breaches happen, it’s vital we can do at least two things:

1. Recognize that a breach has occurred.
2. Build a detailed timeline of events that cannot be repudiated.

Auditing is critical to both of these, and the ability to build a reliable timeline helps answer the following post-event questions:

* What happened?
* How did it happen?
* When did it happen?
* Who did it?

This information can be used in court in extreme circumstances. Good auditing and monitoring solutions also help identify vulnerabilities in our security systems. With these points in mind, we should ensure robust auditing and monitoring are high on our list of priorities, and we shouldn’t go live in production without them.

### Baseline best practices <a href="#baseline-best-practices" id="baseline-best-practices"></a>

Various tools and checks can help us ensure we provision our Kubernetes environment according to best practices and company policies. The _Center for Information Security (CIS)_ publishes an industry-standard benchmark for Kubernetes security, and [Aqua Security](https://aquasec.com/) has written an easy-to-use tool called [_kube-bench_](https://github.com/aquasecurity/kube-bench) to run the CIS tests against our cluster and generate reports. Unfortunately, kube-bench can’t inspect the control plane nodes of hosted Kubernetes services.

We should consider running kube-bench as part of the node provisioning process and pass or fail node provisioning based on the results. We can also use kube-bench reports as a baseline for use in the aftermath of incidents. This allows us to compare the kube-bench reports from before and after the incident and determine if and where any configuration changes occurred.

### Container and Pod lifecycle events <a href="#container-and-pod-lifecycle-events" id="container-and-pod-lifecycle-events"></a>

Pods and containers are ephemeral objects that come and go constantly. This means we’ll see many events announcing new ones and many events announcing terminated ones. With this in mind, consider configuring log retention to keep the logs from terminated Pods so they’re available for inspection even after termination. Our container runtime may also keep logs relating to container lifecycle events.

### Forensic checkpointing <a href="#forensic-checkpointing" id="forensic-checkpointing"></a>

**Forensics** is the science of collecting and examining available evidence to construct a trail of events, especially when we suspect malicious behavior. The ephemeral nature of containers has made this challenging in the past. However, recent technologies such as _Checkpoint/Restore in Userspace (CRIU)_ are making it easier to silently capture the state of running containers and restore them in a sandbox environment for deeper analysis. At the time of writing, CRIU is an alpha feature in Kubernetes, and the only runtime currently supporting it is CRI-O.

### Application logs <a href="#application-logs" id="application-logs"></a>

Application logs are also important when identifying potential security-related issues. However, not all applications send their logs to the same place. Some send them to their container’s _standard out (`stdout`)_ or _standard error (`stderr`)_ streams, where our logging tools can pick them up alongside container logs. However, some send logs to proprietary log files in bespoke locations. Be sure to research this for each application and configure things so we don’t miss logs.

### Actions performed by users <a href="#actions-performed-by-users" id="actions-performed-by-users"></a>

Most of our Kubernetes configuration and administration will be done via the API server, where all requests should be logged. However, it’s also possible for malicious actors to gain remote SSH access to control plane nodes and directly manipulate Kubernetes objects. This may include access to the cluster store and etcd nodes.

We’ve already said we should limit SSH access to cluster nodes and bolster security with multi-factor authentication (MFA). However, we should also log all SSH activity and ship it to a secure log aggregator. We should also consider mandating that two competent people be present for all SSH access to control plane nodes.

### Managing log data <a href="#managing-log-data" id="managing-log-data"></a>

A key advantage of containers is application density — we can run a lot more applications on our servers and in our data centers. This results in massive amounts of log data and audit data that are overwhelming without specialized tools to sort and make sense of it. Fortunately, advanced tools exist that not only store the data, but can use it for proactive analysis as well as post-event reactive analysis.

### Alerting for security-relevant events <a href="#alerting-for-security-relevant-events" id="alerting-for-security-relevant-events"></a>

As well as being useful for post-event analysis and repudiation, some events are significant enough to warrant immediate investigation. Examples include:

* _Privileged Pod creation by a human user:_ Privileged Pods can often gain root-level access on the node, and we will typically have policies in place to prevent their creation. On the rare occasions they are needed, they will usually be created by automated processes with service accounts.
* _Exec sessions by human users:_ Exec sessions grant _shell-like_ access to containers and are typically only used to troubleshoot issues. We should investigate exec sessions that aren’t for troubleshooting and consider deleting them to prevent tampering.
* _Attempts to access the cluster from the internet:_ It’s a common practice to prevent access to the control plane from the internet. As such, we should monitor for successful and unsuccessful attempts to connect to the control plane from the internet, and successful attempts will typically indicate a security misconfiguration we should fix.

### Migrating existing apps to Kubernetes <a href="#migrating-existing-apps-to-kubernetes" id="migrating-existing-apps-to-kubernetes"></a>

It can be useful to use a crawl, walk, and run strategy when migrating applications to Kubernetes:

1. Crawl: Threat modeling our existing apps will help us understand their current security posture. For example, which of our existing apps do and don’t communicate over TLS.
2. Walk: When moving to Kubernetes, we need to ensure that the security posture of these apps remains unchanged. For example, if an app doesn’t communicate over TLS, do _not_ change this as part of the migration.
3. Run: After the migration, we'll start improving the security of applications. We’ll start with simple non-critical apps and carefully work our way up to mission-critical apps. We may also want to methodically deploy deeper levels of security, such as initially configuring apps to communicate over one-way TLS and then eventually over two-way TLS.

The key point is not to change the security posture of an app as part of migrating it to Kubernetes. This is because performing a migration and making changes can make it easier to misdiagnose issues — was it the security change or the migration?

### Real-world example <a href="#real-world-example" id="real-world-example"></a>

An example of a container-related vulnerability that could’ve easily been prevented by implementing some of the best practices we’ve discussed occurred in February 2019. CVE-2019-5736 allowed a container process running as root to gain root access on the worker node and all containers running on the host.

As dangerous as the vulnerability was, the following things covered in this chapter would’ve prevented the issue:

* Image vulnerability scanning
* Not running processes as root
* Enabling SELinux

As the vulnerability has a CVE number, scanning tools would’ve found it and alerted on it. Even if scanning platforms missed it, policies that prevent root containers and standard SELinux policies would have prevented exploitation of the vulnerability.
