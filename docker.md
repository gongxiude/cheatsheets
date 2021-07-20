---
title: Docker CLI
category: Docker
layout: 2017/sheet
---

Docker
-------------

## Basic

### Build image

```bash
docker build -t registry/repo:tag .
```

Create an `image` from a Dockerfile.

### Push image

```bash
docker push -t reigstry/repo:tag
```

### Run container

On daemon

```bash
docker run -d -p LOCAL_PORT:CONTAINER_PORT -v LOCAL_DIR:CONTAINER_DIR -e ENV=somthing registry/repo:tag
```

### Auto completion

Add docker plugin to `~/.oh-my-zsh/plugins/docker/_docker` and check `~/.zshrc`

```bash
plugins=(git 
docker
)
fpath+=($ZSH/plugins/docker)
autoload -U compinit && compinit
```

See [stackoverflow](https://stackoverflow.com/questions/37428133/zsh-docker-plugin-not-working) and [docker docs](https://docs.docker.com/compose/completion/#zsh)

Containers
-----------------

### Interact with container

```bash
docker exec -it CONTAINER_NAME_OR_ID /bin/bash 
```

Run commands in a `container`.

### Automatically remove the container when it exits

```bash
docker run -rm -it registry/repo:tag /bin/bash
```

Images
------

### Manage images

```bash
$ docker images
  REPOSITORY   TAG        ID
  ubuntu       12.10      b750fe78269d
  me/myapp     latest     7b2431a8d968
```

```bash
$ docker images -a   # also show intermediate
```

Manages `image`s.

### Remove container

```bash
docker rmi CONTAINER_ID_OR_NAME
```

Deletes `image`s.

### Export and import image

```bash
docker save imagename > imagename.tar
docker load -i imagename.tar
```

示例
------

### 查找主机上磁盘占用比较多的container

很多时候在kubernetes运行过程中， 存在某个pod因为一些原因造成磁盘的使用率比较高， 我们应该如何找到pod，并进行磁盘的清理呢？ 

因为kubernetes没有收集磁盘相关的数据， 我们只能ssh到宿主机上查找对应的container

使用docker系统df命令，我们可以得到一个docker使用的总结信息，包括以下内容:
- 所有Images的总大小
- 所有Containers的总大小
- 本地卷大小
- 和缓存

```
$ docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              95                  92                  12.29GB             2.514GB (20%)
Containers          188                 184                 54.79GB             0B (0%)
Local Volumes       0                   0                   0B                  0B
Build Cache         0                   0                   0B                  0B
```
通过上文的输出可以看出， 目前Image的存储使用了12G， containers的存储使用了54G， 所以先执行如下命令清理冗余对象：  


默认情况下，如果运行`docker Image`，只能得到每个Image的大小，我们可以运行`docker ps`加上`--size`获得到正在运行容器的大小。
```
$ docker ps --size
315be0878bba        nginx       "/bin/sh -c 'echo \"j…"   24 hours ago        Up 24 hours                             k8s_nginx-df8b4b58d-lx7w6_pre_84f02139-1970-4663-9bb8-dfd613717d80_0                   173MB (virtual 1.07GB)
......................................

```



### 清除所有未使用的镜像、容器、卷和网络 


Docker 提供了一个命令来清理的资源——images, containers, volumes, and networks：
```
$ docker system prune
```

要额外删除任何已停止的容器和所有未使用的images，添加-a参数到命令中

```
$ docker system prune -a
```


### 删除Docker Image

使用`docker images` 机上`-a`参数的命令来定位要删除的Image的ID。 之后将ID传递给`docker rmi`: 

- Show Image

```
docker images -a
```
- Remove Image

```
docker rmi <ImageID> <ImageID>
```

### 删除 dangling images

当build docker 镜像的时候，有时会遇到用一个甚至多个中间层镜像，这会一定程度上减少最终打包出来 docker 镜像的大小，但是会产生一些tag 为 none 的无用镜像，也称为悬挂镜像 (dangling images)

- 列出所有的 dangling images

```bash
docker images -f "dangling=true"
```

- 删除所有的 dangling images

```bash
docker rmi $(docker images -f "dangling=true" -q)
```

### 根据模式删除图像

你可以找到所有使用相匹配的组合模式的图像`docker images`和[`grep`](https://www.digitalocean.com/community/tutorials/using-grep-regular-expressions-to-search-for-text-patterns-in-linux)。满意后，您可以使用`awk`将 ID 传递给 来删除它们`docker rmi`。请注意，这些实用程序不是由 Docker 提供的，也不一定在所有系统上都可用：

- List

```bash
docker images -a |  grep "pattern"
```

- Remove

```bash
docker images -a | grep "pattern" | awk '{print $3}' | xargs docker rmi
```

### 删除所有Image

使用`docker images`添加`-a`可以列出系统上所有的docker镜像， 如果确定要将他们全部删除， 可以使用`-q`参数将ImageID传给`docker rmi`

- List

```bash
docker images -a
```

- Remove

```bash
docker rmi $(docker images -a -q)
```

### 删除Container

使用`docker ps`带有`-a`标志的命令来定位要删除的容器的名称或 ID：

- List

```bash
docker ps -a
```


- Remove

```bash
docker rm <ID_or_Name> <ID_or_Name>
```

### 退出时移除容器

如果要临时创建一个容器， 并且运行完成后不再保留，可以运行`docker run --rm`

```bash
docker run --rm image_name
```

 
### 移除所有exited容器

您可以使用容器定位`docker ps -a`并按其状态过滤它们：已创建、正在重新启动、正在运行、已暂停或已退出。要查看已退出容器的列表，请使用该`-f`标志根据状态进行过滤。当您确认要删除这些容器时，使用`-q`将 ID 传递给`docker rm`命令。

- List

```bash
docker ps -a -f status=exited
```

- Remove

```bash
docker rm $(docker ps -a -f status=exited -q)
```


### 使用多个过滤器删除容器

Docker 过滤器可以通过使用附加值重复过滤器标志来组合。这将生成满足任一条件的容器列表。例如，如果您想删除所有标记为**Created**（使用无效命令运行容器时可能导致的状态）或**Exited**的容器，您可以使用两个过滤器：

- List

```bash
docker ps -a -f status=exited -f status=created
```

- Remove

```bash
docker rm $(docker ps -a -f status=exited -f status=created -q)
```


### 停止并删除所有容器

使用`docker ps` 查看系统上所有运行的容器， 添加`-a`参数显示系统上所有容器(包括exited状态的容器)。 添加`-q`参数后将ID传给`docker stop`和`docker rm`停止不删除所有容器


- List

```bash
docker ps -a
```

- Remove

```bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
```



### 删除一个或多个特定卷 - Docker 1.9 及更高版本

使用该`docker volume ls`命令定位要删除的卷名或名称。然后您可以使用以下`docker volume rm`命令删除一个或多个卷：

- List

```bash
docker volume ls
```

- Remove

```bash
docker volume rm volume_name volume_name
```


### 删除dangling卷 - Docker 1.9 及更高版本

由于卷的点是独立于容器而存在的，所以当一个容器被移除时，不会同时自动移除一个卷。当卷存在并且不再连接到任何容器时，它被称为悬垂卷。要找到它们以确认您要删除它们，您可以使用`docker volume ls`带有过滤器的命令将结果限制为悬空体积。当您对列表感到满意时，您可以使用以下命令将它们全部删除`docker volume prune`：

- List

```bash
docker volume ls -f dangling=true
```

- Remove

```bash
docker volume prune
```



Also see
--------

 * [Getting Started](http://www.docker.io/gettingstarted/) _(docker.io)_
