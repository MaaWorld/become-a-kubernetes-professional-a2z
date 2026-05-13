---
icon: kubernetes
---

# Moving Images From Non-Production to Production

Many organizations have separate environments for development, testing, and production. Usually, development environments have fewer rules and are places where developers can experiment. This can involve non-standard images our developers eventually want to use in production. The following sections outline some measures we can take to ensure that only safe images get approved for production.

### Vulnerability scanning <a href="#vulnerability-scanning" id="vulnerability-scanning"></a>

Vulnerability scanning should be at the top of the list for vetting images before allowing them into production. These services scan our images at a binary level and check their contents against databases of known security vulnerabilities (CVEs).

We should integrate vulnerability scanning into our CI/CD pipelines and implement policies that automatically fail builds and quarantine images if they contain particular categories of vulnerabilities. For example, we might implement a build phase that scans images and automatically fails anything using an image with known _critical_ vulnerabilities.

However, some scanning solutions are better than others and will allow us to create highly customizable policies. For example, a Python method that performs TLS verification might be vulnerable to denial-of-service attacks when the Common Name contains a lot of wildcards. However, if we never use Python this way, we might not consider the vulnerability relevant and want to mark it as a false positive. Not all scanning solutions allow us to do this.

### Configuration as code <a href="#configuration-as-code" id="configuration-as-code"></a>

Scanning app code for vulnerabilities is widely accepted as good production hygiene. However, scanning our Dockerfiles, Kubernetes YAML files, Helm charts, and other configuration files is less widely adopted.

A well-publicized example of not reviewing configuration files was when an IBM data science experiment embedded private TLS keys in its container images. This meant attackers could pull the image and gain root access to the nodes hosting the containers. The whole thing would’ve been easily avoided if they’d performed a security review against their Dockerfiles. Continuous advancements are being made in automating checks like these with tools that implement policy as code rules.

### Sign container images <a href="#sign-container-images" id="sign-container-images"></a>

Trust is a big deal today, and cryptographically signing content at every stage in the software delivery pipeline is becoming the norm. Fortunately, Kubernetes and most container runtimes support cryptographically signing and verifying images.

In this model, developers cryptographically sign their images, and consumers cryptographically verify them when they pull them and run them. This gives the consumer confidence they’re working with the correct image and that it hasn’t been tampered with.

The following figure shows the high-level process for signing and verifying images.

<figure><img src="../.gitbook/assets/image (2).png" alt="" width="545"><figcaption><p>Process for signing and verifying images</p></figcaption></figure>

Image signing and verification is usually implemented by the container runtime. We should look at tools that allow us to define and enforce enterprise-wide signing policies so that it’s not left up to individual users.

### Image promotion workflow <a href="#image-promotion-workflow" id="image-promotion-workflow"></a>

With everything we’ve covered so far, our build pipelines should include as many of the following as possible:

1. Policies forcing the use of signed images
2. Network rules restricting which nodes can push and pull images
3. RBAC rules protecting image repositories
4. Use of approved base images
5. Image scanning for known vulnerabilities
6. Promotion and quarantining of images based on scan results
7. Review and scan infrastructure-as-code configuration files

There are more things we can do and the list isn’t supposed to represent an exact workflow.

***

