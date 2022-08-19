---
title: "DevAttackOps: Containerizing Red Team Infrastructure (Part 1)"
date: 2022-08-19T08:57:43-04:00
draft: true
description: "How to build a CI/CD pipeline for your container deployments"
keywords: ["red-team", "infrastructure", "docker", "containers", "devattackops"]
tags: ["red-team", "infrastructure", "docker", "containers", "devattackops"]
cover:
    image: "blog-covers/containers.jpeg"
    alt: "Containers"
    relative: true 
    responsiveImages: true
---

So here is my first actual blog post. I am going to talk about how Red Teams can use the same CI/CD pipelines used by developers to both simplify Red Team infrastructure and generally make your life easier. As our European friends say: Let's get stuck in!

# Why Containerize?

Here's a question that I kept asking myself: what the hell is all the rage about containers? And honestly, it's a great question! As a Red Teamer, here's two big reasons why you should care (yes there are likely more, but these are the reasons from my perspective):

## Repeatability

Red Teams are in the interesting category of teams since we are often deploying infrastructure into a variety of places. In doing so, we need to fight with various configurations that may be baked into each cloud provider's OS images. This isn't that big of a deal, but when you do run into one stupid configuration that is blocking your software from installing, it can be a huge time sink to try and determine what is going wrong. However, when you package all your code into a container, you know that no matter where you deploy that container to, it will always work the same way.

## Ease of Use

Have you ever had someone come up to you and complain that their install of <software-name> isn't working? Or have you ever onboard-ed a new team member and told them to just go play around with CobaltStrike just to get familiar with Command and Control frameworks? I am willing to bet you have answered yes to one of these. If you have, then you can benefit from containers. When you build a container, instead of directing that person to the documentation or stack overflow post of all the ways a CobaltStrike install can fail, you could just containerize the solution and give that person the `Dockerfile` or a registry to pull from and they are all set.

# Key Terms

So great, maybe I have convinced you about containerizing your red team tools! Before diving into "how" you do this, I need to define some key terms so I don't lose you in the process:

* Dockerfile: 
* Buildtime Arguments:
* Runtime Arguments: 


# Building your First Container

Wow, that was boring. Now for the fun stuff!

So let't work on building your first container. Just for example's sake, let's build a CobaltStrike container. Before we get into it, I will assume you have a basic understanding of how docker works. And also since my team uses debian-based operating systems, I will be making my example use `debian:stable-slim` as the base image.

## Automating the CobaltStrike Install

Before we can build the container for CobaltStrike, we need to automate the commands to install all dependencies, download CobaltStrike package, extract the package, and then run the update script (which will also license the product). Let's break each step down...

### Install all dependencies

Since CobaltStrike is Java-based, we need to make sure that we have java installed into the container. Additionally, there are other dependencies that we will want to install into the container to make sure everything works as intended. Since debian uses `apt` we can install the latest java package and other packages from `apt` using the following command:

```bash
apt-get install --no-install-recommends -y ca-certificates expect git openjdk-11-jdk wget curl iproute2 gnupg
```

### Download the CobaltStrike Package

Great, now that we have all the required dependencies, we can download the package from their website. Should be a simple `wget` command right? Well not exactly...

To download CobaltStrike, you first need to go to https://download.cobaltstrike.com/download and then enter your license key. Doing so will then redirect you to another page where you select your operating system _and then_ you can download CobaltStrike. This is a little challenging considering when you enter your license key, you then get a "token" that is part of the download URL. In order to get that token, you can make the initial `GET` request with your license key as a query parameter and use some bash magic to extract that token.

```bash
curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3
```

From that command, we get the token that we need to include in part of the download URL. However, just getting the token does us no good. We need to save off that token and using it in a subsequent request. We can do this by exporting the results of the curl command above to an environment variable by wrapping that command in `export TOKEN=$(<curl-command>)`.

```bash
export TOKEN=$(curl -s https://download.cobaltstrike.com/download -d "dlkey=${COBALTSTRIKE_LICENSE}" | grep 'href="/downloads/' | cut -d '/' -f3)
```

Now that we have the token, all we need to do it make a `wget` to get the and use bash expansion to expand that token in the `wget` request.

```bash
wget https://download.cobaltstrike.com/downloads/${TOKEN}/latest46/cobaltstrike-dist.tgz
```

### Extracting and Updating CobaltStrike

Now that we have the package, we can use `tar` to extract the package.

```bash
tar zxf cobaltstrike-dist.tgz 
```

Before we can run the `teamserver`, we need to run the `update` script to license the package we just downloaded. Since the `update` script will expect we provide the license key via standard input, we can just use an `echo` command to give it the key.

```bash
echo ${COBALTSTRIKE_LICENSE} | ./update
```

### Converting to a Dockerfile

Great, now we have a working version of CobaltStrike installed! Now we want to take all of this and put it into a Dockerfile. Since a Dockerfile can just take and run bash commands, we can take everything we just learned and put it right into a Dockerfile. The only difference here is that we will want to pass the CobaltStrike license as a `build-arg` to the container so that when we run the container, we have a fully licensed version of CobaltStrike. The way we do this is by defining a `ARG` in the Dockerfile and then pass that `ARG` in at build time. The Dockerfile then looks like below.

```dockerfile
FROM debian:stable-slim

# Required Arguments
ARG COBALTSTRIKE_LICENSE

# Install all dependencies
RUN apt-get update && \
	apt-get install --no-install-recommends -y ca-certificates expect git openjdk-11-jdk wget curl iproute2 gnupg && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
	update-java-alternatives -s java-1.11.0-openjdk-amd64

# Install and Update Cobalt Strike
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

And the build command looks like this.

```bash
docker build -t cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE .
```

# Building a CI/CD Pipeline

## Attempt #1 

```yaml
build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
  variables:
    CI_REGISTRY_PATH: $CI_REGISTRY/5tag3/terry-container-registry
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest
    - docker build -t $CI_REGISTRY_PATH/deimos:latest ./deimos
    - docker push $CI_REGISTRY_PATH/deimos:latest
    - docker build -t $CI_REGISTRY_PATH/gophish:latest ./gophish
    - docker push $CI_REGISTRY_PATH/gophish:latest
    - docker build -t $CI_REGISTRY_PATH/sliver:latest ./sliver
    - docker push $CI_REGISTRY_PATH/sliver:latest
```

## Attempt #2

```yaml
image: docker:19.03.12

stages:
  - build

# Global Job Config for all container builds (include build arg variables here)
.job_configuration_template: &job_configuration
  stage: build
  services:
    - docker:19.03.12-dind
  variables:
    CI_REGISTRY_PATH: $CI_REGISTRY/5tag3/terry-container-registry
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script: 
   - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Actual Builds
build_cobaltstrike:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest

build_deimos:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/deimos:latest ./deimos
    - docker push $CI_REGISTRY_PATH/deimos:latest

build_sliver:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/sliver:latest ./sliver
    - docker push $CI_REGISTRY_PATH/sliver:latest

build_gophish:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/gophish:latest ./gophish
    - docker push $CI_REGISTRY_PATH/gophish:latest

build_evilginx2:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/evilginx2:latest ./evilginx2
    - docker push $CI_REGISTRY_PATH/evilginx2:latest

```

# Extra Credit

## Docker Compose
