---
title: Gitlab DevOps for projects in external repositories
date: '2025-06-30'
draft: false
--- 

If you work in DevOps, you will most likely at some point in time encounter a scenario where you will have to deploy an application that is being actively developed by another team or company.  


If they build docker images on release, everything is fine and dandy, but what if you need to track their master branch instead, or if they don't provide docker images to begin with?  

Working with git yourself, your first idea would probably be to introduce it as a submodule in your project, but [as you may know Git submodules introduce their own set of problems](https://www.feoh.org/posts/git-submodules-are-awful-but-occasionally-necessary.html), which has net them [https://news.ycombinator.com/item?id=31792303](a negative opinion depending on who you ask).

If you work in GitLab, there is another way! You can use [Mirrored Repositories](https://docs.gitlab.com/user/project/repository/mirror/), which enable you to track the remote repository as if it lived in your own architecture. 

However, Mirrored Repositories are not entirely without flaws. While you can select only specific branches or tags to be mirrored, any file you introduce will force you to manually merge each new commit into your custom branch and you will not be able to track your target branch with your custom code any longer. This is especially interesting in the case of automated devops/CI/CD architecture you may want to introduce, since any `.gitlab-ci.yml` file will be overwritten by an automated pull in the mirrored repo.

GitLab also has a solution for this problem though. You can use [GitLab CI/CD for external repositories]. This sets the external repository up as a mirrored one and it also creates a lightweight repository to track your CI/CD configuration. You can't, however, introduce any new assets to the build step in this way.

To solve this issue, we can create a setup like this:

![Gitlab TriProject](../../assets/Gitlab%20Triproject.jpg)

The first project is a vanilla repository mirror without any files changed.
You will configure it's CI/CD file in settings under "General pipeliines" like so. It's important to enable the checkbox for triggering Pipelines on mirror fetches when you set up the mirror.

![Mirror Settings](../../assets/mirror%20settings.png)

You can then use the following Gitlab CI file to build a docker image for your mirrored project. I use Kaniko since our runner exist on a k8s infrastructure.

```
workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^develop.*/'
    - if: '$CI_COMMIT_BRANCH =~ /^release.*/'

stages:
  - prepare-build
  - build
  - trigger-deployment

variables:
  CACHE_TTL: 72h0m0s

.build: &build-docker-image
  stage: build
  dependencies:
    - prepare-build
  image:
    name: gcr.io/kaniko-project/executor:v1.20.0-debug
    entrypoint: [""]
  script:
    - echo "Building..."
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - ts=$(date +%s)
    - >
      if [[ -z "$IMAGE_TAG" ]]; then
        export IMAGE_TAG="$CI_COMMIT_REF_SLUG"
      fi
    - /kaniko/executor
      --registry-mirror=mirror.gcr.io
      --context $CI_PROJECT_DIR
      --dockerfile $DOCKERFILE_NAME
      --destination $CI_REGISTRY_IMAGE/$IMAGE_NAME:$IMAGE_TAG
      --cache=true
      --cache-ttl $CACHE_TTL
      --cache-copy-layers #magic___^_^___line #magic___^_^___line #magic___^_^___line #magic___^_^___line #magic___^_^___line #magic___^_^___line

prepare-build:
  variables:
    ASSET_REPO: bay.fis-build-assets
  image: alpine/git
  stage: prepare-build
  script:
    - echo "Preparing build..."
    - git clone https://${CI_REGISTRY_USER}:${CI_JOB_TOKEN}@git.it.hm.edu/zit/bay.fis/${ASSET_REPO}.git
    - cp -R ${ASSET_REPO}/* .
    - rm -r ${ASSET_REPO}
  artifacts:
    untracked: true


build-bayfis:
  !!merge <<: *build-docker-image
  variables:
    IMAGE_NAME: bayfis
    DOCKERFILE_NAME: Dockerfile.bayfis

build-postgres:
  !!merge <<: *build-docker-image
  variables:
    IMAGE_NAME: postgres
    DOCKERFILE_NAME: Dockerfile.postgres

trigger:
  stage: trigger-deployment
  trigger:
    project: zit/bay.fis/bay.fis-deployment
  variables:
    PIPELINE_COMMIT_REF: $CI_COMMIT_REF_NAME
    PIPELINE_SOURCE_BRANCH: $CI_COMMIT_BRANCH
    PIPELINE_SOURCE_TAG: $CI_COMMIT_TAG
    UPSTREAM_CI_REGISTRY_IMAGE: $CI_REGISTRY_IMAGE
    UPSTREAM_BAYFIS_IMAGE: $CI_REGISTRY_IMAGE/bayfis:$CI_COMMIT_REF_SLUG
    UPSTREAM_POSTGRES_IMAGE: $CI_REGISTRY_IMAGE/postgres:$CI_COMMIT_REF_SLUG
    # RAILS_EXPORTER_IMAGE: $CI_REGISTRY_IMAGE/rails_exporter:latest
```