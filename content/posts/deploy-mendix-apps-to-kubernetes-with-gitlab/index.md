---
title: Deploy Mendix apps to Kubernetes with Gitlab
date: 2022-05-02T23:20:57.708Z
tags:
  - devops
  - on-premises
  - gitlab-pipeline
  - mendix
  - process-automation
---
In this post, You can learn the process to deploy the Mendix application to Kubernetes with the Gitlab Pipeline. It guides you to prepare the Gitlab Pipeline with all appropriate steps.

This is the second post in a series that automates the deployment of Mendix applications using Gitlab. In the first post, the Gitlab pipeline was implemented to move code from the Team server to a Gitlab repository.

You can learn more about moving Mendix applications from Team Server to Gitlab at [Source code continues the integration of Mendix application with Gitlab Pipeline](https://safoorsafdar.com/post/source-code-continues-integration-of-mendix-application-with-gitlab-pipeline).

**A little bit of background about the infrastructure environment...**

Mendix is a low code collaborative development platform for mobile and web applications. The complete infrastructure was provisioned for on-premises, including Kubernetes and Gitlab as two of its main components.

Development, staging, and production environments were provisioned using Kubespray as on-premises infrastructure and the Docker Registry to store Docker images for all mentioned environments.

> :bulb: **Info!** Kubespray is a composition of Ansible playbooks, inventory, provisioning tools, and domain knowledge for generic OS/Kubernetes clusters configuration management tasks. Kubespray provides a highly available cluster. composable attributes. support for most popular Linux distributions.

**You are going to implement...**

👇 This post covers the implementation of Gitlab Pipeline to deploy a Mendix application to Kubernetes. The following steps will be covered:

* Containerize the Mendix application with Docker
* Build the Docker image from the source code
* Publish the Docker Image to Docker Registry
* Define the Mendix application deployment template
* Deploy the application to the development and staging environment
* Deploy the application to the production environment
* Clean the local Docker images

## Containerize the Mendix application with Docker

The Mendix build pack for docker provided a standard way to build and run your Mendix application in a Docker container. You can learn more about the build pack at [GitHub - mendix/docker-mendix-buildpack: Build and Run Mendix in Docker](https://github.com/mendix/docker-mendix-buildpack)

Mendix Build pack will help you to prepare to containerize the version of your application. It can be part of your source code from start on Team Server or you can merge required docker files later in the process. In this post, let's assume this part is already implemented.

Let's start building Gitlab Pipeline to deploy the Mendix application to Kubernetes.

> 🚀  You can find the Gitlab Pipeline code at [Deploy Mendix apps to Kubernetes with Gitlab · GitHub](https://gist.github.com/safoorsafdar/c169d5007e1aa88d900ae7198114292f)

👇 Base configuration to start with Pipeline...

```yaml
image: alpine
stages:
  - build
  - publish
  - deploy
  - clean
```

👆 `image` is to define docker-in-docker based image to execute Gitlab pipeline stages in. and `stages` is to divide the complete process into multiple steps.

and custom CI/CD variables to use later in pipeline implementation.

```yaml
variables:
  MENDIX_BUILD_PATH: mxsrc
  MENDIX_BUILDPACK_VERSION: v3.6.4
  CONTAINER_IMAGE_NAME: ${REGISTRY_IP}/${CI_PROJECT_PATH}
  CONTAINER_IMAGE_TAG: ${CI_BUILD_REF_NAME}_${CI_COMMIT_SHORT_SHA}
  CONTAINER_IMAGE: ${CONTAINER_IMAGE_NAME}:${CONTAINER_IMAGE_TAG}
  CONTAINER_IMAGE_LATEST: ${CONTAINER_IMAGE_NAME}:latest
  KUBECONFIG: /etc/deploy/config
  DEV_NS: apps-dev
  STG_NS: apps-stg
  PRD_NS: apps-prod
  CHART_PATH: mxchart
```

👆 What are these variables?

* `MENDIX_BUILD_PATH` Mendix application source code path on the Gitlab Runner server.
* `MENDIX_BUILDPACK_VERSION` is the version of the Mendix Build pack.
* `REGISTRY_IP` contains the IP of Docker Registry to access it from the Gitlab Runner server.
* `CONTAINER_*` are the supported variables to name the docker image.
* `CI_PROJECT_PATH` is a predefined variable from Gitlab that provides the project namespace with the project name.
* `CI_BUILD_REF_NAME` is to get the current build ref name. 
  *Note: This variable does not exist anymore. Review Gitlab Predefined Variables.*
* `CI_COMMIT_SHORT_SHA` is the first eight characters of the commit SHA `CI_COMMIT_SHA` variable.
* `KUBECONFIG` is the Kubernetes config file path to store the Kubernetes configuration.
* `DEV_NS`, `STG_NS`, and `PRD_NS` are the Kubernetes namespace names.
* `CHART_PATH` is the path of the Helm chart on the Gitlab runner server.

## Build the Docker image from the source code

```yaml
# Build Stage to build Mendix application docker image
build:
  image: docker:19.03.1
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  services:
    - docker:19.03.1-dind
  stage: build
  only:
    - develop
    - master
  before_script:
    - docker info --format '{{json .}}'
    - echo ${CONTAINER_IMAGE} # just to make sure variables are populating
    - echo ${CONTAINER_IMAGE_LATEST}
  script:
    - docker build --build-arg BUILD_PATH=$MENDIX_BUILD_PATH --build-arg CF_BUILDPACK=$MENDIX_BUILDPACK_VERSION -t ${CONTAINER_IMAGE_NAME}:${CONTAINER_IMAGE_TAG} .
    - docker tag ${CONTAINER_IMAGE_NAME}:${CONTAINER_IMAGE_TAG} ${CONTAINER_IMAGE_NAME}:latest
```

👆 What is happening here?

* This stage will only execute for `master` and `develop` branches of the repository.
* `services` is to bind base docker image for docker-in-docker execution, it will execute script in `image: docker:19.03.1` docker image. 
* It will prepare the build of the source code using `docker build` command and then tag it with `docker tag` command. You can find more documentation on `docker build` command on the Mendix Build pack.
* `DOCKER_TLS_CERTDIR` is to let docker-in-docker communicate over TLS with host docker.

## Publish the Docker Image to Docker Registry

```yaml
# Publish the build docker image to the docker repository
publish:
  stage: publish
  image: docker:19.03.1
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  services:
    - docker:19.03.1-dind
  only:
    - develop
    - master
  before_script:
    - docker info
    - echo "{\"insecure-registries\":[\"$REGISTRY_IP\"]} >> ~/.docker/daemon.json"
  script:
    - docker push ${CONTAINER_IMAGE_NAME}:${CONTAINER_IMAGE_TAG}
    - docker push ${CONTAINER_IMAGE_NAME}:latest
  retry: 2
  allow_failure: true
```

👆 What is happening here?

* It will push the build docker image to the Docker registry.
* It will try to run the same stage twice if some error occurs during the execution to push the Docker image to the Docker registry. 
* Docker registry was deployed on on-premises infrastructure to communicate over the IP instead of DNS. SSL/HTTPS certificate was not provisioned for secure communication. `before_script` will automatically add `$REGISTRY_IP` to insecure communication for Docker with Docker registry on Gitlab Runner server.

## Define the Mendix application deployment template

```yaml
# Deployement template for multiple environment
.deploy_template: &deploy_template_def
  stage: deploy
  before_script:
    - mkdir -p /etc/deploy
    - echo ${KUBECONFIG_DATA} | base64 -d > ${KUBECONFIG}
    - helm init --kubeconfig ${KUBECONFIG} --service-account tiller --client-only
    - helm version

  script:
    - export API_VERSION="$(grep "appVersion" ${CHART_PATH}/Chart.yaml | cut -d" " -f2 | sed -e 's/^"//' -e 's/"$//')"
    - echo ${API_VERSION}
    - export RELEASE_NAME=${APP_NAME}-v${API_VERSION/./-}
    - echo ${RELEASE_NAME}
    - export DEPLOYS=$(helm ls | grep $RELEASE_NAME | wc -l)
    - echo ${DEPLOYS}
    - if [ ${DEPLOYS} -eq 0 ]; then helm install ${CHART_PATH} --namespace=${NAMESPACE} --name=${RELEASE_NAME} --set nameOverride=${APP_NAME} --set image.repository=${CONTAINER_IMAGE_NAME} --set image.tag=${CONTAINER_IMAGE_TAG} --set ENV_LICENSE_ID=${ENV_LICENSE_ID} --set ENV_LICENSE_KEY=${ENV_LICENSE_KEY} --set ingress.annotations."nginx\.ingress\.kubernetes\.io/session-cookie-path"=${EP_PATH} --set ingress.paths[0]=${EP_PATH} --set ingress.hosts[0]=${EP_HOST} --set ENV_ADMIN_PASSWORD=${ADMIN_PASSWORD} --set ENV_MXRUNTIME_DATABASETYPE=${MXRUNTIME_DATABASETYPE} --set ENV_MXRUNTIME_DATABASEJDBCURL=${MXRUNTIME_DATABASEJDBCURL} --set ENV_MXRUNTIME_DATABASEUSERNAME=${MXRUNTIME_DATABASEUSERNAME} --set ENV_MXRUNTIME_DATABASEPASSWORD=${MXRUNTIME_DATABASEPASSWORD}; else echo "verion found; upgrading"; helm upgrade ${RELEASE_NAME} ${CHART_PATH} --namespace=${NAMESPACE} --set image.repository=${CONTAINER_IMAGE_NAME} --set image.tag=${CONTAINER_IMAGE_TAG} --set ingress.annotations."nginx\.ingress\.kubernetes\.io/session-cookie-path"=${EP_PATH} --set ingress.paths[0]=${EP_PATH} --set ingress.hosts[0]=${EP_HOST}; fi
```

👆 What is happening here?

* At the time of implementation, Helm 2 version was used. 
* It will save the Kubernetes config data from the `${KUBECONFIG_DATA}` to the `${KUBECONFIG}` path.
* `helm init` will initialize the Helm client with the Kubernetes cluster. This command assumes Helm V2 has been configured on the Kubernetes cluster. So, it will only configure `--client-only` Helm Client on the Gitlab runner server.
* As a core part of the implementation for this stage, You will be updating the Helm chart release on Kubernetes or creating a new release for your Helm chart.
* `script` get's the Helm chart version and check if it is already deployed on the Kubernetes cluster. If not, it will install the Helm chart on Kubernetes.