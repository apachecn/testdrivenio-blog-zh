# 通过 Docker 层缓存和构建工具包加快 CI 构建

> 原文：<https://testdriven.io/blog/faster-ci-builds-with-docker-cache/>

本文着眼于如何使用 Docker 层缓存和[构建工具包](https://docs.docker.com/develop/develop-images/build_enhancements/)来加速您在 [CircleCI](https://circleci.com/) 、 [GitLab CI](https://docs.gitlab.com/ee/ci/) 和 [GitHub Actions](https://github.com/features/actions) 上基于 Docker 的构建。

## Docker 层缓存

Docker 会在构建映像时缓存每一层，只有在自上次构建以来该层或其上的层发生了变化时，才会重新构建每一层。因此，您可以使用 Docker 缓存显著加快构建速度。让我们看一个简单的例子。

*Dockerfile* :

```
`# pull base image
FROM  python:3.9.7-slim

# install netcat
RUN  apt-get update && \
    apt-get -y install netcat && \
    apt-get clean

# set working directory
WORKDIR  /usr/src/app

# install requirements
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# add app
COPY  . .

# run server
CMD  gunicorn -b 0.0.0.0:5000 manage:app` 
```

> 您可以在 GitHub 上的 [docker-ci-cache](https://github.com/testdrivenio/docker-ci-cache) repo 中找到该项目的完整源代码。

第一次 Docker 构建可能需要几分钟才能完成，这取决于您的连接速度。后续构建应该只需要几秒钟，因为层在第一次构建后会被缓存:

```
`[+] Building 0.4s (12/12) FINISHED
 => [internal] load build definition from Dockerfile                                                                     0.0s
 => => transferring dockerfile: 37B                                                                                      0.0s
 => [internal] load .dockerignore                                                                                        0.0s
 => => transferring context: 35B                                                                                         0.0s
 => [internal] load metadata for docker.io/library/python:3.9.7-slim                                                     0.3s
 => [internal] load build context                                                                                        0.0s
 => => transferring context: 555B                                                                                        0.0s
 => [1/7] FROM docker.io/library/python:[[email protected]](/cdn-cgi/l/email-protection):bdefda2b80c5b4d993ef83d2445d81b2b894bf627b62bd7b0f01244de2b6a  0.0s
 => CACHED [2/7] RUN apt-get update &&     apt-get -y install netcat &&     apt-get clean                                0.0s
 => CACHED [3/7] WORKDIR /usr/src/app                                                                                    0.0s
 => CACHED [4/7] COPY ./requirements.txt .                                                                               0.0s
 => CACHED [5/7] RUN pip install -r requirements.txt                                                                     0.0s
 => CACHED [6/7] COPY project .                                                                                          0.0s
 => CACHED [7/7] COPY manage.py .                                                                                        0.0s
 => exporting to image                                                                                                   0.0s
 => => exporting layers                                                                                                  0.0s
 => => writing image sha256:2b8b7c5a6d1b77d5bcd689ab265b0281ad531bd2e34729cff82285f5abdcb59f                             0.0s
 => => naming to docker.io/library/cache                                                                                 0.0s` 
```

即使您对源代码进行了更改，也应该只需要几秒钟就可以完成构建，因为不需要下载依赖项。只有最后两层需要重建，换句话说:

```
 `=> [6/7] COPY project .
 => [7/7] COPY manage.py .` 
```

要避免缓存失效:

1.  用不太可能改变的命令开始你的 docker 文件
2.  尽可能晚地放置更有可能改变的命令(如`COPY . .`)
3.  仅添加必要的文件(使用*)。dockerignore* 文件)

> 要获得更多的技巧和最佳实践，请查看针对 Python 开发人员的 Docker 最佳实践文章。

## 构建工具包

如果你使用的是 Docker 版本> = [19.03](https://docs.docker.com/engine/release-notes/#19030) ，你可以使用 BuildKit，一个容器映像构建器，来代替 Docker 引擎中传统的映像构建器后端。如果没有 BuildKit，如果本地图像注册表中不存在图像，您需要在构建之前提取远程图像，以便利用 Docker 层缓存。

示例:

```
`$ docker pull mjhea0/docker-ci-cache:latest

$ docker docker build --tag mjhea0/docker-ci-cache:latest .` 
```

使用 BuildKit，您不需要在构建之前提取远程映像，因为它会在映像注册表中缓存每个构建层。然后，当您构建映像时，在构建过程中会根据需要下载每个层。

要启用 BuildKit，请将`DOCKER_BUILDKIT`环境变量设置为`1`。然后，打开内嵌层缓存，使用`BUILDKIT_INLINE_CACHE`构建参数。

示例:

```
`export DOCKER_BUILDKIT=1

# Build and cache image
$ docker build --tag mjhea0/docker-ci-cache:latest --build-arg BUILDKIT_INLINE_CACHE=1 .

# Build image from remote cache
$ docker build --cache-from mjhea0/docker-ci-cache:latest .` 
```

## CI 环境

因为 CI 平台为每个构建提供了一个全新的环境，所以您需要使用一个远程图像注册表作为 BuildKit 的层缓存的缓存源。

步骤:

1.  登录图像注册中心(如[码头中心](https://hub.docker.com/)、[弹性集装箱注册中心](https://aws.amazon.com/ecr/) (ECR)、以及[码头](https://quay.io/)等等)。

    > 值得注意的是，GitLab 和 GitHub 都有自己的注册表，可以在平台上的库(公共和私有)中使用，分别是 [GitLab 容器注册表](https://docs.gitlab.com/ee/user/packages/container_registry/)和 [GitHub 包](https://github.com/features/packages)。

2.  使用 Docker build 的`--cache-from` [选项](https://docs.docker.com/engine/reference/commandline/build/#options)将现有图像用作缓存源。

3.  如果构建成功，将新的映像推送到注册表中。

让我们看看如何在 CircleCI、GitLab CI 和 GitHub Actions 上做到这一点，使用单级和多级 Docker 构建，使用和不使用 Docker Compose。每个示例都使用 Docker Hub 作为映像注册中心，并将`REGISTRY_USER`和`REGISTRY_PASS`设置为 CI 构建中的变量，以便向注册中心推送数据或从中提取数据。

> 确保在构建环境中将`REGISTRY_USER`和`REGISTRY_PASS`设置为环境变量:
> 
> 1.  循环
> 2.  [GitLab CI](https://docs.gitlab.com/ee/ci/variables/)
> 3.  [GitHub 动作](https://help.github.com/en/actions/configuring-and-managing-workflows/using-variables-and-secrets-in-a-workflow)

## 单阶段构建

圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈：

```
`# _config-examples/single-stage/circle.yml version:  2.1 jobs: build: machine: image:  ubuntu-2004:202010-01 environment: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 steps: -  checkout -  run: name:  Log in to docker hub command:  docker login -u $REGISTRY_USER -p $REGISTRY_PASS -  run: name:  Build from dockerfile command:  | docker build \ --cache-from $CACHE_IMAGE:latest \ --tag $CACHE_IMAGE:latest \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  run: name:  Push to docker hub command:  docker push $CACHE_IMAGE:latest` 
```

GitLab CI:

```
`# _config-examples/single-stage/.gitlab-ci.yml image:  docker:stable services: -  docker:dind variables: DOCKER_DRIVER:  overlay2 CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 stages: -  build docker-build: stage:  build before_script: -  docker login -u $REGISTRY_USER -p $REGISTRY_PASS script: -  docker build --cache-from $CACHE_IMAGE:latest --tag $CACHE_IMAGE:latest --file ./Dockerfile --build-arg BUILDKIT_INLINE_CACHE=1 "." after_script: -  docker push $CACHE_IMAGE:latest` 
```

GitHub 操作:

```
`# _config-examples/single-stage/github.yml name:  Docker Build on:  [push] env: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 jobs: build: name:  Build Docker Image runs-on:  ubuntu-latest steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Log in to docker hub run:  docker login -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_PASS }} -  name:  Build from dockerfile run:  | docker build \ --cache-from $CACHE_IMAGE:latest \ --tag $CACHE_IMAGE:latest \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  name:  Push to docker hub run:  docker push $CACHE_IMAGE:latest` 
```

### 构成

如果您正在使用 Docker Compose，您可以将`cache_from` [选项](https://docs.docker.com/compose/compose-file/compose-file-v3/#cache_from)添加到 Compose 文件，当您运行`docker-compose build`时，它会映射回`docker build --cache-from <image>`命令。

示例:

```
`version:  '3.8' services: web: build: context:  . cache_from: -  mjhea0/docker-ci-cache:latest image:  mjhea0/docker-ci-cache:latest` 
```

为了利用 BuildKit，请确保您使用的是 Docker Compose >= [1.25.0](https://github.com/docker/compose/releases/tag/1.25.0) 版本。要启用 BuildKit，请将`DOCKER_BUILDKIT`和`COMPOSE_DOCKER_CLI_BUILD`环境变量设置为`1`。然后，再次打开内嵌层缓存，使用`BUILDKIT_INLINE_CACHE` build 参数。

圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈：

```
`# _config-examples/single-stage/compose/circle.yml version:  2.1 jobs: build: machine: image:  ubuntu-2004:202010-01 environment: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 steps: -  checkout -  run: name:  Log in to docker hub command:  docker login -u $REGISTRY_USER -p $REGISTRY_PASS -  run: name:  Build images command:  docker-compose build --build-arg BUILDKIT_INLINE_CACHE=1 -  run: name:  Push to docker hub command:  docker push $CACHE_IMAGE:latest` 
```

GitLab CI:

```
`# _config-examples/single-stage/compose/.gitlab-ci.yml image:  docker/compose:latest services: -  docker:dind variables: DOCKER_DRIVER:  overlay2 CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 stages: -  build docker-build: stage:  build before_script: -  docker login -u $REGISTRY_USER -p $REGISTRY_PASS script: -  docker-compose build --build-arg BUILDKIT_INLINE_CACHE=1 after_script: -  docker push $CACHE_IMAGE:latest` 
```

GitHub 操作:

```
`# _config-examples/single-stage/compose/github.yml name:  Docker Build on:  [push] env: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 jobs: build: name:  Build Docker Image runs-on:  ubuntu-latest steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Log in to docker hub run:  docker login -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_PASS }} -  name:  Build Docker images run:  docker-compose build --build-arg BUILDKIT_INLINE_CACHE=1 -  name:  Push to docker hub run:  docker push $CACHE_IMAGE:latest` 
```

## 多阶段构建

使用[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)模式，您必须对每个中间阶段应用相同的工作流(构建，然后推送),因为这些映像在最终映像创建之前就被丢弃了。`--target` [选项](https://docs.docker.com/compose/compose-file/compose-file-v3/#target)可用于单独构建多阶段构建的每个阶段。

*Dockerfile.multi* :

```
`# base
FROM  python:3.9.7  as  base
COPY  ./requirements.txt /
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

# stage
FROM  python:3.9.7-slim
RUN  apt-get update && \
    apt-get -y install netcat && \
    apt-get clean
WORKDIR  /usr/src/app
COPY  --from=base /wheels /wheels
COPY  --from=base requirements.txt .
RUN  pip install --no-cache /wheels/*
COPY  . /usr/src/app
CMD  gunicorn -b 0.0.0.0:5000 manage:app` 
```

圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈：

```
`# _config-examples/multi-stage/circle.yml version:  2.1 jobs: build: machine: image:  ubuntu-2004:202010-01 environment: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 steps: -  checkout -  run: name:  Log in to docker hub command:  docker login -u $REGISTRY_USER -p $REGISTRY_PASS -  run: name:  Build base from dockerfile command:  | docker build \ --target base \ --cache-from $CACHE_IMAGE:base \ --tag $CACHE_IMAGE:base \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  run: name:  Build stage from dockerfile command:  | docker build \ --cache-from $CACHE_IMAGE:base \ --cache-from $CACHE_IMAGE:stage \ --tag $CACHE_IMAGE:stage \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  run: name:  Push base image to docker hub command:  docker push $CACHE_IMAGE:base -  run: name:  Push stage image to docker hub command:  docker push $CACHE_IMAGE:stage` 
```

GitLab CI:

```
`# _config-examples/multi-stage/.gitlab-ci.yml image:  docker:stable services: -  docker:dind variables: DOCKER_DRIVER:  overlay2 CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 stages: -  build docker-build: stage:  build before_script: -  docker login -u $REGISTRY_USER -p $REGISTRY_PASS script: -  docker build --target base --cache-from $CACHE_IMAGE:base --tag $CACHE_IMAGE:base --file ./Dockerfile.multi --build-arg BUILDKIT_INLINE_CACHE=1 "." -  docker build --cache-from $CACHE_IMAGE:base --cache-from $CACHE_IMAGE:stage --tag $CACHE_IMAGE:stage --file ./Dockerfile.multi --build-arg BUILDKIT_INLINE_CACHE=1 "." after_script: -  docker push $CACHE_IMAGE:stage` 
```

GitHub 操作:

```
`# _config-examples/multi-stage/github.yml name:  Docker Build on:  [push] env: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 jobs: build: name:  Build Docker Image runs-on:  ubuntu-latest steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Log in to docker hub run:  docker login -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_PASS }} -  name:  Build base from dockerfile run:  | docker build \ --target base \ --cache-from $CACHE_IMAGE:base \ --tag $CACHE_IMAGE:base \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  name:  Build stage from dockerfile run:  | docker build \ --cache-from $CACHE_IMAGE:base \ --cache-from $CACHE_IMAGE:stage \ --tag $CACHE_IMAGE:stage \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  name:  Push base image to docker hub run:  docker push $CACHE_IMAGE:base -  name:  Push stage image to docker hub run:  docker push $CACHE_IMAGE:stage` 
```

### 构成

示例合成文件:

```
`version:  '3.8' services: web: build: context:  . cache_from: -  mjhea0/docker-ci-cache:stage image:  mjhea0/docker-ci-cache:stage` 
```

圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈圈：

```
`# _config-examples/multi-stage/compose/circle.yml version:  2.1 jobs: build: machine: image:  ubuntu-2004:202010-01 environment: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 steps: -  checkout -  run: name:  Log in to docker hub command:  docker login -u $REGISTRY_USER -p $REGISTRY_PASS -  run: name:  Build base from dockerfile command:  | docker build \ --target base \ --cache-from $CACHE_IMAGE:base \ --tag $CACHE_IMAGE:base \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  run: name:  Build Docker images command:  docker-compose -f docker-compose.multi.yml build --build-arg BUILDKIT_INLINE_CACHE=1 -  run: name:  Push base image to docker hub command:  docker push $CACHE_IMAGE:base -  run: name:  Push stage image to docker hub command:  docker push $CACHE_IMAGE:stage` 
```

GitLab CI:

```
`# _config-examples/multi-stage/compose/.gitlab-ci.yml image:  docker/compose:latest services: -  docker:dind variables: DOCKER_DRIVER:  overlay CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 stages: -  build docker-build: stage:  build before_script: -  docker login -u $REGISTRY_USER -p $REGISTRY_PASS script: -  docker build --target base --cache-from $CACHE_IMAGE:base --tag $CACHE_IMAGE:base --file ./Dockerfile.multi --build-arg BUILDKIT_INLINE_CACHE=1 "." -  docker-compose -f docker-compose.multi.yml build --build-arg BUILDKIT_INLINE_CACHE=1 after_script: -  docker push $CACHE_IMAGE:base -  docker push $CACHE_IMAGE:stage` 
```

GitHub 操作:

```
`# _config-examples/multi-stage/compose/github.yml name:  Docker Build on:  [push] env: CACHE_IMAGE:  mjhea0/docker-ci-cache DOCKER_BUILDKIT:  1 COMPOSE_DOCKER_CLI_BUILD:  1 jobs: build: name:  Build Docker Image runs-on:  ubuntu-latest steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Log in to docker hub run:  docker login -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_PASS }} -  name:  Build base from dockerfile run:  | docker build \ --target base \ --cache-from $CACHE_IMAGE:base \ --tag $CACHE_IMAGE:base \ --file ./Dockerfile.multi \ --build-arg BUILDKIT_INLINE_CACHE=1 \ "." -  name:  Build images run:  docker-compose -f docker-compose.multi.yml build --build-arg BUILDKIT_INLINE_CACHE=1 -  name:  Push base image to docker hub run:  docker push $CACHE_IMAGE:base -  name:  Push stage image to docker hub run:  docker push $CACHE_IMAGE:stage` 
```

## 结论

本文概述的缓存策略应该适用于单阶段构建和包含两个或三个阶段的多阶段构建。

添加到构建步骤的每个阶段都需要一个新的构建，并为每个父阶段添加`--cache-from`选项。因此，每个新阶段都会增加更多的混乱，使得 CI 文件越来越难以阅读。幸运的是，BuildKit 支持多阶段构建，Docker 层缓存使用单阶段构建。有关这种高级构建工具包模式的更多信息，请阅读以下文章:

1.  [高级 docker 文件:使用 BuildKit 和多阶段构建实现更快的构建和更小的映像](https://www.docker.com/blog/advanced-dockerfiles-faster-builds-and-smaller-images-using-buildkit-and-multistage-builds/)
2.  [Docker 用 BuildKit 和 buildx 在多主机上构建缓存共享](https://medium.com/titansoft-engineering/docker-build-cache-sharing-on-multi-hosts-with-buildkit-and-buildx-eb8f7005918e)
3.  [利用 Buildkit 的注册表缓存加速 CI/CD 中的多级 Docker 构建](https://dev.to/pst418/speed-up-multi-stage-docker-builds-in-ci-cd-with-buildkit-s-registry-cache-11gi)

最后，需要注意的是，虽然缓存可能会加快您的 CI 构建速度，但您应该不时地在没有缓存的情况下重建您的映像，以便下载最新的操作系统补丁和安全更新。关于这方面的更多信息，请查看[这个线程](https://www.reddit.com/r/docker/comments/bhm9i0/faster_ci_builds_with_docker_cache/)。

--

代码可以在 [docker-ci-cache](https://github.com/testdrivenio/docker-ci-cache) repo 中找到:

1.  [单级示例](https://github.com/testdrivenio/docker-ci-cache/tree/master/_config-examples/single-stage)
2.  [多阶段示例](https://github.com/testdrivenio/docker-ci-cache/tree/master/_config-examples/multi-stage)

干杯！