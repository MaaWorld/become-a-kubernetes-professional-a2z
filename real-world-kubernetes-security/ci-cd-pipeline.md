---
icon: kubernetes
---

# CI/CD Pipeline

The previous chapter showed us how to threat-model Kubernetes using the STRIDE model. In this chapter, we’ll learn about security-related challenges we’re likely to encounter when implementing Kubernetes in the real world.

The chapter's goal is to show us things from the high-level view of a security architect. It does not provide _cookbook-_&#x73;tyle solutions.

### Security in the software delivery pipeline <a href="#security-in-the-software-delivery-pipeline" id="security-in-the-software-delivery-pipeline"></a>

Containers revolutionized the way we build, ship, and run applications. Unfortunately, this has also made it easier than ever to run dangerous code.

Let’s look at some ways to secure the supply chain that gets application code from a developer’s laptop to production servers.

#### Image repositories <a href="#image-repositories" id="image-repositories"></a>

We store images in public and private registries that we divide into _repositories_.

Public registries are on the internet and are the easiest way to push and pull images. However, we should be very careful when using them:

1. We need to adequately protect the images we store on public registries.
2. We should not trust the images we pull from public registries.

Some public registries have the concept of official images and community images. Generally, official images are safer than community images, but we should _always_ do our due diligence.

Official images are usually provided by product vendors and undergo vigorous vetting processes to ensure quality. We should expect them to implement good practices, be regularly scanned for vulnerabilities, and contain up-to-date patches and fixes. Some of them may even be supported by the product vendor or the company hosting the registry.

Community images do not undergo rigorous vetting, and we should practice extreme caution when using them.

With these points in mind, we should implement a standardized way for developers to obtain and consume images. We should also make the process as frictionless as possible so that developers don’t feel the need to bypass the process.

Let’s discuss a few things that might help.

#### Use approved base images <a href="#use-approved-base-images" id="use-approved-base-images"></a>

Most images start with a _base layer_ and then add other layers to form a useful image.

The following figure shows an oversimplified example of an image with three layers. The base layer has the core OS and filesystem components, the middle layer has the libraries and dependencies, and the top layer has our app. The combination of the three is the image and contains everything needed to run the application.

<figure><img src="../.gitbook/assets/image.png" alt="" width="311"><figcaption><p>Image layering</p></figcaption></figure>

It’s usually a good practice to maintain a small number of _approved base images_. These are usually derived from official images and hardened according to our corporate policies and requirements. For example, we might create a limited number of _approved base images_ based on the official Alpine Linux image we’ve tweaked to meet our requirements (patches, drivers, audit settings, and more).

The following figure shows three applications built on top of two approved base images. The app on the left builds on top of our approved Alpine Linux base image, whereas the other two apps are web apps that build on top of our approved Alpin+NGINX base image.

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="560"><figcaption><p>Using approved base image</p></figcaption></figure>

While we need to invest up-front effort to create our approved base images, they bring the following benefits:

* Standard set of drivers
* Known patches
* Standardized audit settings
* Reduced software sprawl (less unofficial base images)
* Simplified testing (testing against a small set of known bases)
* Simplified updates (fewer base images to patch)
* Simplified troubleshooting (a well-understood and limited set of base images)

Having an approved set of base images also allows developers to focus on applications without caring about OS-related stuff. It may also allow us to reduce the number of support contracts and suppliers we have to deal with.

#### Manage the need for non-standard base images <a href="#manage-the-need-for-non-standard-base-images" id="manage-the-need-for-non-standard-base-images"></a>

As good as having a small number of approved base images is, we may still have legitimate requirements for bespoke configurations. In these situations, we’ll need good processes to:

* Identify why an existing approved base image cannot be used.
* Determine whether an existing approved base image can be updated to meet requirements (including if it’s worth the effort).
* Determine the support implications of bringing an entirely new image into the environment.

In most cases, we’ll want to update an existing base image such as adding a device driver for GPU computing rather than introducing an entirely new image.

#### Control access to images <a href="#control-access-to-images" id="control-access-to-images"></a>

There are several ways to protect our organization’s images.

A secure and practical option is to host our private registries inside our firewalls. This allows us to control how registries are deployed, replicated, and patched. We can also create repositories and policies to fit our organizational needs and integrate them with existing identity management providers such as Active Directory.

If we can’t manage our private registries, we can host our images in _private repositories_ on public registries. However, not all public registries are equal, and we’ll need to take great care in choosing the right one and configuring it correctly.

Whichever solution we choose, we should only host images that are approved for use within our organization. These will typically be from a _trusted_ source and vetted by our information security team. We should place access controls on repositories so that only approved users can push and pull them.

Away from the registry itself, we should also:

* Restrict which cluster nodes have internet access, keeping in mind that our image registry may be on the internet.
* Configure access controls that only allow authorized users and nodes to push to repositories.

If we’re using a public registry, we’ll probably need to grant our cluster nodes access to the internet so they can pull images. In scenarios like this, it’s a good practice to limit internet access to the addresses and ports our registries use. We should also implement strict RBAC rules on the registry to control who can push and pull images from which repositories. For example, we might restrict developers so they can only push and pull against _dev_ and _test_ repositories, whereas we may allow our operations teams to push and pull against production repositories.

Finally, we may only want a subset of nodes (_build nodes_) to be able to _push_ images. We may even want to lock things down so that only our automated build systems can push to specific repositories.
