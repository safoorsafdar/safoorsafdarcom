---
title: Source code continues integration of Mendix application with Gitlab Pipeline
date: 2022-04-10T01:05:25.244Z
tags:
  - devops
  - on-premises
  - gitlab-pipeline
  - mendix
  - process-automation
---
Mendix is a high productivity low-code collaborative development app platform that enables you to build and continuously improve mobile and web applications at scale.

Mendix supports the use of a centralized version control repository based on Subversion (SVN), which is the Mendix Team Server. Every project built using the Mendix Platform comes with the Team Server version control system.

The infrastructure technology stack was deployed on-premises, including Kubernetes and Gitlab as two of its main components. The aim was to deploy the Mendix application to Kubernetes in a continuous integration/continuous delivery fashion using the help of Gitlab Pipeline.

Git is a distributed version control system, but Subversion (SVN) is a centralized version control system. Developers may find it difficult to create a Continuous Integration/Continuous Deployment pipeline that converts SVN to Git, as there is no suitable way to implement it in a declarative paradigm.

The application code should be present in the Gitlab repository so that another pipeline process can make it Kubernetes-friendly. This involves preparing a Docker image for deployment and publishing it to the Docker repository, and deploying an application through a Helm chart to the Kubernetes environment.

👇 Here are the step-by-step process of implementing this pipeline.

* Fetch the changes from Mendix Team Server
* Clone the counterpart repository from Gitlab
* Merge the changes from Mendix SVN to the Gitlab repository locally.
* Publish it to Gitlab's respective repository.

> 🚀 You can find the Gitlab Pipeline code at [Source code continues integration of Mendix application with Gitlab Pipeline · GitHub](https://gist.github.com/safoorsafdar/c25505ad69b77f91f6ac90f8b21f44f8)

## Base configuration to start with Gitlab Pipeline

```yaml
# pipeline.yaml
image: alpine
stages:
  - fetch
  - sync
  - push
  - clean
```

👆

* `image` is to define docker-in-docker based image to execute Gitlab pipeline stages. 
* and `stages` is to divide the complete process into multiple steps.

## Fetch the changes from Mendix Team Server

```yaml{3-7}
# pipeline.yaml
variables:
  SVN_SRC_PATH: ${CI_BUILDS_DIR}/${CI_PROJECT_PATH}/svnsrc
  SVN_REPO_PATH: "<path to team server>"
  SVN_USERNAME: "<team server username>"
  SVN_PASSWORD: "<team server password>"
  SVN_TRUNK_FOLDER: "trunk"
```

👆 Above mentioned are some variables defined for SVN to use later in the stage

* `SVN_SRC_PATH` where SVN trunk will download
* `SVN_REPO_PATH` is the complete path to SVN Team Server. That usually starts with `[https://teamserver.sprintr.com/](https://teamserver.sprintr.com/)`.
* `SVN_USERNAME` User Name to access the Team Server.
* `SVN_PASSWORD` Password to access the Team Server.
* `SVN_TRUNK_FOLDER` Folder path on SVN to fetch changes from. Mostly it's the `trunk` folder.

👌 Let's define the stage to download code from Team Server

```yaml
fetch-svn:
  image: nbrun/svn-client:latest
  stage: fetch
  when: always
  before_script:
    - svn --version
    - mkdir -p $SVN_SRC_PATH
  script:
    - echo "svn fetching"
    - svn checkout $SVN_REPO_PATH $SVN_SRC_PATH --trust-server-cert --non-interactive --no-auth-cache --username $SVN_USERNAME --password "$SVN_PASSWORD";
    - cd $SVN_SRC_PATH
    - svn info > ${SVN_TRUNK_FOLDER}/svn.info
  artifacts:
    name: "$CI_JOB_NAME-svnsrc"
    paths:
      - svnsrc/${SVN_TRUNK_FOLDER}/
    expire_in: 1 day
```

👆 What is happening here?

* `image` SVN CLI docker image.
* `svn checkout` will check out the latest changes from the Team server without saving the credentials on the Gitlab runner server.
* `svn info` will save information of the latest fetches from Team Server to `svn.info`. It would help to compare git commit to the SVN release information.

## Clone the counterpart repository from Gitlab

```yaml
variables:
# append below variables in `variables`
  GIT_SRC_PATH: ${CI_BUILDS_DIR}/${CI_PROJECT_PATH}/gitsrc
  GIT_USERNAME: "<git username>"
  GIT_PASSWORD: "<git password>"
  GIT_REPO_BRANCH: "develop"
  GIT_SRC_FOLDER: "mxsrc"
  GIT_REPO_HTTP_PATH: "http://${GIT_USERNAME}:${GIT_PASSWORD}@example.com.ae/test-group/example.git"
```

* `GIT_REPO_BRANCH` is the branch where SVN changes will merge, which could be the "development" or "integration" environment branch.

The next stage is to clone the Gitlab counter repository

```yaml
fetch-git:
  image: pallet/git-client:latest
  stage: fetch
  when: always
  before_script:
    - git --version
    - mkdir -p $GIT_SRC_PATH
  artifacts:
    name: "$CI_JOB_NAME-gitsrc"
    paths:
      - gitsrc/
    expire_in: 1 day
  script:
    - echo "git fetching"
    - git clone --single-branch --branch ${GIT_REPO_BRANCH} ${GIT_REPO_HTTP_PATH} ${GIT_SRC_PATH}
    - cd $GIT_SRC_PATH
    - git config user.email ${GIT_USERNAME}
    - git config user.name "Safoor Safdar"
```

👆 What is happening here?

* `image` git client docker-in-docker based image.
* Script to clone the git repository is pretty straight forward and it will only download a single branch based on the variable `GIT_REPO_BRANCH`.

## Merge the changes from Mendix Team Server to the Gitlab repository

The Gitlab runner server would have two folders, one for Git at `$GIT_SRC_PATH` and the second for SVN source code at `$SVN_SRC_PATH`.

To keep it simple, it can merge into a cloned branch with `rsync`.

```yaml
merge-svn-to-git:
  image: eeacms/rsync:latest
  when: always
  stage: sync
  dependencies:
    - fetch-git
    - fetch-svn
  before_script:
    - rsync --version
    - ls -lah svnsrc/
    - ls -lah gitsrc/
  artifacts:
    name: "$CI_JOB_NAME-sync"
    paths:
      - gitsrc/
    expire_in: 1 day
  script:
    - echo "merging svn into git"
    - rsync -av --progress --delete svnsrc/${SVN_TRUNK_FOLDER}/ gitsrc/${GIT_SRC_FOLDER}
```

* `rsync` will keep the Git repo code as exact as it is on the SVN source folder.
* `dependencies` this stage should happen after downloading of Gitlab counterpart repository code and Mendix Team Server code.

## Publish it to the Gitlab repository

```yaml
pushToGit:
  image: pallet/git-client:latest
  stage: push
  when: always
  dependencies:
    - merge-svn-to-git
  before_script:
    - git --version
    - ls -lah gitsrc/${GIT_SRC_FOLDER}
    - cat gitsrc/${GIT_SRC_FOLDER}/svn.info
  script:
    - echo "pushing to git"
    - cd gitsrc/
    - git add --all .
    - git commit -m "auto commit $(date) by:${SVN_USERNAME}"
    - git push ${GIT_REPO_HTTP_PATH} ${GIT_REPO_BRANCH}
```

👆 It will commit the changes with a custom commit message and push it to the Gitlab repo. This step is only dependent on the `merge-svn-to-git` stage.

> 📝 Communication on Gitlab runner should be open to access SVN Team Server and Gitlab.

Whenever this pipeline is executed, it should be able to fetch the latest changes from Mendix Team Server and push them to the Gitlab repo.

For the final stage, You can "Clean" the temp directories or docker images from the Gitlab runner server.

```yaml
garbag-collector:
  image: docker:19.03.1
  stage: clean
  when: always
  script:
  - echo "cleaning"image: alpine
  - rm -rf ./*
```

💥 You may want to create a temp branch during the process after "rsync" the SVN and Gitlab repository, and based on that temp branch, anyone can open the PR to follow the proper practices to graduate your changes to the Kubernetes environment.

You can learn more about the [CI/CD pipelines | GitLab](https://docs.gitlab.com/ee/ci/pipelines/), and Configuring the Docker in Docker [GitLab Runner | GitLab](https://docs.gitlab.com/runner/) and [Mendix](https://docs.mendix.com/)