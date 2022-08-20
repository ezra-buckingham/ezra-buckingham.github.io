---
title: "Containers Ci_cd"
date: 2022-08-20T07:37:55-04:00
draft: true
description: ""
keywords: []
tags: []
cover:
    image: "blog-covers/"
    alt: ""
    relative: true 
    responsiveImages: true
---



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

