---
title: "DevAttackOps: Containerizing Red Team Infrastructure (Part 1)"
date: 2022-08-19T08:57:43-04:00
draft: false
description: "Containerizing Attack and C2 infrastructure"
keywords: ["red-team", "infrastructure", "docker", "containers", "devattackops"]
tags: ["red-team", "infrastructure", "docker", "containers", "devattackops"]
cover:
    image: "blog-covers/containers.jpeg"
    alt: "Containers"
    relative: true 
    responsiveImages: true
---

**TLDR;** Building containers for Red Team applications is easy. I have included a Cobalt Strike Dockerfile for you in the [Artifacts](#artifacts) section.

So here is my first actual blog post. In this series, I am going to talk about how Red Teams can use the same tools used by developers to both simplify Red Team infrastructure and generally make your life easier. In Part 1, I will be talking about why Red Teams _should_ use containers in their workflows. As our European friends say, "Let's get stuck in!"

# Why Containerize?

Here is a question that I kept asking myself: what the hell is all the rage about containers? Honestly, it is a great question! As a Red Teamer, here are two big reasons why you should care (yes there are likely more, but here are the reasons from my perspective):

## Repeatability

Red Teams are in the interesting category of teams since we are often deploying infrastructure into a variety of places. In doing so, we need to fight with various configurations that may be baked into each cloud provider's OS images. This is not that big of a deal, but when you do run into a stupid configuration that is blocking your software from installing, it can be a huge time sink to try and determine what is going wrong. However, when you package all your code into a container, you know that no matter where you deploy that container to, it will always work the same way.

## Ease of Use

Have you ever had someone come up to you and complain that their install of _insert-software-name_ is not working? Or have you ever onboarded a new team member and told them to just go play around with Cobalt Strike just to get familiar with Command and Control (C2) frameworks? I am willing to bet you have answered yes to one of these questions. If you have, then you can benefit from containers. When you build a container, instead of directing that person to the documentation or a Stack Overflow post of all the ways a Cobalt Strike install can fail, you could just containerize the solution and give that person the `Dockerfile` or a registry to pull from and they are all set.

# Key Terms

Great, maybe I have convinced you about containerizing your Red Team tools! Before diving into "how" you do this, I need to define some key terms so I do not lose you in the process:

* Dockerfile: a text document that contains all the commands a user could call on the command line to assemble an image (From Docker official documentation [here](https://docs.docker.com/engine/reference/builder/))
* Buildtime Arguments: dynamic arguments passed into Docker at the container "build time" to allow for more control over a container image when building
* Runtime Arguments: arguments that are passed into the `docker run` command and then passed as arguments to the container execution point

# Building your First Container

Wow, that was boring. Now for the fun stuff!

Let's work on building your first container. For example's sake, let's build a Cobalt Strike container. Before we begin, I will assume you have a very basic understanding of how Docker works. And also, since my team uses Debian-based operating systems, I will be making my example with `debian:stable-slim` as the base image.

## Automating the Cobalt Strike Install

Before we can build a container for Cobalt Strike, we need to automate the commands to install all the dependencies, download the Cobalt Strike package, extract the package, and run the update script which will also license the product. Let's break each step down...

### Install All Dependencies

Since Cobalt Strike is Java-based, we need to make sure that we have Java installed in the container. Additionally, there are other dependencies that we will want to install in the container to make sure everything works as intended. Since Debian uses `apt`, we can install the latest Java package and other packages using the following command:

```bash
apt-get install --no-install-recommends -y ca-certificates curl expect git gnupg iproute2 openjdk-11-jdk wget
```

### Download the Cobalt Strike Package

Great, now that we have all the required dependencies, we can download the package from their website. This should be a simple `wget` request, right? Well not exactly...

To download Cobalt Strike, go to https://download.cobaltstrike.com/download and enter your license key. You will be redirected to another page where you select your operating system and then download the package. This is a little challenging considering when you enter your license key, you get a token that is part of the download URL. In order to capture that token, you can make the initial `GET` request with your license key as a query parameter and use some Bash magic to extract that token. In this example, I have set the `COBALTSTRIKE_LICENSE` environment variable which allows me to dynamically reference it in the initial `curl` request.

```bash
export COBALTSTRIKE_LICENSE="<cobaltstrike_license"
```

```bash
curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3
```

With this `curl` command, we get the token that we need to include in the download URL. However, just getting the token does us no good. We need to save the token and use it in a subsequent request. We can do this by exporting the results of the `curl` command above to an environment variable by wrapping that command in `export TOKEN=$(<curl-command>)`.

```bash
export TOKEN=$(curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3)
```

Now that we have the token, all we need to do is make a `wget` to get the `cobaltstrike-dist.tgz` file and use Bash expansion to expand that token in the `wget` request.

```bash
wget https://download.cobaltstrike.com/downloads/${TOKEN}/latest46/cobaltstrike-dist.tgz
```

### Extracting and Updating Cobalt Strike

Now that we have the package, we can use `tar` to extract the package.

```bash
tar zxf cobaltstrike-dist.tgz 
```

However, before we can run the `teamserver`, we need to run the `update` script to license the package we just downloaded. Since the `update` script will expect us to provide the license key via standard input, we can use an `echo` command to give it the key.

```bash
echo ${COBALTSTRIKE_LICENSE} | ./update
```

## Creating the Dockerfile

Great, now we have a working version of Cobalt Strike installed! Since a Dockerfile can take and run Bash commands to setup a container image, we can take everything we just learned and put it into a Dockerfile. To start building our first container, run the following commands:

```bash
mkdir cobaltstrike
cd cobaltstrike
touch Dockerfile
```

### Using Buildtime Arguments

Using our newly created directory and file, we now can start building out the Dockerfile. In the example above, I exported the `COBALTSTRIKE_LICENSE` as an environment variable. But how the hell can I mimic that functionality with Docker so I do not need to hardcode my license key into the Dockerfile? This is where buildtime arguments come in. Using a buildtime argument, we will pass the Cobalt Strike license using the `build-arg` flag so that when we build the container, we have a fully licensed version of Cobalt Strike. The way we do this is by defining a `ARG` in the Dockerfile and then pass that `ARG` at build time. 

```dockerfile
FROM debian:stable-slim

# Required Arguments
ARG COBALTSTRIKE_LICENSE
```

Now we can use that same environment variable called `COBALTSTRIKE_LICENSE` we set earlier to pass in the license key.

```bash
docker build -t cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE .
```

### Converting the Manual Commands

Now we need to convert all the manual commands we ran earlier into Dockerfile commands. Thankfully, we can use the `RUN` command in our Dockerfile to just copy and paste all the commands we ran earlier.

```dockerfile
# Install all dependencies
RUN apt-get update && \
	apt-get install --no-install-recommends -y ca-certificates curl expect git gnupg iproute2 openjdk-11-jdk wget && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
	update-java-alternatives -s java-1.11.0-openjdk-amd64

# Install and update Cobalt Strike
RUN echo "COBALTSTRIKE_LICENSE: ${COBALTSTRIKE_LICENSE}" && \
  export TOKEN=$(curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3) && \
	cd /opt && \
	wget https://download.cobaltstrike.com/downloads/${TOKEN}/latest46/cobaltstrike-dist.tgz  && \
	tar zxf cobaltstrike-dist.tgz && \
	rm /etc/ssl/certs/java/cacerts && \
	update-ca-certificates -f && \
	cd /opt/cobaltstrike && \
	echo ${COBALTSTRIKE_LICENSE} | ./update && \
	mkdir /opt/cobaltstrike/mount

```

**Pro Tip:** If you think you have errors in your Dockerfile, run the build command we defined earlier. This will tell you exactly where you have issues.

### Exposing Ports

One of the most difficult concepts in Docker is networking. Because a container is an entire operating system _inside an operating system_, the container has no knowledge of the base operating system's networking configuration. Nor should it. Because of this, we need to tell the container _only what it needs to know_. For example, what ports should be exposed to the host operating system.

For Cobalt Strike, the most commonly used ports are:
* 50050/TCP: teamserver connection
* 443/TCP: HTTPS C2
* 80/TCP: HTTP C2
* 53/UDP: DNS C2

Because a container is a self-contained operating system (pardon my pun), we must use the `EXPOSE` function in our Dockerfile to tell the container to _explicitly allow_ specific ports to be accessed. Doing this in a Dockerfile is very easy.

```dockerfile
EXPOSE 50050 443 80 53/udp
```

By using the `EXPOSE` definition above, we are explicitly telling the container to allow networking connections to happen on ports 50050/TCP, 443/TCP, 80/TCP, and 53/UDP. 

### Setting the Entrypoint

The purpose of our container is to be a standalone implementation of a Cobalt Strike teamserver. Up to this point, we have Cobalt Strike being installed into the container and we have told the container what ports should be accessible, but we still have yet to run the `teamserver` command to stand up the actual teamserver. In Docker, we can use the `ENTRYPOINT` function to tell the container _what program to start when the container starts_. This means that when I run this container, I want to have it automatically run the `teamserver` command.

```dockerfile
ENTRYPOINT ["./teamserver"]
```

In defining this `ENTRYPOINT`, the container will now boot and immediately execute that command. However, keep in mind that if the `teamserver` command exits or errors out, the container will stop. Whenever the `ENTRYPOINT` command exits, the entire container will stop. This is typically a _gotcha_ with most developers who start tinkering around with containers. 

Also, for those of you who have played with containers before, you probably know that there is also the `CMD` command that also exists. The key difference is that when you use `CMD` and pass in runtime arguments, you need to explicitly tell the container what base binary you want to run (essentially overwriting that `CMD` in the Dockerfile). If you want to learn more, here is a great [blog post](https://awstip.com/docker-run-vs-cmd-vs-entrypoint-78ca2e5472bd) that covers the key differences.

### Putting it all Together

Wow! You are still reading this? Kudos to you for being dedicated to wanting to learn! Now that we have everything, we need to actually build the container. Let's put it all together and build this damn thing! If we put all the commands above together, we get a Dockerfile that looks like this:

```dockerfile
FROM debian:stable-slim

# Required Arguments
ARG COBALTSTRIKE_LICENSE

# Install all dependencies
RUN apt-get update && \
	apt-get install --no-install-recommends -y ca-certificates curl expect git gnupg iproute2 openjdk-11-jdk wget && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
	update-java-alternatives -s java-1.11.0-openjdk-amd64

# Install and update Cobalt Strike
RUN echo "COBALTSTRIKE_LICENSE: ${COBALTSTRIKE_LICENSE}" && \
  export TOKEN=$(curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3) && \
	cd /opt && \
	wget https://download.cobaltstrike.com/downloads/${TOKEN}/latest46/cobaltstrike-dist.tgz  && \
	tar zxf cobaltstrike-dist.tgz && \
	rm /etc/ssl/certs/java/cacerts && \
	update-ca-certificates -f && \
	cd /opt/cobaltstrike && \
	echo ${COBALTSTRIKE_LICENSE} | ./update && \
	mkdir /opt/cobaltstrike/mount

# Expose the ports and run it
WORKDIR /opt/cobaltstrike
EXPOSE 50050 443 80 53/udp
ENTRYPOINT ["./teamserver"]
```

And with that Dockerfile in our `cobaltstrike` directory, we can now build the container image.

```bash
docker build -t cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE .
```

## Using the Container (without a Malleable C2 Profile)

We have a new fancy schmancy Cobalt Strike container. Now we want to _use_ it. This is where we need to use the `docker run` command. 

```bash
docker run -it cobaltstrike 192.168.1.1 password
```

With the command above, we are essentially running the following command `./teamserver 192.168.1.1 password`, but it is all in a container. This means that it is all running in its own operating system. The `-it` flag starts the container in interactive mode. This allows us to see the standard output of the container. We can use `ctrl + c` on that process to kill the container.

Remember what I said earlier, _the hardest part to Docker is networking_. Although we are running a container, other computers have no idea that a container is running and listening on the ports we had `EXPOSE`'d. To fix this, we need to define port proxies in our docker run command. This will tell the host operating system what ports it should listen to and what ports it should map to the container ports. This can be accomplished using the `-p <host_port>:<container_port>` syntax.

```bash
docker run -it -p "50050:50050/tcp" -p "443:443/tcp" -p "80:80/tcp" -p "53:53/udp" cobaltstrike 192.168.1.1 password
```

From here, we now have a full Cobalt Strike container listening on all interfaces of your host operating system! 

## Using the Container (with a Malleable C2 Profile)

All of that was super cool, right? But wait a minute... what the hell do I do if I want to use a Malleable C2 profile with the container? This is where bind mounts come into play. We need to tell the container to mount to the host operating system to allow the container to access files on the host. Keep in mind, doing so _may have security implications_. Make sure you understand those implications before doing this with all of your containers.

Inside the run command we can create a bind mount using the `--mount` parameter. Let's say you already have a Malleable C2 profile created called `c2.profile` and it's located in the `cobaltstrike` directory you created earlier. In order to use that profile in the container, you need to create a bind mound to the `cobaltstrike` directory in the container and then reference that profile (that now exists in the container's target directory) in the entrypoint.

```bash
docker run -it --mount type=bind,source="$(pwd)",target=/opt/cobaltstrike/mount -p "50050:50050/tcp" -p "443:443/tcp" -p "80:80/tcp" -p "53:53/udp" cobaltstrike 192.168.1.1 password /opt/cobaltstrike/mount/c2.profile
```

With that, you have everything you now need to build and run your very own Cobalt Strike container.

# Demonstration

With everything we have learned, we can now build the container image.

{{< figure align=center src="../../blog-images/container-build.png" >}}

After building the image, we can run the container with the bind mount to use our custom C2 profile.

{{< figure align=center src="../../blog-images/container-run.png" >}}

We see the container has booted properly. Now we can use the host operating system's IP address to connect to the container.

{{< figure width="450" height="auto" align=center src="../../blog-images/connecting-to-cobaltstrike.png" >}}

After we hit connect, we become connected to our containerized version of Cobalt Strike!

{{< figure align=center src="../../blog-images/connected-to-cobaltstrike.png" >}}

# What's Next?

Next week, I will release a blog post on how we can leverage a container registry to take this container image and allow other operators use the prebuilt image. We will use one line of code to pull that container down and run it! Stay tuned!

# Artifacts

The Dockerfile:

```dockerfile
FROM debian:stable-slim

# Required Arguments
ARG COBALTSTRIKE_LICENSE

# Install all dependencies
RUN apt-get update && \
	apt-get install --no-install-recommends -y ca-certificates curl expect git gnupg iproute2 openjdk-11-jdk wget && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
	update-java-alternatives -s java-1.11.0-openjdk-amd64

# Install and update Cobalt Strike
RUN echo "COBALTSTRIKE_LICENSE: ${COBALTSTRIKE_LICENSE}" && \
  export TOKEN=$(curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3) && \
	cd /opt && \
	wget https://download.cobaltstrike.com/downloads/${TOKEN}/latest46/cobaltstrike-dist.tgz  && \
	tar zxf cobaltstrike-dist.tgz && \
	rm /etc/ssl/certs/java/cacerts && \
	update-ca-certificates -f && \
	cd /opt/cobaltstrike && \
	echo ${COBALTSTRIKE_LICENSE} | ./update && \
	mkdir /opt/cobaltstrike/mount

# Expose the ports and run it
WORKDIR /opt/cobaltstrike
EXPOSE 50050 443 80 53/udp
ENTRYPOINT ["./teamserver"]
```

The build command:

```bash
docker build -t cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE .
```

The run command (without Malleable C2):

```bash
docker run -it -p "50050:50050/tcp" -p "443:443/tcp" -p "80:80/tcp" -p "53:53/udp" cobaltstrike 192.168.1.1 password
```

The run command (with Malleable C2):

```bash
docker run -it --mount type=bind,source="$(pwd)",target=/opt/cobaltstrike/mount -p "50050:50050/tcp" -p "443:443/tcp" -p "80:80/tcp" -p "53:53/udp" cobaltstrike 192.168.1.1 password /opt/cobaltstrike/mount/c2.profile
```
