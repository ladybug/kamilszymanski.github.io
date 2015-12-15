---
layout: post
title: Optimizing Docker images for image size and build time
date: 2015-12-10T21:50:50+01:00
excerpt: "Image layers explained in context of image size and build time"
tags: [docker]
comments: true
share: true
---

Nowadays you can choose one of available storage drivers supported by Docker that suits best your environment and use-case, yet you should not even think about it until you understand image layers (not to mention images and containers).
You know those simple and not so sexy layers that make up an image and often fall into oblivion because new shiny tools get more traction than important basics.

In this blog post I'll discuss layers in context of image size and build time because I've seen a couple failed attempts to optimize for those.

Let's start with a brief recap of images and layers[^1]:

- Docker image is a tagged hierarchy of read-only layers plus some metadata
- each layer has its own UUID and each successive layer builds on top of the layer below it
- each Dockerfile instruction generates a new layer

Looks so simple as it needs no explanation, yet I have stumbled upon Dockerfiles looking like this:

{% highlight dockerfile %}
FROM centos:7.1.1503
RUN yum -y install java-1.8.0-openjdk-devel-1:1.8.0.65-2.b17.el7_1.x86_64
RUN yum clean all
{% endhighlight %}

What's wrong with this Dockerfile?
Well, the second `RUN` command has no desired impact on image size even if it looks as it should shrink the image.
Let's look once more at our brief recap of images and layers:

- Docker image is a tagged **hierarchy of read-only layers** plus some metadata
- each layer has its own UUID and **each successive layer builds on top of the layer below it**
- **each Dockerfile instruction generates a new layer**

Now it should be obvious where this Dockerfile fails at optimizing image size.
Anyway let's take a deeper look so that we understand how are those yum caches removed from image layers.

To not go too deep let's limit ourselves to AUFS storage driver.
The AUFS storage driver deletes a file from a layer by placing a whiteout file that effectively obscures the existence of the file in image’s lower, read-only layers.
Out of that you can deduce that image size is the sum of the sizes of the individual layers and that each additional instruction added to your Dockerfile will only ever increase the size of your image.

Fixing previous Dockerfile is as simple as combining both `RUN` commands:

{% highlight dockerfile %}
FROM centos:7.1.1503
RUN yum -y install java-1.8.0-openjdk-devel-1:1.8.0.65-2.b17.el7_1.x86_64 && \
    yum clean all
{% endhighlight %}

Let's build and inspect both of those images to prove it.
To build an image execute `docker build -t <TAG> .` inside a directory containing Dockerfile.
You should end up with 2 images with different sizes:

{% highlight console %}
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
combined-layers     latest              defa7a199555        4 seconds ago        407 MB
separate-layers     latest              b605eff36c7b        About a minute ago   471.8 MB
{% endhighlight %}

You should have no trouble in telling which one was build from which Dockerfile but let's look at what layers make up each of those images:

{% highlight console %}
$ docker history --no-trunc separate-layers
IMAGE                                                              CREATED             CREATED BY                                                                                         SIZE
b605eff36c7b418aa30e315dc0a1d809d08a1ebb8e574e934b11f5ad7cd490dc   2 minutes ago       /bin/sh -c yum clean all                                                                           2.277 MB
4c363330dc9057ce6285be496aa02212759ecfc75c4ad3a9a74e6e2f3dacb1dd   2 minutes ago       /bin/sh -c yum -y install java-1.8.0-openjdk-devel-1:1.8.0.65-2.b17.el7_1.x86_64                   257.5 MB
173339447b7ec3e8cb93edc61f3815ff754ec66cfadf48f1953ab3ead6a754c5   8 weeks ago         /bin/sh -c #(nop) CMD ["/bin/bash"]                                                                0 B
4e1d113aa16e0631795a4b31c150921e35bd1a3d4193b22c3909e29e6f7c718d   8 weeks ago         /bin/sh -c #(nop) ADD file:d68b6041059c394e0f95effd6517765405402b4302fe16cf864f658ba8b25a97 in /   212.1 MB
a2c33fe967de5a01f3bfc3861add604115be0d82bd5192d29fc3ba97beedb831   7 months ago        /bin/sh -c #(nop) MAINTAINER The CentOS Project <cloud-ops@centos.org> - ami_creator               0 B
{% endhighlight %}

{% highlight console %}
$ docker history --no-trunc combined-layers
IMAGE                                                              CREATED             CREATED BY                                                                                              SIZE
defa7a199555834ac5c906cf347eece7fa33eb8e90b30dfad5f9ab1380988ade   48 seconds ago      /bin/sh -c yum -y install java-1.8.0-openjdk-devel-1:1.8.0.65-2.b17.el7_1.x86_64 &&     yum clean all   195 MB
173339447b7ec3e8cb93edc61f3815ff754ec66cfadf48f1953ab3ead6a754c5   8 weeks ago         /bin/sh -c #(nop) CMD ["/bin/bash"]                                                                     0 B
4e1d113aa16e0631795a4b31c150921e35bd1a3d4193b22c3909e29e6f7c718d   8 weeks ago         /bin/sh -c #(nop) ADD file:d68b6041059c394e0f95effd6517765405402b4302fe16cf864f658ba8b25a97 in /        212.1 MB
a2c33fe967de5a01f3bfc3861add604115be0d82bd5192d29fc3ba97beedb831   7 months ago        /bin/sh -c #(nop) MAINTAINER The CentOS Project <cloud-ops@centos.org> - ami_creator                    0 B
{% endhighlight %}

This clearly shows that chaining commands allows you to clean up layers before they are committed, but that doesn't mean you should put everything into a single layer.
If you take another look at the output from previous commands you will spot that the 3 bottom layers of both images have the same UUIDs which means they are shared between those images (and that's why `docker images` command reports virtual size of images).
One might say that disk space is so cheap that you should not bother about it but there are other aspects of caching image layers you should care about.
One of the more important ones is image build time.
Simply speaking if you can reuse a layer you don't need to build it.
And if your images travel over the network (e.g. by being promoted from stage to stage in the pipeline) you can save a lot of time (and reduce network traffic) by making use of layers cache because what is actually transferred are image layers that are later on combined to make up the image itself.

By now it should be obvious that you should put instructions least likely to change at the top of your Dockerfile to reuse caching as much as possible and try to make changes only at the bottom of your Dockerfile.

Given all the above one might think that the most optimal solution would be to have a separate layer for all things that happen to change together with a clean up performed for each layer, but as usual it's not that simple.
First of all Docker limits the number of layers to 127 but you should never go anywhere near that number anyway.
A Dockerfile with tons of layers doesn't sound like maintainable one nor having only the necessary bits, instead it looks like it builds an image that does too much and should be splitted into separate images.
But more importantly layers don't come for free, depending on storage driver used there are some penalties to pay[^2].
For example in AUFS each layer can introduce latency to container write performance[^3] on the first write to each file existing in the image layers stack, especially if the file is big and exists below many image layers.

So in the end, as usual, it all boils down to knowing what you optimize for and consciously making necessary compromises.

[^1]: if you need an introduction or deep-dive into layers check [Docker documentation](https://docs.docker.com/)
[^2]: consult your storage driver documentation for more details on it's performance
[^3]: if you find yourself under heavy write workloads consider using data volumes (they bypass the storage driver)
