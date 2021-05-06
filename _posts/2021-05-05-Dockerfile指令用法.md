---
layout:     post
title:      Dockerfile指令用法
subtitle:   介绍dockerfile主要指令
date:       2021-05-06
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - docker
---

Dockfile是一种被Docker程序解释的脚本 Dockerfile由一条一条的指令组成，每条指令对应Linux下面的一条命令 Docker程序将这些Dockerfile指令翻译真正的Linux命令 Dockerfile有自己书写格式和支持的命令，Docker程序解决这些命令间的依赖关系，类似于Makefile Docker程序将读取Dockerfile，根据指令生成定制的image

# Dockerfile的书写规则及指令使用方法

- Dockerfile的指令是忽略大小写的，建议使用大写这样可以与真正需要执行的shell指令区分开，使用#作为注释，每一行只支持一条指令，每条指令可以携带多个参数。
- Dockerfile的指令根据作用可以分为两种，构建指令和设置指令。构建指令用于构建image，其指定的操作不会在运行image的容器上执行；设置指令用于设置image的属性，其指定的操作将在运行image的容器中执行。

## FROM(构建指令)

指定基础image

构建指令，必须指定且需要在Dockerfile其他指令的前面（必须以FROM指令作为第一条非注释指令）。后续的指令都依赖于该指令指定的image。FROM指令指定的基础image可以是官方远程仓库中的，也可以位于本地仓库 

- 指定基础image为该image的最后修改的版本

```
FROM <image>
```

- 指定基础image为该image的一个tag版本

```dockerfile
FROM <image>
```

## MAINTAINER(构建指令)

用来指定镜像创建者信息

用于将image的制作者相关的信息写入到image中。当我们对该image执行docker inspect命令时，输出中有相应的字段记录该信息。 指令格式:

```dockerfile
MAINTAINER <name>
```

## RUN(构建指令)

安装软件用

RUN可以运行任何被基础image支持的命令。如基础image选择了CentOS，那么软件管理部分只能使用CentOS的命令(如yum)。

- RUN命令将在当前image中执行任意合法命令并提交执行结果。命令执行提交后，就会自动执行Dockerfile中的下一个指令。
- 层级 RUN 指令和生成提交是符合Docker核心理念的做法。它允许像版本控制那样，在任意一个点，对image 镜像进行定制化构建。
- RUN 指令缓存不会在下个命令执行时自动失效。比如 RUN apt-get dist-upgrade -y 的缓存就可能被用于下一个指令. --no-cache 标志可以被用于强制取消缓存使用。

```dockerfile
RUN <command> (the command is run in a shell - /bin/sh -c)    # 常用
RUN ["executable", "param1", "param2" ... ] (exec form)
```

## CMD(设置指令)

设置container启动时执行的操作

用于container启动时指定的操作。该操作可以是执行自定义脚本，也可以是执行系统命令。该指令只能在文件中存在一次，如果有多个，则只执行最后一条。 该指令有三种格式：

- 常用

```dockerfile
CMD ["executable","param1","param2"] (like an exec, this is the preferred form)
```

- 几乎不用

```dockerfile
CMD command param1 param2 (as a shell)
```

- 当Dockerfile指定了ENTRYPOINT时使用

```dockerfile
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
```

## ENTRYPOINT(设置指令)

设置container启动时执行的操作

指定容器启动时执行的命令，可以多次设置，但是只有最后一个有效。

- 两种格式

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"] (like an exec, the preferred form)
ENTRYPOINT command param1 param2 (as a shell)
```

- 独自使用： 当独自使用时，如果你还使用了CMD命令且CMD是一个完整的可执行的命令，那么CMD指令和ENTRYPOINT会互相覆盖只有最后一个CMD或者ENTRYPOINT有效。
    
```dockerfile
   # CMD指令将不会被执行，只有ENTRYPOINT指令被执行  
    CMD echo “Hello, World!”  
    ENTRYPOINT ls -l
```

- CMD指令配合使用:使用CMD指令来指定ENTRYPOINT的默认参数，这时CMD指令不是一个完整的可执行命令，仅仅是参数部分；ENTRYPOINT指令只能使用JSON方式指定执行命令，而不能指定参数。

```dockerfile
FROM ubuntu  
CMD ["-l"]  
ENTRYPOINT ["/usr/bin/ls"]
```

## EXPOSE(设置指令)

指定容器需要映射到宿主机器的端口

## ENV(设置指令)

用于设置环境变量

ENV指令可以用于为docker容器设置环境变量,设置了后，后续的RUN命令都可以使用，container启动后，可以通过docker inspect查看这个环境变量。 格式：

```dockerfile
ENV <key> <value>
```

假如你安装了JAVA程序，需要设置JAVA_HOME，那么可以在Dockerfile中这样写：

```dockerfile
ENV JAVA_HOME /path/to/java/dirent
```

## ADD(构建指令)

从src复制文件到container的dest路径(请使用COPY代替ADD，COPY功能简单够用)

所有拷贝到container中的文件和文件夹权限为0755，uid和gid为0；如果是一个目录，那么会将该目录下的所有文件添加到container中，不包括目录；如果文件是可识别的压缩格式，则docker会帮忙解压缩（注意压缩格式）；如果是文件且中不使用斜杠结束，则会将视为文件，的内容会写入；如果是文件且中使用斜杠结束，则会文件拷贝到目录下。

```dockerfile
ADD <src> <dest>
```

- src:是相对被构建的源目录的相对路径，可以是文件或目录的路径，也可以是一个远程的文件url; （copy仅提供本地文件向容器的基本功能）
- dest:是container中的绝对路径

## COPY(构建指令)

复制本地主机的src文件为container的dest

复制本地主机的src文件（为Dockerfile所在目录的相对路径、文件或目录 ）到container的dest。目标路径不存在时，会自动创建。 格式： 当使用本地目录为源目录时，推荐使用COPY

```dockerfile
COPY <src> <dest>
```

## WORKDIR(设置指令)

可以多次切换(相当于cd命令)，对RUN,CMD,ENTRYPOINT生效

```dockerfile
WORKDIR /path/to/workdir
# 在 /p1/p2 下执行 vim a.txt  
WORKDIR /p1      #切换了一次
WORKDIR p2       #又切换了一次
RUN vim a.txt
```

## ARG

设置构建镜像时变量

ARG指令在Docker1.9版本才加入的新指令，ARG 定义的变量只在建立 image 时有效，建立完成后变量就失效消失

```dockerfile
ARG <key>=<value>
```

## LABEL

定义标签

定义一个 image 标签 Owner，并赋值，其值为变量 Name 的值。 格式：

```dockerfile
LABEL Owner=$Name
```

# 如何编写最佳的Dockerfile

Dockerfile的语法非常简单，然而如何加快镜像构建速度，如何减少Docker镜像的大小却不是那么直观，需要积累实践经验。

目标：

- 更快的构建速度
- 更小的Docker镜像大小
- 更少的Docker镜像层
- 充分利用镜像缓存
- 增加Dockerfile可读性
- 让Docker容器使用起来更简单

总结：

- 编写.dockerignore文件
- 容器只运行单个应用
- 将多个RUN指令合并为一个
- 基础镜像的标签不要用latest
- 每个RUN指令后删除多余文件
- 选择合适的基础镜像(alpine版本最好)
- 设置WORKDIR和CMD
- 使用ENTRYPOINT (可选)
- 在entrypoint脚本中使用exec
- COPY与ADD优先使用前者
- 合理调整COPY与RUN的顺序
- 设置默认的环境变量，映射端口和数据卷
- 使用LABEL设置镜像元数据
- 添加HEALTHCHECK

## 通过一个例子

示例Dockerfile犯了几乎所有的错。接下来一步步优化它。假设我们需要使用Docker运行一个Node.js应用，下面就是它的Dockerfile(CMD指令太复杂下面是错误的，仅供参考)。

```dockerfile
FROM ubuntu
ADD . /app
RUN apt-get update  
RUN apt-get upgrade -y  
RUN apt-get install -y nodejs ssh mysql  
RUN cd /app && npm install
# this should start three processes, mysql and ssh
# in the background and node app in foreground
# isn't it beautifully terrible? <3
CMD mysql & sshd & npm start
```

构建镜像

```shell script
docker build -t wtf .
```

### 编写.dockerignore文件

构建镜像时，Docker需要先准备context，将所有需要的文件收集到进程中。默认的context包含Dockerfile目录中的所有文件，但是实际上，我们并不需要.git目录，node_modules目录等内容。 .dockerignore 的作用和语法类似于 .gitignore，可以忽略一些不需要的文件，这样可以有效加快镜像构建时间，同时减少Docker镜像的大小。示例如下:
                                        
```
.git/
node_modules/
```

### 容器只运行单个应用

从技术角度讲，你可以在Docker容器中运行多个进程。你可以将数据库，前端，后端，ssh，supervisor都运行在同一个Docker容器中。但是，这会让你非常痛苦:

- 非常长的构建时间(修改前端之后，整个后端也需要重新构建)
- 非常大的镜像大小
- 多个应用的日志难以处理(不能直接使用stdout，否则多个应用的日志会混合到一起)
- 横向扩展时非常浪费资源(不同的应用需要运行的容器数并不相同)
- 僵尸进程问题 - 你需要选择合适的init进程
因此，我建议大家为每个应用构建单独的Docker镜像，然后使用 Docker Compose 运行多个Docker容器。

现在，我从Dockerfile中删除一些不需要的安装包，另外，SSH可以用docker exec替代。示例如下：

```dockerfile
FROM ubuntu
ADD . /app
RUN apt-get update  
RUN apt-get upgrade -y
# we should remove ssh and mysql, and use
# separate container for database 
RUN apt-get install -y nodejs  # ssh mysql  
RUN cd /app && npm install
CMD npm start
```

### 将多个RUN指令合并为一个

Docker镜像是分层的，下面这些知识点非常重要:

- Dockerfile中的每个指令都会创建一个新的镜像层。
- 镜像层将被缓存和复用
- 当Dockerfile的指令修改了，复制的文件变化了，或者构建镜像时指定的变量不同了，对应的镜像层缓存就会失效
- 某一层的镜像缓存失效之后，它之后的镜像层缓存都会失效
- 镜像层是不可变的，如果我们再某一层中添加一个文件，然后在下一层中删除它，则镜像中依然会包含该文件(只是这个文件在Docker容器中不可见了)。
- Docker镜像类似于洋葱。它们都有很多层。为了修改内层，则需要将外面的层都删掉。记住这一点的话，其他内容就很好理解了。

现在，我们将所有的RUN指令合并为一个。同时把apt-get upgrade删除，因为它会使得镜像构建非常不确定(我们只需要依赖基础镜像的更新就好了)

```dockerfile
FROM ubuntu
ADD . /app
RUN apt-get update \  
    && apt-get install -y nodejs \
    && cd /app \
    && npm install
CMD npm start
```

记住一点，我们只能将变化频率一样的指令合并在一起。将node.js安装与npm模块安装放在一起的话，则每次修改源代码，都需要重新安装node.js，这显然不合适。因此，正确的写法是这样的:

```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y nodejs  
ADD . /app  
RUN cd /app && npm install
CMD npm start
```

### 基础镜像的标签不要用latest

当镜像没有指定标签时，将默认使用latest 标签。因此， FROM ubuntu 指令等同于FROM ubuntu:latest。当时，当镜像更新时，latest标签会指向不同的镜像，这时构建镜像有可能失败。如果你的确需要使用最新版的基础镜像，可以使用latest标签，否则的话，最好指定确定的镜像标签。

示例Dockerfile应该使用16.04作为标签。

```dockerfile
FROM ubuntu:16.04  # it's that easy!
RUN apt-get update && apt-get install -y nodejs  
ADD . /app  
RUN cd /app && npm install
CMD npm start
```

### 每个RUN指令后删除多余文件

假设我们更新了apt-get源，下载，解压并安装了一些软件包，它们都保存在/var/lib/apt/lists/目录中。但是，运行应用时Docker镜像中并不需要这些文件。我们最好将它们删除，因为它会使Docker镜像变大。

示例Dockerfile中，我们可以删除/var/lib/apt/lists/目录中的文件(它们是由apt-get update生成的)

```dockerfile
FROM ubuntu:16.04
RUN apt-get update \  
    && apt-get install -y nodejs \
    # added lines
    && rm -rf /var/lib/apt/lists/*
ADD . /app  
RUN cd /app && npm install
CMD npm start
```

### 选择合适的基础镜像(alpine版本最好)

在示例中，我们选择了ubuntu作为基础镜像。但是我们只需要运行node程序，有必要使用一个通用的基础镜像吗？node镜像应该是更好的选择。

```dockerfile
FROM node
ADD . /app  
# we don't need to install node 
# anymore and use apt-get
RUN cd /app && npm install
CMD npm start
```

更好的选择是alpine版本的node镜像。alpine是一个极小化的Linux发行版，只有4MB，这让它非常适合作为基础镜像。

```dockerfile
FROM node:7-alpine
ADD . /app  
RUN cd /app && npm install
CMD npm start
```

apk是Alpine的包管理工具。它与apt-get有些不同，但是非常容易上手。另外，它还有一些非常有用的特性，比如no-cache和 --virtual选项，它们都可以帮助我们减少镜像的大小。

### 设置WORKDIR和 CMD

WORKDIR指令可以设置默认目录，也就是运行RUN / CMD / ENTRYPOINT指令的地方。

CMD指令可以设置容器创建是执行的默认命令。另外，你应该将命令写在一个数组中，数组中每个元素为命令的每个单词

```dockerfile
FROM node:7-alpine
WORKDIR /app  
ADD . /app  
RUN npm install
CMD ["npm", "start"]
```

### 使用ENTRYPOINT (可选)

ENTRYPOINT指令并不是必须的，因为它会增加复杂度。ENTRYPOINT是一个脚本，它会默认执行，并且将指定的命令错误其参数。它通常用于构建可执行的Docker镜像。entrypoint.sh如下:

```shell script
#!/usr/bin/env sh
# $0 is a script name, 
# $1, $2, $3 etc are passed arguments
# $1 is our command
CMD=$1
case "$CMD" in  
  "dev" )
    npm install
    export NODE_ENV=development
    exec npm run dev
    ;;
  "start" )
    # we can modify files here, using ENV variables passed in 
    # "docker create" command. It can't be done during build process.
    echo "db: $DATABASE_ADDRESS" >> /app/config.yml
    export NODE_ENV=production
    exec npm start
    ;;
   * )
    # Run custom command. Thanks to this line we can still use 
    # "docker run our_image /bin/bash" and it will work
    exec $CMD ${@:2}
    ;;
esac
```

示例dockerfile：

```dockerfile
FROM node:7-alpine
WORKDIR /app  
ADD . /app  
RUN npm install
ENTRYPOINT ["./entrypoint.sh"]  
CMD ["start"]
```

可以使用如下命令运行该镜像:

运行开发版本:

```shell script
docker run our-app dev
```

运行生产版本:

```shell script
docker run our-app start
```

运行bash:

docker run -it our-app /bin/bash

### 在entrypoint脚本中使用exec

在前文的entrypoint脚本中，我使用了exec命令运行node应用。不使用exec的话，我们则不能顺利地关闭容器，因为SIGTERM信号会被bash脚本进程吞没。exec命令启动的进程可以取代脚本进程，因此所有的信号都会正常工作。

### COPY与ADD优先使用前者

COPY指令非常简单，仅用于将文件拷贝到镜像中。ADD相对来讲复杂一些，可以用于下载远程文件以及解压压缩包(参考官方文档)。

```dockerfile
FROM node:7-alpine
WORKDIR /app
COPY . /app  
RUN npm install
ENTRYPOINT ["./entrypoint.sh"]  
CMD ["start"]
```

### 合理调整COPY与RUN的顺序

我们应该把变化最少的部分放在Dockerfile的前面，这样可以充分利用镜像缓存。

示例中，源代码会经常变化，则每次构建镜像时都需要重新安装NPM模块，这显然不是我们希望看到的。因此我们可以先拷贝package.json，然后安装NPM模块，最后才拷贝其余的源代码。这样的话，即使源代码变化，也不需要重新安装NPM模块。

```dockerfile
FROM node:7-alpine
WORKDIR /app
COPY package.json /app  
RUN npm install  
COPY . /app
ENTRYPOINT ["./entrypoint.sh"]  
CMD ["start"]
```

### 设置默认的环境变量，映射端口和数据卷

运行Docker容器时很可能需要一些环境变量。在Dockerfile设置默认的环境变量是一种很好的方式。另外，我们应该在Dockerfile中设置映射端口和数据卷。示例如下:

```dockerfile
FROM node:7-alpine
ENV PROJECT_DIR=/app
WORKDIR $PROJECT_DIR
COPY package.json $PROJECT_DIR  
RUN npm install  
COPY . $PROJECT_DIR
ENV MEDIA_DIR=/media \  
    NODE_ENV=production \
    APP_PORT=3000
VOLUME $MEDIA_DIR  
EXPOSE $APP_PORT
ENTRYPOINT ["./entrypoint.sh"]  
CMD ["start"]
```

ENV指令指定的环境变量在容器中可以使用。如果你只是需要指定构建镜像时的变量，你可以使用ARG指令。

### 使用LABEL设置镜像元数据

使用LABEL指令，可以为镜像设置元数据，例如镜像创建者或者镜像说明。旧版的Dockerfile语法使用MAINTAINER指令指定镜像创建者，但是它已经被弃用了。有时，一些外部程序需要用到镜像的元数据，例如nvidia-docker需要用到com.nvidia.volumes.needed。示例如下:

```dockerfile
FROM node:7-alpine  
LABEL maintainer "jakub.skalecki@example.com" 
 ...
```

### 添加HEALTHCHECK

运行容器时，可以指定--restart always选项。这样的话，容器崩溃时，Docker守护进程(docker daemon)会重启容器。对于需要长时间运行的容器，这个选项非常有用。但是，如果容器的确在运行，但是不可(陷入死循环，配置错误)用怎么办？使用HEALTHCHECK指令可以让Docker周期性的检查容器的健康状况。我们只需要指定一个命令，如果一切正常的话返回0，否则返回1。

```dockerfile
FROM node:7-alpine  
LABEL maintainer "jakub.skalecki@example.com"
ENV PROJECT_DIR=/app  
WORKDIR $PROJECT_DIR
COPY package.json $PROJECT_DIR  
RUN npm install  
COPY . $PROJECT_DIR
ENV MEDIA_DIR=/media \  
    NODE_ENV=production \
    APP_PORT=3000
VOLUME $MEDIA_DIR  
EXPOSE $APP_PORT  
HEALTHCHECK CMD curl --fail http://localhost:$APP_PORT || exit 1
ENTRYPOINT ["./entrypoint.sh"]  
CMD ["start"]
```

当请求失败时，curl --fail 命令返回非0状态。