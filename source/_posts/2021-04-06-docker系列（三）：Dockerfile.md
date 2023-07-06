---
title: docker系列（三）：Dockerfile
date: 2021-04-06 17:24:30
categories:
- docker
tags:
- docker
---

> docker系列（二）：Dockerfile

<!--more-->

## 一、Dockerfile是什么
Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明，通过 `docker build` 命令执行 Dockerfile 文件，可以构建一个Docker镜像。

Dockerfile 的文件命名就为 `Dockerfile`，文本格式是一行行指令，每个指令有一个命令和参数组成，类似于命令行可执行文件。例如：
```
# FROM指令指定基础镜像
FROM ubuntu:latest

# COPY指令用于复制文件
COPY /myapp/target/myapp.jar /myapp/myapp.jar

# CMD指令指定容器启动时执行的命令
CMD echo Starting Docker Container
```

一般会把 Dockerfile 放在一个空目录里面，这个目录里面的文件就称为 build 的上下文。移动到 Dockerfile 所在的目录，然后执行
```
$ docker build -t REPOSITORY/IMAGE:TAG .
```
指定自定义镜像名称，生成新的镜像。

## 二、Dockerfile原理

### Dockerfile上下文
在使用 docker build 命令通过 Dockerfile 创建镜像时，会产生一个 `build 上下文(context)`。所谓的 build 上下文就是 docker build 命令的 PATH 或 URL 指定的路径中的文件的集合。`在镜像 build 过程中可以引用上下文中的任何文件，但不能引用上下文以外的文件`。例如：
```
$ docker build .
```
`.`就指定了当前目录为上下文环境。
```
$ docker build -f /path/to/a/Dockerfile .
```
指定了 Dockerfile 文件的位置，但上下文环境还是当前目录。也就是 `Dockerfile 文件和上下文环境可以不在同一个目录`。

对于 COPY 和 ADD 命令来说，如果要把本地的文件拷贝到镜像中，那么本地的文件必须是在上下文目录中的文件。因为在执行 build 命令时，docker 客户端会把上下文中的所有文件发送给 docker daemon。考虑 docker 客户端和 docker daemon 不在同一台机器上的情况，build 命令只能从上下文中获取文件。如果我们在 Dockerfile 的 COPY 和 ADD 命令中引用了上下文中没有的文件，就会收到类似下面的错误：
![image-20230705175422573](image-20230705175422573.png)

`一般情况下 build 上下文应该为一个空文件夹`。因为 docker 客户端会将上下文的全部内容挂载到 docker daemon。如果使用包含很多内容的目录例如根目录 `/` 作为 PATH，会将硬盘驱动器的全部内容全部暴露，增加系统风险。

### Dockerfile缓存
Dockerfile 是由一行行指令组成的，每一行指令都会产生一行`镜像缓存`。如下 Dockerfile 文件
```
FROM busybox:latest
COPY a.txt /usr/a.txt
CMD echo "hello busybox"
```
执行 docker build，输出内容如下：
```
$ sudo docker build -t test_busybox:v1.0 .
[sudo] password for mcb: 
Sending build context to Docker daemon   2.56kB
Step 1/3 : FROM busybox:latest
 ---> 62aedd01bd85
Step 2/3 : COPY a.txt /usr/a.txt
 ---> c0dcd0ba7521
Step 3/3 : CMD echo "hello busybox"
 ---> Running in ee7acd2e64f4
Removing intermediate container ee7acd2e64f4
 ---> 8bf7d58518fb
Successfully built 8bf7d58518fb
Successfully tagged test_busybox:v1.0
```
生成镜像 `test_busybox:v1.0`，Dockerfile 的三条指令分别对应 Step 1-3，生成了三层镜像缓存。如果我们再修改一下 Dockerfile，增加一条指令 `ENV MY_VAR 123`，变成：
```
FROM busybox:latest
COPY a.txt /usr/a.txt
ENV MY_VAR 123
CMD echo "hello busybox"
```
再执行 docker build，输出内容如下：
```
$ sudo docker build -t test_busybox:v2.0 .
Sending build context to Docker daemon   2.56kB
Step 1/4 : FROM busybox:latest
 ---> 62aedd01bd85
Step 2/4 : COPY a.txt /usr/a.txt
 ---> Using cache
 ---> c0dcd0ba7521
Step 3/4 : ENV MY_VAR 123
 ---> Running in 4397c65fbca8
Removing intermediate container 4397c65fbca8
 ---> 3f24a6cb1c21
Step 4/4 : CMD echo "hello busybox"
 ---> Running in ec8f2c77bb95
Removing intermediate container ec8f2c77bb95
 ---> ab7efbf4edfd
Successfully built ab7efbf4edfd
Successfully tagged test_busybox:v2.0
```
生成镜像 `test_busybox:v2.0`，Step 也变成 4 步，注意 Step 2/4 里面，有一句 `Using cache`，这里使用了上一步构建产生的缓存层 `c0dcd0ba7521`。所以每次 docker build 都会尽量去复用已经存在的镜像缓冲层，这样可以节省磁盘空间和提高构建速度。我们也可以指定 `--no-cache` 参数，表示不使用已有的镜像缓存，这样每步 Step 都会生成新的镜像缓存。

如果我们对同一个 Dockerfile 执行多次 docker build，如果文件内容不变，只是修改生成镜像的名称，不会生成多个镜像，只会保留一个镜像，镜像名称是最后执行 docker build 指定的命名。

注意，`这里的镜像缓冲层和 overlay2 里面的 layer 不是同一个概念`。overlay2 的 layer 保存在 `/var/lib/docker/overlay2` 目录，而镜像缓冲层保存 `/var/lib/docker/image/overlay2/imagedb/` 目录。
```
root:/var/lib/docker$ find . -name "c0dcd0ba7521*"
./image/overlay2/imagedb/metadata/sha256/c0dcd0ba752121cf93931167a9e75e5dcc59a0dd5f0e249083ac9ba6856e0679
./image/overlay2/imagedb/content/sha256/c0dcd0ba752121cf93931167a9e75e5dcc59a0dd5f0e249083ac9ba6856e0679
```

### 其他

#### docker build和docker commit的区别

docker build 和 docker commit 都可以生成新的镜像，两者的区别是：
* 使用 docker commit 方式创造的镜像，使用 docker history 看不到构建镜像过程中执行的命令。这意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知，后期极难维护。
* 使用 docker build 方式创造的镜像，可以使用 docker history 清楚看到镜像构建过程中执行了哪些命令；能够自由灵活的与宿主机联系（docker commit因为在容器里面，无法做复制拷贝宿主机文件等动作）；后期可扩展性强，有一个 Dockerfile 文件就可以随时运行镜像了。

例如使用 Dockerfile 生成镜像 test_busybox:v1.0
```
FROM busybox:latest
COPY a.txt /usr/a.txt
CMD echo "hello busybox"
```
使用 docker history 查看镜像，每一行都跟 Dockerfile 一一对应
```
$ docker history test_busybox:v1.0
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
8bf7d58518fb   36 minutes ago   /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "echo…   0B        
c0dcd0ba7521   36 minutes ago   /bin/sh -c #(nop) COPY file:39435c5f19162403…   0B        
62aedd01bd85   13 days ago      /bin/sh -c #(nop)  CMD ["sh"]                   0B        
<missing>      13 days ago      /bin/sh -c #(nop) ADD file:7edb4d0af355de4ed…   1.24MB 
```

#### .dockerignore
`.dockerignore` 文件的作用类似于 git 工程中的 `.gitignore` 。不同的是 `.dockerignore` 应用于 docker 镜像的构建，它存在于 docker 构建上下文的根目录，用来排除不需要上传到 docker 服务端的文件或目录。

docker 在构建镜像时首先从构建上下文找有没有 `.dockerignore` 文件，如果有的话则在上传上下文到 docker 服务端时忽略掉 `.dockerignore` 里面的文件列表。这么做显然带来的好处是：
* 构建镜像时能避免不需要的大文件上传到服务端，从而拖慢构建的速度、网络带宽的消耗；
* 可以避免构建镜像时将一些敏感文件及其他不需要的文件打包到镜像中，从而提高镜像的安全性；

`.dockerignore` 文件的写法和 `.gitignore` 类似，支持正则和通配符，具体规则如下：
* 每行为一个条目；
* 以 `#` 开头的行为注释；
* 空行被忽略；
* 构建上下文路径为所有文件的根路径；

文件匹配规则具体语法如下：
| 规则              | 行为                                                       |
| ----------------- | ---------------------------------------------------------- |
| `*/temp*`         | 匹配根路径下一级目录下所有以 temp 开头的文件或目录         |
| `*/*/temp*`       | 匹配根路径下两级目录下所有以 temp 开头的文件或目录         |
| `temp?`           | 匹配根路径下以 temp 开头，任意一个字符结尾的文件或目录     |
| `**/*.go`         | 匹配所有路径下以 .go 结尾的文件或目录，即递归搜索所有路径  |
| `*.md !README.md` | 匹配根路径下所有以 .md 结尾的文件或目录，但 README.md 除外 |

如果两个匹配语法规则有包含或者重叠关系，那么以后面的匹配规则为准。

## 三、Dockerfile指令

### FROM
指定基础镜像，后续的操作都是基于基础镜像。一般 Dockerfile 文件都是以 FROM 开始，FROM 后面只跟一个基础镜像。
```
FROM ubuntu:latest
```
在17.05版本之前的 Docker，只允许 Dockerfile 中出现一个 FROM 指令；Docker 17.05 版本以后允许 Dockerfile支持多个 FROM 指令，但是仍以最后一条 FROM 为准。

多个 FROM 指令的意义是：每一条 FROM 指令都是一个构建阶段，多条 FROM 就是多阶段构建。虽然最后生成的镜像只能是最后一个阶段的结果，但是能够将前置阶段中的文件拷贝到后边的阶段中。

### ARG
ARG 是唯一一个可以放在 FROM 语句之前的指令。ARG 指定可以定义变量，它的规则是：
* 在 FROM 之前声明的 ARG 在构建阶段之外，所以只可以在 FROM 语句中使用，一般设置默认值。
* 在 FROM 之后声明的 ARG 可以在声明后全局使用。
* 构建 Docker 镜像时，必须为所有 ARG 提供值。如果没有设置默认值，必须在执行 docker build 命令通过 --build-arg 参数传进来；如果设置了默认值，可以通过 --build-arg 参数传入新的值，也可以不传参数使用默认值。

```
ARG VERSION=latest
FROM busybox:$VERSION

ARG VERSION
RUN echo $VERSION > image_version
```
第一个 VERSION 设置了默认值，但只能在 FROM 里面使用。第二个 VERSION 没有默认值，必须通过参数传进来。
```
$ docker build --build-arg VERSION=v1.0 .
```

### MAINTAINER
MAINTAINER命令用于说明谁在维护这个Dockerfile文件。
```
MAINTAINER Joe Blocks <joe@blocks.com>
```
MAINTAINER命令并不常用，因为这类信息在Git存储或其他地方有了。

### USER
USER 可以用来指定 USER 指令声明之后之后 RUN , CMD 和 ENTRYPOINT 运行容器和程序时的用户及用户组（如果不指定用户组，默认为root）
```
USER mcb
```

### RUN
RUN 指令用于 在 Docker 镜像 build 过程中执行命令行指令，运行时会在当前镜像上创建一个新的文件层用来保存修改的数据。RUN 指令有 shell 和 exec 两种格式：
```
RUN  (shell form, the command is run in a shell, which by default is /bin/sh -c on Linux or cmd /S /C on Windows)
RUN ["executable", "param1", "param2"] (exec form)
```
executable是一个可执行的文件，如果是 `/bin` 目录下的文件就不需要指定路径，否则就需要指定完整路径或者先通过 WORKDIR 切换到可执行文件所在的目录。
例如
```
RUN apt-get update -y && apt-get install -y htop
或者
RUN ["apt-get", "update", "-y"]
RUN ["apt-get", "install", "-y", "htop"]
```

### CMD
CMD 命令用于指定启动 Docker 容器时需要执行的命令。也可以在 docker run 启动容器的时候通过命令行指定其他命令，命令行命令会覆盖 Dockerfile 里面的 CMD 命令。

CMD 指令只能出现一次。如果 Dockerfile 中出现多个 CMD 指令，则另外的都会忽略，只有最后一个有效。

CMD 指令也可以使用 shell 或 exec 格式，有三种形式：
```
CMD ["executable","param1","param2"] (exec form, 首选方式)
CMD ["param1","param2"] (作为默认参数传递给ENTRYPOINT)
CMD command param1 param2 (shell form)
```

例如传递默认参数给ENTRYPOINT：
```
FROM busybox
ENTRYPOINT ["/bin/ping"]
CMD ["localhost"]
```
最后在容器启动的时候执行的完整命令为：`/bin/ping localhost`。

### ENTRYPOINT
ENTRYPOINT 指令也可以指定在容器启动时要执行的命令，他和 CMD 指令的区别是：ENTRYPOINT 不会被 docker run 的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序。除非使用 --entrypoint 选项强行覆盖 ENTRYPOINT 指令。

ENTRYPOINT 指令也只能出现一次。Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效。

ENTRYPOINT 也包含 shell 或 exec 两种格式：
```
ENTRYPOINT ["executable", "param1", "param2"] (exec form, 首选方式)
ENTRYPOINT command param1 param2 (shell form)
```

>RUN、CMD 和 ENTRYPOINT 的区别？

* RUN：在容器 build 阶段执行，可以有多个 RUN 命令。
* CMD：在容器 run 阶段启动的时候执行，提供默认命令及参数 (不一定会执行，只是默认) ，有可能会被 docker run 后面命令参数替换。CMD 指令也只能出现一次，如果存在多个 CMD 指令，仅最后一个生效。
* ENTRYPOINT：和 CMD 一样，在容器 run 阶段启动的时候执行，而且一定会执行。ENTRYPOINT 指令也只能出现一次，如果存在多个 ENTRYPOINT 指令，仅最后一个生效。

>shell 和 exec 格式的区别？

* shell 格式：底层会调用 `/bin/sh -c` 来执行命令，可以解析变量。
```
ENV name Docker

# 输出hello Docker
CMD echo "hello $name"

# 上面等价于
CMD ["/bin/sh", "-c", "echo hello $name"]
```
* exec 格式：需要指定使用什么脚本来执行，默认不能解析变量。
```
ENV name Docker

# 这里会输出字符串hello $name，而不是hello Docker，因为$name没有被解析，直接被当作字符串输出了
ENTRYPOINT ["echo", "hello $name"]

# 如果要解析变量，需要指定bash脚本来执行
ENTRYPOINT ["/bin/bash", "-c", "echo hello $name"]
```

### COPY
从上下文目录中复制文件或者目录到容器里指定路径。
例如复制文件：把主机的/myapp/target/myapp.jar文件复制到Docker进行中的/myapp/myapp.jar文件。
```
COPY /myapp/target/myapp.jar /myapp/myapp.jar
```
复制目录：把主机的/myapp/config/prod目录复制到Docker镜像中的/myapp/config目录。
```
COPY /myapp/config/prod /myapp/config
```
复制多个文件到Docker镜像中的一个目录中：将主机的/myapp/config/prod/conf1.cfg文件和/myapp/conig/prod/conf2.cfg文件复制到Docker镜像中的/myapp/config/目录中。注意，目标目录必须以/（斜杠）结束。
```
COPY /myapp/config/prod/conf1.cfg /myapp/config/prod/conf2.cfg /myapp/config/
```

### ADD
类似于 COPY，区别在于：
* 可以添加远程服务器上的文件。这里下载远程地址的 myapp.jar 到 Docker 镜像的 /myapp/ 目录中。如果是添加一个远程的压缩 tar 包，Docker 并不会解压这个文件。
```
ADD http://jenkov.com/myapp.jar /myapp/
```
* 添加的文件如果为打包文件 tar 格式或者压缩文件 gzip、bzip2、xz 格式，会自动复制并解压到。这里会解压 myapp.tar 到 Docker 镜像的 /myapp/ 目录中。
```
ADD myapp.tar /myapp/
```

### ENV
设置环境变量，在后续的指令或者在容器内部，通过 `$` 符号就可以使用这个环境变量。有两种格式
```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```
例如设置变量BASE_URL
```
ENV BASE_URL www.songpangpang.com
RUN curl -SLO "https://$BASE_URL/"
```

### VOLUME
定义匿名数据卷。在启动容器时没有没有使用 `-v` 参数显式挂载数据卷，会自动挂载到匿名卷。挂载的意义是持久化保存在宿主机上，避免容器重启后数据丢失。
```
VOLUME ["/path1", "/path2"]
VOLUME /path
```

### WORKDIR
用于设置 Dockerfile 中的 RUN、CMD 和 ENTRYPOINT 指令执行命令的工作目录（默认为根目录）。WORKDIR 指定的工作目录，必须是提前创建好的。

WORKDIR 可以在一个 Dockerfile 中出现多次，如果使用相对路径，那它会相对于前面一个 WORKDIR 指令对应的目录。例如：
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
最后输出的路径为：`/a/b/c`。

### EXPOSE
暴露端口。
```
EXPOSE <端口1> [<端口2>...]
```

### LABEL
LABEL 可以为镜像添加元数据，元数据以键值对的形式出现。推荐将所有的元数据放在同一条 label 指令。
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
可以通过 `docker inspect` 查看容器定义的 label。

### HEALTHCHECK
定期执行健康检查，监视 Docker 容器中运行的应用程序的运行状况。如果命令返回 0，Docker 将认为应用程序和容器正常；如果命令返回 1，Docker 会认为应用程序和容器不正常。
```
HEALTHCHECK [options] <命令>
```
options说明：
* --interval：健康检查间隔时间，默认为30s
* --start-period：健康检查开始时间。有些应用程序可能需要一段时间启动，因此，只有经过某段时间后再进行健康检查才有意义。默认情况下，Docker会立即检查Docker容器的监控状况。
* --timeout：健康检查最大超时时间，如果健康检查超时，Docker也会认为容器不健康。
* --retries：监控检查最大重试次数。

例如使用了java应用程序的com.jenkov.myapp.HealthCheck作为健康检查的命令，每60s执行一次。
```
HEALTHCHECK --interval=60s java -cp com.jenkov.myapp.HealthCheck
```

### ONBUILD
ONBUILD 是一个特殊的指令，后面跟的是其它指令。ONBUILD 指定的命令，在本次构建镜像的过程中不会执行；只有当以当前镜像为基础镜像（通过  FROM 引用），去构建下一级镜像的时候才会被执行。

例如使用以下 Dockerfile 构建镜像 my-node，三行 ONBUILD 指令不会执行。
```
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```
新的 Dockerfile 以 my-node 作为基础镜像引入时，之前基础镜像的那三行 ONBUILD 就会开始执行。
```
FROM my-node
```

## 四、Dockerfile实战
通过基础镜像 centos:7，在该镜像中安装 jdk 和 tomcat 以后将其制作为一个新的镜像 mycentos:7。
```
# 指明构建的新镜像是来自于 centos:7 基础镜像
FROM centos:7
# 通过镜像标签声明了作者信息
LABEL maintainer="mrhelloworld.com"
# 设置工作目录
WORKDIR /usr/local
# 新镜像构建成功以后创建指定目录
RUN mkdir -p /usr/local/java && mkdir -p /usr/local/tomcat
# 拷贝文件到镜像中并解压
ADD jdk-11.0.6_linux-x64_bin.tar.gz /usr/local/java
ADD apache-tomcat-9.0.37.tar.gz /usr/local/tomcat
# 暴露容器运行时的 8080 监听端口给外部
EXPOSE 8080
# 设置容器内 JAVA_HOME 环境变量
ENV JAVA_HOME /usr/local/java/jdk-11.0.6/
ENV PATH $PATH:$JAVA_HOME/bin
# 启动容器时启动 tomcat 并查看 tomcat 日志信息
ENTRYPOINT /usr/local/tomcat/apache-tomcat-9.0.37/bin/startup.sh && tail -f /usr/local/tomcat/apache-tomcat-9.0.37/logs/catalina.out
```
