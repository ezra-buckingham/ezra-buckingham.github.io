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

As a Red Teamer, I already have a lot I have to learn and remember. The last thing I want to remember is how to install a specific software on a server (leave that up to the sysadmins of the world, you all are the real heroes). In using a CI/CD pipeline, I can use my single source of truth for building all my software and "automate away the minutiae" of infrastructure work so I can focus on the value-add activities of my job


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

## Creating the Pipeline & Building the Image

Since we are using GitLab to handle the pipeline, all we need to do to enable the repository to integrate with the GitLab CI/CD is add a `.gitlab-ci.yml` file in the root of our repository (as per the [GitLab documentation](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html)). The `.gitlab-ci.yml` file is going to be our workhorse: it will hold all the build and deploy commands for our container images. 

When you look at some of the examples on the web of how to structure the `.gitlab-ci.yml` file, it can get really confusing and scary quickly. However, the one thing to remember is that this file is just a fancy way to run shell commands on a computer. With that, let's look at a simple example of the base `.gitlab-ci.yml` file we will use for our repository!

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

In this configuration, we are telling GitLab exactly how to run the "build" stage of our pipeline (which is the default stage). Within that "build" stage, we are telling GitLab to use the `docker:19.03.12` base container image (yes container-ception, we are using a container to build our containers). You will also see that we are using a "service" called `docker:19.03.12-dind` which now tells GitLab that we want to use the Docker-in-Docker image to execute all of our commands. 










you need to create a new Git repository to hold all of your `Dockerfile`s and any custom configuration files for each container. To do this, you can use the GitLab GUI and then clone down the repository you create.

Continuing with CobaltStrike as our example, we 



One of the most important parts of a good CI/CD pipeline is organization of code. When you are building out all your images and pushing


## Attempt #1 

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

# Artifacts
