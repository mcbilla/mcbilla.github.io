---
title: docker系列（四）：Docker Compose
date: 2021-04-08 19:02:24
categories:
- 中间件
- Docker
tags:
- Docker
---

> docker系列（四）：Docker Compose

<!--more-->

## 一、Docker Compose 基础

### Docker Compose是什么
Docker Compose 是单个宿主机上快速编排多个容器的工具。使用 Docker Compose 不再需要使用 shell 脚本来启动容器。 

Compose 通过一个配置文件来管理多个 Docker 容器，在配置文件中，所有的容器通过 services 来定义，然后使用 docker-compose 脚本来启动，停止和重启应用，和应用中的服务以及所有依赖服务的容器，非常适合组合使用多个容器进行开发的场景。

使用 Docker Compose 的步骤如下：

* 新建空目录作为 Docker Compose 的根目录。
* 定义容器资源，新建 `Dockerfile` 文件定义容器启动需要的资源，并把容器需要的所有资源移动到根目录下。
* 编排服务，新建 `docker-compose.yml` 配置文件，编排服务启动顺序和依赖关系。
* 在根目录下执行 `docker-compose up` 启动所有容器。

`docker-compose.yml` 典型示例如下：

```
# yaml 配置实例
version: '3'
services:
  web:
    build: .
    ports:
   - "5000:5000"
    volumes:
   - .:/code
    - logvolume01:/var/log
    links:
   - redis
  redis:
    image: redis
volumes:
  logvolume01: {}
```

### Docker Compose安装
安装可能需要 root 权限。
```
$ sudo pip install -U docker-compose
```

查看是否安装成功

```
$ docker-compose --version
```

## 二、 Docker Compose 配置文件
Docker Compose 使用 YAML 文件来定义多服务的应用。文件可以命名为：

* `compose.yaml`（官方推荐）
* `compose.yml`
* `docker-compose.yaml`
* `docker-compose.yml`

一个完整的 docker-compose.yml 使用示例如下：

```
services:
  frontend:
    image: awesome/webapp
    ports:
      - "443:8043"
    networks:
      - front-tier
      - back-tier
    configs:
      - httpd-config
    secrets:
      - server-certificate

  backend:
    image: awesome/database
    volumes:
      - db-data:/etc/data
    networks:
      - back-tier

volumes:
  db-data:
    driver: flocker
    driver_opts:
      size: "10GiB"

configs:
  httpd-config:
    external: true

secrets:
  server-certificate:
    external: true

networks:
  # The presence of these objects is sufficient to define them
  front-tier: {}
  back-tier: {}
```

* 启动一个前端服务 webapp 和一个后端服务 database
* 为 webapp 配置 front-tier 网络、back-tier 网络和 sercret 验证
* 为 database 配置持久化卷。
* webapp 和 database 通过 back-tier 网络进行通讯。

整体架构如下所示：

```
(External user) --> 443 [frontend network]
                            |
                  +--------------------+
                  |  frontend service  |...ro...<HTTP configuration>
                  |      "webapp"      |...ro...<server certificate> #secured
                  +--------------------+
                            |
                        [backend network]
                            |
                  +--------------------+
                  |  backend service   |  r+w   ___________________
                  |     "database"     |=======( persistent volume )
                  +--------------------+        \_________________/
```

下面是配置文件常用的一级标签：

* `version` ：版本，已废弃。Compose 文件格式有 3 个版本，分别为 `1`、`2.x` 和 `3.x`。目前主流的为 3.x，支持 docker 1.13.0 以上的版本。官方的态度来说，version 字段即将被废弃。如果不加 version 字段启动失败，可以加个 `version: "3.9"`。
* `services` ：必须字段，定义所有的容器服务。
* `networks`：配置容器连接的网络，用于容器间的通讯。
* `volumes`：挂载一个目录或者一个已存在的数据卷容器。
* `secrets`：存储敏感数据，例如服务密码。

* 2 services, backed by Docker images: webapp and database
* 1 secret (HTTPS certificate), injected into the frontend
* 1 configuration (HTTP), injected into the frontend
* 1 persistent volume, attached to the backend
* 2 networks
### services
services 标签是使用最频繁的字段，而且是必须字段。用来定义所有的容器服务

#### build
指定构建镜像的上下文路径。该路径可以使用绝对路径，也可以使用相对路径（根目录为 Compose 文件所在目录），该路径里面必须包含 Dockerfile 文件。
```
services:
  webapp:
    build: ./dir
```
build 字段也可以使用子字段进行更加详细的配置：
```
services:
  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
      labels:
        - "com.example.description=Accounting webapp"
        - "com.example.department=Finance"
        - "com.example.label-with-empty-value"
      target: prod
```
* context：必须字段，上下文路径。
* dockerfile：指定构建镜像的 Dockerfile 文件名。
* args：添加构建参数，这是只能在构建过程中访问的环境变量。
* labels：设置构建镜像的标签。
* target：多层构建，可以指定构建哪一层。

#### image
指定容器运行的镜像。以下格式都可以：
```
image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd # 镜像id
```

#### container_name
指定一个自定义容器名称。如果不指定，容器名称默认以 Compose 文件所在目录名称为前缀。
```
container_name: my-web-container
```
由于Docker容器名称必须是唯一的，因此如果指定了自定义名称，则无法将服务扩展到多个容器。

#### volumes
卷挂载路径设置。设置格式有：
* `宿主机路径:容器路径`
* `宿主机路径:容器路径:访问模式`
例如：
```
volumes:
  # 只需指定一个路径，让引擎创建一个卷
  - /var/lib/mysql
 
  # 指定绝对路径映射
  - /opt/data:/var/lib/mysql
 
  # 相对于当前compose文件的相对路径
  - ./cache:/tmp/cache
 
  # 用户家目录相对路径，而且使用只读模式
  - ~/configs:/etc/configs/:ro
 
  # 命名卷
  - datavolume:/var/lib/mysql
```

#### command
覆盖容器启动后默认执行的命令。
```
command: bundle exec thin -p 3000
command: ["bundle", "exec", "thin", "-p", "3000"]
```

#### entrypoint
覆盖容器默认的 entrypoint。
```
entrypoint: /code/entrypoint.sh
```
也可以是以下格式：
```
entrypoint:
    - php
    - -d
    - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
    - -d
    - memory_limit=-1
    - vendor/bin/phpunit
```

#### depends_on
设置依赖关系。
* docker-compose up ：以依赖性顺序启动服务。在以下示例中，先启动 db 和 redis ，才会启动 web。
* docker-compose up SERVICE ：自动包含 SERVICE 的依赖项。在以下示例中，docker-compose up web 还将创建并启动 db 和 redis。
* docker-compose stop ：按依赖关系顺序停止服务。在以下示例中，web 在 db 和 redis 之前停止。
```
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```
注意：web 服务不会等待 redis db 完全启动 之后才启动。

#### environment
添加环境变量。您可以使用数组或字典、任何布尔值，布尔值需要用引号引起来，以确保 YML 解析器不会将其转换为 True 或 False。
```
environment:
  RACK_ENV: development
  SHOW: 'true'
```

#### ports
映射端口到宿主机上，格式为 `宿主机端口:容器端口` 或者仅仅指定容器的端口（宿主将会随机选择端口）。
```
ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
 - "6060:6060/udp"
```
当使用 HOST:CONTAINER 格式来映射端口时，如果你使用的容器端口小于 60 你可能会得到错误得结果，因为 YAML 将会解析 xx:yy 这种数字格式为 60 进制。所以建议采用字符串格式。

#### expose
暴露端口，但不映射到宿主机，只被连接的服务访问。 
```
expose:
 - "3000"
 - "8000"
```

#### healthcheck
用于检测 docker 服务是否健康运行。
```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"] # 设置检测程序
  interval: 1m30s # 设置检测间隔
  timeout: 10s # 设置检测超时时间
  retries: 3 # 设置重试次数
  start_period: 40s # 启动后，多少秒开始启动检测程序
```

#### network_mode
设置网络模式。
```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

#### restart
* no：是默认的重启策略，在任何情况下都不会重启容器。
* always：容器总是重新启动。
* on-failure：在容器非正常退出时（退出状态非0），才会重启容器。
* unless-stopped：在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器
```
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```

### volumes

挂载一个目录或者一个已存在的数据卷容器，作用是：

* 重用数据卷。可以将主机路径挂载为单个服务定义的一部分，无需在一级`volumes`键中定义它。但是想在多个服务中重用一个卷，则需要在一级volumes 中定义一个命名卷。

* 保护宿主机文件。可以直接使用`HOST:CONTAINER`这样的格式，或者使用`HOST:CONTAINER:ro`这样的格式，后者对于容器来说，数据卷是只读的，这样可以有效保护宿主机的文件系统。

如下实例，web 服务使用命名卷 (`mydata`)，以及为单个服务定义的绑定安装（`db`service下的第一个路径`volumes`）。`db`服务还使用名为`dbdata`（`db`service下的第二个路径`volumes`）的命名卷，使用了旧字符串格式定义它以安装命名卷。命名卷必须列在顶级`volumes`键下。

```
version: "3.9"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"

volumes:
  mydata:
  dbdata:
```

#### 简短语法

简短语法使用通用`[SOURCE:]TARGET[:MODE]`格式，其中 `SOURCE`可以是主机路径或卷名。`TARGET`是安装卷的容器路径。标准模式`ro`用于只读和`rw`读写（默认）。

您可以在主机上挂载一个相对路径，该路径相对于正在使用的 Compose 配置文件的目录展开。相对路径应始终以`.`或开头`..`。

```
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # 命名卷
  - datavolume:/var/lib/mysql
```

#### 长语法

- `type`: 安装类型， `bind`,`tmpfs`或`npipe`
- `source`: 安装源、主机上用于绑定安装的路径或在顶级volumes 中定义的卷的名称 。不适用于 tmpfs 挂载。
- `target`：安装卷的容器中的路径
- `read_only`: 将卷设置为只读的标志
- `bind`: 配置额外的绑定选项

```
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

networks:
  webnet:

volumes:
  mydata:
```

### networks

默认情况下，Compose 会创建一个默认网络，`services` 服务的所有容器都加入默认网络，并且可以被该网络上的其他容器访问。我们可以创建通过 `networks` 自定义更复杂的网络，在相同网络下的容器可以进行通讯。例如

```
version: "3"
services:

  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    # Use a custom driver
    driver: custom-driver-1
  backend:
    # Use a custom driver which takes special options
    driver: custom-driver-2
    driver_opts:
      foo: "1"
      bar: "2"
```

一级配置`networks` 创建了自定义的网络 。这里配置了两个`frontend`和`backend` ，且自定义了网络类型。每一个services下，`proxy` , `app` , `db`都定义了`networks`配置。

1. `proxy` 只加入到 `frontend`网络。
2. `db` 只加入到`backend`网络。
3. `app`同时加入到 `frontend`和`backend` 。
4. `db`和`proxy`不能通讯，因为不在一个网络中。
5. `app`和两个都能通讯，因为`app`在两个网络中都有配置。
6. `db`和`proxy`要通讯，只能通过`app`这个应用来连接。

### secrets

#### 简短语法

简短的语法仅指定机密名称。以下示例授予`redis`服务对`my_secret`和`my_other_secret`机密的访问权限。`./my_secret.txt`文件的内容被设置为 my_secret，`my_other_secret`被定义为外部机密，这意味着它已经在Docker中定义，无论是通过运行`docker secret create`命令还是通过另一个堆栈部署，都不会重新创建。如果外部机密不存在，堆栈部署将失败并显示`secret not found`错误。

```
version: "3.9"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    secrets:
      - my_secret
      - my_other_secret
secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

#### 长语法

- `source`：定义机密标识符。
- `target`：要挂载在`/run/secrets/`服务的任务容器中的文件的名称，默认是 source。
- `uid`和`gid`：`/run/secrets/`在服务的任务容器中拥有文件的数字 UID 或 GID 。
- `mode`：要挂载在`/run/secrets/` 服务的任务容器中的文件的权限，以八进制表示法。例如，`0444` 代表可读。

下面的示例表示将my_secret 命名为`redis_secret`，模式为`0440`（组可读），和用户组为`103`。该`redis`服务无权访问该`my_other_secret`机密。

```
version: "3.9"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    secrets:
      - source: my_secret
        target: redis_secret
        uid: '103'
        gid: '103'
        mode: 0440
secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

## 三、Docker Compose 命令

docker-compose 命令需要在 Compose 文件所在目录下执行的，基本格式：
```
$ docker-compose [options] [COMMAND] [ARGS...]
```
options说明
* -f, --file FILE 指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定。
* -p, --project-name NAME 指定项目名称，默认将使用所在目录名称作为项目名。
* --verbose 输出更多调试信息。
* -v, --version 打印版本并退出。

详细参考 [这里](https://yeasy.gitbook.io/docker_practice/compose/commands)

### 启动服务
```
$ docker-compose up [options] [SERVICE...]
```
options说明：
* -d 在后台运行服务容器。
* --no-color 不使用颜色来区分不同的服务的控制台输出。
* --no-deps 不启动服务所链接的容器。
* --force-recreate 强制重新创建容器，不能与 --no-recreate 同时使用。
* --no-recreate 如果容器已经存在了，则不重新创建，不能与 --force-recreate 同时使用。
* --no-build 不自动构建缺失的服务镜像。
* -t, --timeout TIMEOUT 停止容器时候的超时（默认为 10 秒）。

该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。链接的服务都将会被自动启动，除非已经处于运行状态。可以说，大部分时候都可以直接通过该命令来启动一个项目。

常用的有两种格式：
* `docker-compose up`：默认情况，启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很方便进行调试。当通过 Ctrl-C 停止命令时，所有容器将会停止。
* `docker-compose up -d`，后台启动并运行所有的容器。一般推荐生产环境下使用该选项。

默认情况，如果服务容器已经存在，docker-compose up 将会尝试停止容器，然后重新创建（保持使用 volumes-from 挂载的卷），以保证新启动的服务匹配 docker-compose.yml 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 docker-compose up --no-recreate。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 docker-compose up --no-deps -d <SERVICE_NAME> 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务。

### 列出所有服务
```
$ docker-compose ps
```

### 重新构建服务
```
$ docker-compose build
```

### 停止服务
```
$ docker-compose down
```

### 查看服务日志
```
$ docker-compose logs
```

### 启动已存在的服务
```
$ docker-compose start
```

### 停止已运行的服务
```
$ docker-compose stop
```

### 重启服务
```
$ docker-compose restart
```



## 感谢

[docker compose 配置文件 .yml 全面指南](https://zhuanlan.zhihu.com/p/387840381)
