---
title: "DevAttackOps: Deploying Containers with Docker Compose (Part 3)"
date: 2022-10-16T16:14:53-04:00
draft: false
description: "Deploy dynamic containers using Docker Compose and Ansible"
keywords: ["red-team", "infrastructure", "docker", "containers", "devattackops", "docker-compose"]
tags: ["red-team", "infrastructure", "docker", "containers", "devattackops", "docker-compose"]
cover:
    image: "blog-covers/docker-compose-deploy-ansible.jpeg"
    alt: "Gears and chains"
    relative: true 
    responsiveImages: true
---

**TLDR;** Now that we have a container image registry, we can leverage the power of Docker Compose to deploy "containers as a configuration" to hosts. Check out the [Artifacts](#artifacts) section to see some examples.

**Disclaimer:** If you have not read DevAttackOps Part 1 or Part 2, you should start there. We will be building off the concepts of those posts in this post.

Welcome to part 3 of the DevAttackOps series where I talk about all things regarding Red Team infrastructure automation! If you have stuck around this long, thank you! Let's start with some basics and the problem we are trying to address.

# Multi-Container Problems

We now have an understanding of the value-add we get from containers for Red Team infrastructure. However, let's revisit the `docker run` command we use to start a container. To start Cobalt Strike, we would use something like the following:

```bash
docker run -it --mount type=bind,source="$(pwd)",target=/opt/cobaltstrike/mount -p "50050:50050/tcp" -p "443:443/tcp" -p "80:80/tcp" -p "53:53/udp" cobaltstrike 192.168.1.1 password /opt/cobaltstrike/mount/c2.profile
```

When we investigate this `docker run` command, we can see a lot of complex syntax. Let's remember, I am an idiot and remembering that command syntax is unrealistic for my tiny brain. I often have to go back to [DevAttackOps Part 1](../containerizing-red-team-infra) to remember what command I need to use to start Cobalt Strike with all the proper port mappings and bind mounts for Cobalt Strike to operate as I need. 

Oh and what if I _also_ wanted to use Sliver on that exact same server? Well now I need to go look up the command I use for Sliver too. Luckily, I am super organized with all my code and it takes me no time at all to find that command (please read with heavy sarcasm). To start Sliver, we would use something like the following:

```bash
docker run -it --name sliver -p "3333:3333/tcp" -p "3443:443/tcp" -p "380:80/tcp" -p "353:53/udp" sliver -p 3333
```

Also, take note of the port mappings I use for Sliver as I am using 2 different command and control frameworks on the same host. Because of this, I now need to change some of the port mappings so that I don't have port collisions on the host operating system (since both Cobalt Strike and Sliver both have some of the same protocols implemented for C2).

This got me thinking... Wouldn't it be nice if I could instead of having to remember a complex command and syntax like the two above, and use a single configuration file to define all those options? Yes it would be! And lucky for me, others before me thought of the exact same thing and came up with a solution called "Docker Compose."

# What is Docker Compose?

On the official Docker website, they define Docker Compose as the following:

> a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration. (https://docs.docker.com/compose/)

This means now I can take the commands above, convert them into a `docker-compose.yml` file (which is the standard name for a Docker Compose file), use some automation to deploy that `docker-compose.yml` to a host, and then use the following command to start all the containers (no matter their configuration):

```bash
docker-compose up -d
```

This is so much easier to remember than the two `docker run` commands I have listed above. But now, we need to figure out how to convert the two commands above into Docker Compose syntax.

**Note:** You will need to install Docker Compose as it will often not come pre-installed with Docker. Follow the directions to install it by following the [official Docker instructions here](https://docs.docker.com/compose/install/). Also, some versions of Docker compose use the `docker compose` syntax while others use the `docker-compose` syntax. They both operate in the same way, the only difference is the hyphen :) 

## Compose Syntax

If we look at the Docker Compose [getting started](https://docs.docker.com/compose/gettingstarted/#step-3-define-services-in-a-compose-file) page, there are a few examples that we can glean some information from.

```yml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine"
```

In this example, we are telling Docker to bring up two different containers, one called `web` and another called `redis`. In the `web` container, we are building the container image on the fly using a `Dockerfile` that is located in the current directory and then mapping the `web` container TCP port 5000 to the host operating system's TCP port 8000. With the `redis` container, we are just pulling an image back from the official Docker registry (this is assumed as there is no full path to a container registry / image).

This seems simple enough. It's just a YAML file that defines two containers where one gets built on the fly and the other gets pulled from an external registry. Let's take this example and adapt it to work for just Sliver first.

# Converting Docker Run to Docker Compose

In [DevAttackOps Part 2](../containers-ci_cd) we built out a CI/CD pipeline to automatically build out our container images and publish them to a private registry. Today, we are going to stick with pulling those images from our private registry instead of building the images on the fly like the `web` container does in the example above. So first, let's start with Sliver. 

## Sliver Conversion

To start converting Sliver, we first need to create a `docker-compose.yml` file. 

```bash
touch docker-compose.yml
```

Then, we need to create a key called `sliver` under the `services` key inside of our newly created `docker-compose.yml` file and tell `sliver` to pull the container `image` from our private registry we built in [Part 2](../containers-ci_cd). And just for added ease of use, we will add a `container_name` to `sliver` called `sliver` just to make it easier to label / reference containers once they are running.

```yml
version: "3.9"
services:
  sliver:
    container_name: sliver
    image: registry.gitlab.com/ezragit/container-ci_cd/sliver:latest
```

All we are doing is telling Docker Compose to pull back the image from our private container registry and once the container is started, name it `sliver`. If we now save our new `docker-compose.yml` on a host with Docker and Docker Compose already installed and then run `docker-compose up -d` in the same directory as the `docker-compose.yml` file, Docker will pull back that image and run it in detached mode (hence the `-d`). 

**Remember:** You may need to authenticate to your private container registry before running `docker-compose up -d` (more information exists in [Part 2](../containers-ci_cd/#understanding-the-container-registry))

Running `docker-compose up -d` with the `sliver` container as defined above works just fine, but Sliver requires that you provide runtime arguments to the container to tell it to run in daemon mode and what port to expose for multiplayer mode. Since Docker Compose exposes the runtime arguments as a configuration option, we can easily add our custom runtime arguments using the `command` key inside of the `docker-compose.yml` file.

```yaml
version: "3.9"
services:
  sliver:
    container_name: sliver
    image: registry.gitlab.com/ezragit/container-ci_cd/sliver:latest 
    command: "daemon -p 3333"
```

Sweet, now the only thing we still need to do is map the ports. Using the same syntax as the `web` container above, we can create a `ports` list and convert the `docker run` port mappings to a YAML list.

```yaml
version: "3.9"
services:
  sliver:
    container_name: sliver
    image: registry.gitlab.com/ezragit/container-ci_cd/sliver:latest 
    command: "daemon -p 3333"
    ports:
      - "3333:3333/tcp"
      - "3443:443/tcp"
      - "380:80/tcp"
      - "353:53/udp"
```

Since some of the Sliver ports are TCP and some are UDP, I find it is generally a better practice to be explicit in telling Docker what protocol each port will use, but keep in mind that it will default to TCP. But now that we have our Docker Compose definition for Sliver, we can try it out!

### Starting Sliver

Now, in the same directory as our new `docker-compose.yml` file, we can use the `docker-compose up -d` command to start Sliver in detached mode.

{{< figure width="450" height="auto" align=center src="../../blog-images/start-sliver-docker-compose.png" >}}

We can confirm that the container is up by running `docker conatiner ls`.

{{< figure align=center src="../../blog-images/show-running-sliver.png" >}}

And then if we want to stop the container, we can run `docker-compose down`.

{{< figure width="350" height="auto" align=center src="../../blog-images/stop-sliver-docker-compose.png" >}}

Sweet, it works! Wow, I never thought that would actually work.

## Cobalt Strike Conversion

We have Sliver converted, but what if we wanted to convert Cobalt Strike too. For sanity's sake, let's start with a fresh `docker-compose.yml` file to make sure we are isolating any errors to be Cobalt Strike specific.

```bash
echo "" > docker-compose.yml
```

Now that we have a fresh compose file, we will want to follow all the same steps as above, but with the Cobalt Strike `docker run` command. Doing so will give us the following `docker-compose.yml`.

```yml
version: "3.9"
services:
  cobaltstrike:
    container_name: cobaltstrike
    image: registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
    command: "192.168.1.1 password /opt/cobaltstrike/mount/c2.profile"
    ports:
      - "1111:50050/tcp"
      - "1443:443/tcp"
      - "180:80/tcp"
      - "153:53/udp"
```

If we save the `docker-compose.yml` with that content and then use `docker-compose up -d`, we will see the Cobalt Strike container start for about 5 seconds and then exit. This is because we haven't defined the bind mount that will give the container access to the `/opt/cobaltstrike/mount/c2.profile` file. We can fix this by adding a `volumes` key inside of `cobaltstrike` and defining a list item that defines the path to mount inside the container.

```yml
version: "3.9"
services:
  cobaltstrike:
    container_name: cobaltstrike
    image: registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
    command: "192.168.1.1 password /opt/cobaltstrike/mount/c2.profile"
    ports:
      - "1111:50050/tcp"
      - "1443:443/tcp"
      - "180:80/tcp"
      - "153:53/udp"
    volumes:
      - type: bind
        source: /opt/container/cobaltstrike/mount
        target: /opt/cobaltstrike/mount
```

Before we start the container again, we need to make sure the `/opt/container/cobaltstrike/mount/c2.profile` file and path exist. 

```bash 
mkdir -p /opt/container/cobaltstrike/mount && vi /opt/container/cobaltstrike/mount/c2.profile
```

And then just for example's sake, you can paste the following into `c2.profile`.

```
set sample_name "Docker Compose";

http-get {
  set uri "/itstheredteam";
  client {
    metadata {
      netbiosu;
      parameter "tmp";
    }
  }
  server {
    header "Content-Type" "application/octet-stream";
    output {
      print;
    }
  }
}

http-post {
  set uri "/isittheredteam";
  client {
    header "Content-Type" "application/octet-stream";
    id {
      uri-append;
    }
    output {
      print;
    }
  }
  server {
    header "Content-Type" "text/html";
    output {
      print;
    }
  }
}
```

Now, we can run the `docker-compose up -d` command and Cobalt Strike should run with the exact same configuration as our `docker run` command.

## Combining the Container Compose Definitions

We now have two working Docker Compose definitions for our Sliver and Cobalt Strike containers. Now let's combine them! Going back to the example provided from the Docker Compose documentation, we can just combine both the `sliver` and `cobaltstrike` container definitions under the `services` key and that's it!

```yml
version: "3.9"
services:
  sliver:
    container_name: sliver
    image: registry.gitlab.com/ezragit/container-ci_cd/sliver:latest
    command: "daemon -p 3333"
    ports:
      - "3333:3333/tcp"
      - "3443:443/tcp"
      - "380:80/tcp"
      - "353:53/udp"

  cobaltstrike:
    container_name: cobaltstrike
    image: registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
    command: "192.168.1.1 password /opt/cobaltstrike/mount/c2.profile"
    ports:
      - "1111:50050/tcp"
      - "1443:443/tcp"
      - "180:80/tcp"
      - "153:53/udp"
    volumes:
      - type: bind
        source: /opt/container/cobaltstrike/mount
        target: /opt/cobaltstrike/mount
```

Now, in the same directory as our new `docker-compose.yml` file, we can use the `docker-compose up -d` command to start both Sliver and Cobalt Strike in detached mode.

{{< figure width="450" height="auto" align=center src="../../blog-images/start-cs-sliver-docker-compose.png" >}}

We can confirm that the containers are up by running `docker conatiner ls`.

{{< figure align=center src="../../blog-images/show-running-cs-sliver.png" >}}

And then if we want to stop both containers, we can run `docker-compose down`.

{{< figure width="450" height="auto" align=center src="../../blog-images/stop-cs-sliver-docker-compose.png" >}}

And boom, that's it. Now we can easily take this `docker-compose.yml` file to any server and use the exact same syntax to start all of our containers!

# Artifacts

```yml
services:
  cobaltstrike:
    container_name: cobaltstrike
    image: <image_path>
    command: "192.168.1.1 password /opt/cobaltstrike/mount/c2.profile"
    ports:
      - "1111:50050/tcp"
      - "1443:443/tcp"
      - "180:80/tcp"
      - "153:53/udp"
    volumes:
      - type: bind
        source: /opt/container/cobaltstrike/mount
        target: /opt/cobaltstrike/mount

  sliver:
    container_name: sliver
    image: <image_path>
    command: "daemon -p 3333"
    ports:
      - "3333:3333/tcp"
      - "3443:443/tcp"
      - "380:80/tcp"
      - "353:53/udp"
    volumes:
      - type: bind
        source: /opt/container/sliver/mount
        target: /opt/sliver/mount
```