---
title: "DevAttackOps: Container CI/CD Pipelines"
date: 2022-08-26T07:37:55-04:00
draft: true
description: "Build your containers in CI/CD Pipelines"
keywords: ["red-team", "infrastructure", "docker", "containers", "devattackops", "gitlab"]
tags: ["red-team", "infrastructure", "docker", "containers", "devattackops", "gitlab"]
cover:
    image: "blog-covers/ci-cd_containers.png"
    alt: "CI/CD"
    relative: true 
    responsiveImages: true
---

**TLDR;** Building CI/CD pipelines for containers is not as daunting as it seems. For GitLab specifically, I have included a GitLab CI/CD configuration to automatically build and deploy your custom container images in the [Artifacts](#artifacts) section.

**Disclaimer:** If you haven't read [DevAttackOps Part 1](../containerizing-red-team-infra), you should start there as we will use the CobaltStrike example we built in that post. 

Welcome to part 2 of the DevAttackOps series where I talk all things Red Team infrastructure automation. In [DevAttackOps Part 1](../containerizing-red-team-infra), I showed how its possible to take a C2 framework and package it up into a container image. In this post, I will talk about how you can expand on that and build out a CI/CD pipeline to automatically build and deploy those images to a container registry where they can be accessed programmatically. 

If you work in IT in any capacity, you have heard the term "DevOps" or "Continuous Integration and Continuous Deployment" (CI/CD for short). These terms are the bread and butter for any junior engineer to throw on a resume and have absolutely zero clue what they actually mean. Today, I will walk you through what they mean to me as a Red Teamer and why you should care too (and my hope is that you integrate a similar solution and then put CI/CD on your resume, just like the junior engineers).

# Key Terms

* CI/CD: a method to frequently deliver apps to customers by introducing automation into the stages of app development (Source: [RedHat](https://www.redhat.com/en/topics/devops/what-is-ci-cd))
* Container Registry: a single place for your team to manage Docker images (Source: [Google Cloud](https://cloud.google.com/container-registry))

# Why Build a CI/CD Pipeline?

If you are reading this, you clearly don't want to be that junior engineer who loves putting buzzwords on their resume (speaking of which I should probably remove BlockChain from mine, but I digress). Let me convince you _why_ you should care. In my mind, the reasons why you should care are the same as in [DevAttackOps Part 1](../containerizing-red-team-infra): repeatability and ease of use. However, I will add one more here: reduce complexity.

## Repeatability

In a CI/CD pipeline, you have a single source of truth for building and deploying all your images. That means that you will never need to "take notes" on what build arguments you need to use to build a container image. Using a single, well-defined pipeline means that your builds will do the _exact same_ thing each build and there is less room for human error.

## Ease of Use

Using a CI/CD pipeline means you have control over _how you build_ and _where you deploy to_. For containers, that means you can automatically deploy all container images to a registry and use _one line of code_ to pull that image down and run that container. This is powerful as it allows anyone of any skill level to use your container images (as long as they can authenticate to your registry).

## Reduce Complexity

As a Red Teamer, I already have a lot I have to learn and remember. The last thing I want to remember is how to install a specific software on a server (leave that up to the sysadmins of the world, you all are the real heroes). In using a CI/CD pipeline, I can use my single source of truth for building all my software and "automate away the minutiae" of infrastructure work so I can focus on the value-add activities of my job.


# Building the Pipeline

As we go through building out this CI/CD pipeline, I want to remind you that _this is not the only solution_. There are may different ways to skin the cat (sorry I hate that phrase too), but I want to provide a tangible example for you to go try on your own. In doing so, I will walk you through how you build out a CI/CD pipeline using a GitLab repository as the source and GitLab container registry as the container registry (both of which are **FREE** so you have no excuses to not build this yourself).

## Creating the Repository

Before you can go building your pipeline, we need to think through how we want to organize our code. What I found is that the best way to manage all the container definitions is by having one single git repository that has a folder for each container. So let's start with creating the repository.

```bash
mkdir container-ci_cd
cd container-ci_cd
git init
```

## Adding your First Container Definition

Inside of this new repository, we can now create a `cobaltstrike` folder to hold our Cobalt Strike `Dockerfile`.

```bash
mkdir cobaltstrike
touch Dockerfile
```

Using your favorite IDE or text editor, you can now edit that `Dockerfile` you just created to hold the same `Dockerfile` you use to build Cobalt Strike. And that's really all you need to do! We will add more containers in a bit.

## Creating a Basic Pipeline

Since we are using GitLab to handle the pipeline, all we need to do to enable the repository to integrate with the GitLab CI/CD is add a `.gitlab-ci.yml` file in the root of our repository (as per the [GitLab documentation](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html)). The `.gitlab-ci.yml` file is going to be our workhorse: it will hold all the build and deploy commands for our container images. 

When you look at some of the examples on the web of how to structure the `.gitlab-ci.yml` file, it can get really confusing and scary quickly. However, the one thing to remember is that this file is just a fancy way to run shell commands on a computer that GitLab will spin up whenever you commit to your repository (or when you manually kick off the build). With that, let's look at a simple example of the base `.gitlab-ci.yml` file we will use for our repository!

```yaml
build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
  before_script:
    - echo "Before"
  script:
    - echo "During"
```

In this configuration, we are telling GitLab exactly how to run the "build" stage of our pipeline (which is the default stage). Within that "build" stage, we are telling GitLab to use the `docker:19.03.12` base container image (yes container-ception, we are using a container to build our containers). You will also see that we are using a "service" called `docker:19.03.12-dind` which now tells GitLab that we want to use the Docker-in-Docker image to execute the container-building commands. Think of the `services` definition as the "tools" we need to successfully run our commands. On a traditional server, in order for us to build containers, we would need to install Docker onto that system. However, with GitLab's pipeline, we can just use another container (with all required tools we need already installed on it) to run each command. If we then commit that file up to GitLab, we see that the job will pull back that container image and run our commands.

{{< figure align=center src="../../blog-images/test-cicd-build.png" >}}

> I realize that may have lost you, so if you want to check out [GitLab's documentation](https://docs.gitlab.com/ee/ci/services/) to fully understand their concept of "services", they do a great job explaining the technical details.

## Understanding the Container Registry

Sweet so now we are using GitLab's CI/CD to execute commands whenever we make changes to the repository. Now we want to make it do something useful, like build our Cobalt Strike container. Since we already know the command to build the container, we can use that as the starting point. However, since we are going to be deploying the container image up to GitLab's container registry, we need to understand how it works. In looking at the [GitLab Container Registry documentation](https://docs.gitlab.com/ee/user/packages/container_registry/), it seems like a standard registry. Only caveat here is that in order to use the registry, we need to use a "personal access token" to sign into the registry before we can push up to it. Before trying to use all these new tools _together_ its important to understand one at a time. 

To start using the registry, we need to generate a personal access token. You can do this by going to your user preferences, selecting "Access Tokens" and generating a new token with the scopes `write_registry` and `read_registry`.

{{< figure align=center src="../../blog-images/generate-pat-gitlab.png" >}}

Once you hit create, you should get a token that starts with `glpat-` of which will now allow you to login to the GitLab registry using the docker CLI.

```bash
docker login -u ezra-buckingham -p glpat-<...redacted...> registry.gitlab.com
```

Now, we can use that Cobalt Strike Dockerfile we created in the `cobaltstrike` folder earlier, build it with a tag that points to our new repository called `container-ci_cd`, and push it up to that repository's registry (note that `ezragit` refers to my gitlab organization and `container_ci-cd` refers to the repository that will be my registry).

```bash
docker build -t registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
docker push registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
```

After running those commands, we can see the Cobalt Strike container in our GitLab registry.

{{< figure align=center src="../../blog-images/container-in-registry.png" >}}

Now to use that container image, any user that has access to that repository can create their own "personal access token" with the `registry_read` scope and can use the same docker login CLI command to login to registry.gitlab.com and then pull down the container image.

```bash
docker run registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
```

## Using the Pipeline to Build and Push the Image

We now have an understanding of the registry and a basic implementation of the CI/CD pipeline. Now, let's combine the two. If we take our `.gitlab-ci.yml` file and copy the commands we just ran into it, everything would work right? Well not exactly... Since the Cobalt Strike container takes in a `build-arg`, we need to somehow pass that into the CI/CD job. This is where GitLab CI/CD variables come into play. Inside the CI/CD settings of your repository, you can store secrets like the `COBALTSTRIKE_LICENSE` and pull them into the job.

{{< figure align=center src="../../blog-images/add-cicd-variable.png" >}}

However, to pull that secret into the job, you must explicitly put that variable into the `variables` block so that it can be passed into the build commands.

```yaml
build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
  variables:
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  script:
    - docker build -t registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push registry.gitlab.com/ezragit/container-ci_cd/cobaltstrike:latest
```

Now that we have this, we can commit and push these changes to try out our fancy new build command. But just this isn't enough, if we push this and see the results, we get an access forbidden when we try to push the image. 

{{< figure align=center src="../../blog-images/cicd-failed-push.png" >}}

This is because we need to authenticate to the registry. What's cool is that we don't need to generate any credentials, we can use the `secrets` embedded in the runner to dynamically authenticate to the registry (depending on what user made the change). We can do this using the `CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD` secrets in a docker login command. However, while we are at it, we should also remove some of the hardcoded values too. We can also leverage the `CI_REGISTRY` to dynamically point to the registry and even create a new variable with the full path to our registry and then reference that in the docker build and push.

```yaml
build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
  variables:
    CI_REGISTRY_PATH: $CI_REGISTRY/ezragit/container-ci_cd
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest
```

And when we push these changes, we can see the job completes successfully and automatically builds and deploys our image to our private registry!

{{< figure align=center src="../../blog-images/test-cicd-container-build.png" >}}

# Expanding the Capabilities

Now that we have a way to automate this, what if we wanted to add another image to our registry? It's easy! Let's build Sliver into our registry. First, we need a new folder for sliver so we can hold our Dockerfile.

```bash
mkdir sliver
touch Dockerfile
```

We can (using skills learned in Part 1) build out that Dockerfile to build our Sliver container image.

```Dockerfile
FROM debian:stable-slim

RUN apt-get update \
    && apt-get -y install git wget zip tar file mingw-w64

WORKDIR /opt/sliver

RUN wget https://github.com/BishopFox/sliver/releases/download/v1.5.16/sliver-server_linux \
    && mv sliver-server_linux sliver-server \
    && chmod +x ./sliver-server

WORKDIR /opt/sliver
EXPOSE 3333 443 80 53/udp
ENTRYPOINT [ "./sliver-server" ]
```

Then we can add the commands needed to build and push the image directly into our `.gitlab-ci.yml` file.

```yaml
build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
  variables:
    CI_REGISTRY_PATH: $CI_REGISTRY/ezragit/container-ci_cd
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest
    - docker build -t $CI_REGISTRY_PATH/sliver:latest ./sliver
    - docker push $CI_REGISTRY_PATH/sliver:latest
```

And yes, it is that easy to build your own CI/CD pipeline to build and deploy container images to a private container registry.

## The Problems with the Solution

We have an awesome start to what is becoming a badass CI/CD pipeline, but we can do better. There are 2 big problems with what we have come up with: failure tracing and speed (There are more, but topic for another time as it gets deeper into the weeds).

### Failure Tracing

In our current solution, if any of the builds or pushes fail, the rest of the pipeline fails. This makes it difficult to track down which container image failed to build and what that failure was.

### Speed

Since all of our commands run in sequence, each build needs to wait until the previous one finishes to start building. 

## Improving the Solution

We can alleviate both of those problems by splitting out each container build into their own build step, but tie them both into the parent "build" step. Using some YAML magic, we can create an anchor and then reference that anchor in each build so that there is a "global" job configuration for each build (if you want to learn more about YAML anchors, here's a great [blog post](https://medium.com/@kinghuang/docker-compose-anchors-aliases-extensions-a1e4105d70bd) on them).

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
    CI_REGISTRY_PATH: $CI_REGISTRY/ezragit/container-ci_cd
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script: 
   - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Actual Builds
build_cobaltstrike:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest

build_sliver:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/sliver:latest ./sliver
    - docker push $CI_REGISTRY_PATH/sliver:latest
```

With this new and improved CI/CD configuration, when we push to this repository, we will see logically each container build split out and can see which is the "trouble child" if the entire pipeline fails. Not to mention, now all the containers can be built in parallel. This means that all images are built at the same time.

{{< figure align=center src="../../blog-images/cicd-parallel-builds.png" >}}

# Artifacts

Here is the `.gitlab-ci.yml` file used to build all the containers.

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
    CI_REGISTRY_PATH: $CI_REGISTRY/ezragit/container-ci_cd
    COBALTSTRIKE_LICENSE: $COBALTSTRIKE_LICENSE
  before_script: 
   - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Actual Builds
build_cobaltstrike:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/cobaltstrike:latest --build-arg COBALTSTRIKE_LICENSE=$COBALTSTRIKE_LICENSE ./cobaltstrike
    - docker push $CI_REGISTRY_PATH/cobaltstrike:latest

build_sliver:
  <<: *job_configuration
  script:
    - docker build -t $CI_REGISTRY_PATH/sliver:latest ./sliver
    - docker push $CI_REGISTRY_PATH/sliver:latest

```