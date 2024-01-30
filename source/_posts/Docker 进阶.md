---
title: Docker 进阶
tag: Docker
date: 2024-1-30 0:34:00
---

# docker 网络

> docker 提供网络 定义来方便各个 容器进行通信

## Docker 网络类型

Docker 提供了几种默认的网络模式：

### 1. 桥接网络

桥接网络是默认的 Docker 网络模式，容器在这种网络下通过桥接方式与主机通信。

```bash
docker network create <network-name>

docker run --network mynet my-container #为 my-container 添加一个桥接模式的网络mynet

ocker network connect mynet nacos #容器运行时中途添加网络
```

### 2. 主机模式

器共享主机的网络栈，直接使用主机的网络命名空间。容器的网络与主机相同，不进行端口映射。

```bash
docker run --network host my-container
```

### 桥接和主机模式差异

1. 网络隔离：

桥接模式：默认模式，每个容器都有自己的网络栈，容器之间相互隔离。Docker 容器使用网络地址转换 (NAT) 进行通信。
主机模式：容器与主机共享网络栈，没有额外的网络隔离。容器使用主机的网络命名空间，与主机共享网络配置。

2. 端口映射：

桥接模式：Docker 容器可以使用端口映射（Port Mapping）将容器内部的端口映射到主机上，使外部可以访问容器中的服务。
主机模式：容器直接使用主机的端口，不进行端口映射。容器内的服务直接绑定到主机的端口上。

3. 主机名：

桥接模式：Docker 容器有各自的主机名，可以通过容器名或 IP 地址进行通信。
主机模式：容器与主机共享主机名。在主机模式下，容器可以使用主机名直接访问主机上的服务，而无需通过网络。

4. 主机网络性能：

桥接模式：因为涉及额外的网络转换，可能会引入一些性能开销。
主机模式：由于容器直接使用主机网络栈，可能会获得更好的性能，但与此同时，可能会引入一些安全性和隔离性的考虑。

## 网络查看

```
docker network inspect mynet  #查看网络mynet
```

## 应用

在一些跨容器通讯中无需记住 ip 只需要记住容器名就可以通信
如在 A 容器 直接 **ping mysql**

# docker 卷映射

> 卷映射 实现宿主机到容器内部文件的映射，并在容器销毁或迁移数据时提供了一种持久性存储的解决方案

## 字符串自定义卷自动映射

```bash

docker volume inspect <volume-name> #docker 会在/var/lib/docker/volumes/<volume-name>linux默认自动创建卷位置

```

运行时 docker run 指定-v 即可

```bash

docker run -d --name my-container -v my-data-volume:/app/data my-image
#my-data-volume卷用字符串即可表示 /app/data#容器内文件

```

## 自定义绝对位置映射

直接指定绝对位置，更灵活，但迁移性不强

```bash
docker run -d --name my-container -v
/root/docer:/app/data my-image
#/root/docer宿主机绝对位置 /app/data#容器内文件
```

# Docker Compose

> Docker Compose 就可以帮助我们实现多个相互关联的 Docker 容器的快速部署。它允许用户通过一个单独的 docker-compose.yml 模板文件（YAML 格式）来定义一组相关联的应用容器。

| docker run 参数 | docker compose 指令 | 说明       |
| --------------- | ------------------- | ---------- |
| --name          | container_name      | 容器名称   |
| -p              | ports               | 端口映射   |
| -e              | environment         | 环境变量   |
| -v              | volumes             | 数据卷配置 |
| --network       | networks            | 网络       |

### yml 文件示例

```yml
version: "3.8"

services:
  mysql:
    image: mysql
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: 123
    volumes:
      - "./mysql/conf:/etc/mysql/conf.d"
      - "./mysql/data:/var/lib/mysql"
      - "./mysql/init:/docker-entrypoint-initdb.d"
    networks:
      - hm-net
  hmall:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: hmall
    ports:
      - "8080:8080"
    networks:
      - hm-net
    depends_on:
      - mysql
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "18080:18080"
      - "18081:18081"
    volumes:
      - "./nginx/nginx.conf:/etc/nginx/nginx.conf"
      - "./nginx/html:/usr/share/nginx/html"
    depends_on:
      - hmall
    networks:
      - hm-net
networks:
  hm-net:
    name: hmall
```

![image](https://s2.loli.net/2024/01/31/48tomhJTgbkXAif.png)

[docercompose 文档](https://docs.docker.com/compose/reference/)

### 启动与检查

```bash
# 1启动所有, -d 参数是后台启动
docker compose up -d

# 结果：
[+] Building 15.5s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                    0.0s
 => => transferring dockerfile: 358B                                                    0.0s
 => [internal] load .dockerignore                                                       0.0s
 => => transferring context: 2B                                                         0.0s
 => [internal] load metadata for docker.io/library/openjdk:11.0-jre-buster             15.4s
 => [1/3] FROM docker.io/library/openjdk:11.0-jre-buster@sha256:3546a17e6fb4ff4fa681c3  0.0s
 => [internal] load build context                                                       0.0s
 => => transferring context: 98B                                                        0.0s
 => CACHED [2/3] RUN ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo   0.0s
 => CACHED [3/3] COPY hm-service.jar /app.jar                                           0.0s
 => exporting to image                                                                  0.0s
 => => exporting layers                                                                 0.0s
 => => writing image sha256:32eebee16acde22550232f2eb80c69d2ce813ed099640e4cfed2193f71  0.0s
 => => naming to docker.io/library/root-hmall                                           0.0s
[+] Running 4/4
 ✔ Network hmall    Created                                                             0.2s
 ✔ Container mysql  Started                                                             0.5s
 ✔ Container hmall  Started                                                             0.9s
 ✔ Container nginx  Started                                                             1.5s

# 2.查看镜像
docker compose images
# 结果
CONTAINER           REPOSITORY          TAG                 IMAGE ID            SIZE
hmall               root-hmall          latest              32eebee16acd        362MB
mysql               mysql               latest              3218b38490ce        516MB
nginx               nginx               latest              605c77e624dd        141MB

# 3.查看容器
docker compose ps
# 结果
NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS              PORTS
hmall               root-hmall          "java -jar /app.jar"     hmall               54 seconds ago      Up 52 seconds       0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
mysql               mysql               "docker-entrypoint.s…"   mysql               54 seconds ago      Up 53 seconds       0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp
nginx               nginx               "/docker-entrypoint.…"   nginx               54 seconds ago      Up 52 seconds       80/tcp, 0.0.0.0:18080-18081->18080-18081/tcp, :::18080-18081->18080-18081/tcp

```
