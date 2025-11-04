---
title: Makefile 以及 Dockfile 详解及用法.
tags: [Makefile,Dockfile]
date: 2025-11-05 02:24:00
---

# Makefile 以及 Dockfile 详解及用法

虽然两者实际上没有什么关联 但名字挺像就拿来一起总结呢\~\~\~\~

## &#x20;Makefile是什么？



[Make](https://en.wikipedia.org/wiki/Make_%28software%29)是最常用的构建工具，诞生于1977年，主要用于C语言的项目。实际上对**文件或者linux上任何操作** 都可以进行 构建 (build)

通常我们可以在Makefile 文件或者makefile 构建 make 具体的方法



Makefile文件由一系列规则（rules）构成。每条规则的形式如下。

```Makefile
<target> : <prerequisites> 
[tab]  <commands>
```

target 是目标（也可以是某个文件也可以是个伪目标<非文件>)，这个必须有，\<prerequisites> 前置条件（通常是文件）可有可无参数，但一旦存在就必须存在该文件

commands 命令细节 ，可有可无

这里主要讲 make 的两个主要 功能&#x20;

文件生成 构造(make)

```Makefile
#Makefile 定义
hello: hello.c
	gcc hello.c -o hello
```

以最初的c语言编译为例，当执行make 命令的时候，会将hello.c源文件 预处理编译汇编最后生成目标文件hello；如果此时再次调用make呢？\
1.若源文件hello.c 没有改变则会返回

```Makefile
make: 'hello' is up to date.
```

2.若源文件hello.c 发生变化 则会新生成hello覆盖原来的

所以make可以用来构建一组编译方法自动判断哪些文件需要重新编译、链接或生成

另外，还有以下例子，用来描述了如何 make c.txt

```Makefile
c.txt: a.txt b.txt
    cat b.txt a.txt > a.txt
```

除了构建文件，还能构建工具如下 调用make clean，make test，make deploy 即可执行对应目标命令，可以看到每个规则都是只有目标和命令没有前提条件，且目标也不是文件，理论上没有.PHONY: clean test deploy 也能够正常执行，但是由于之前说到的目标会被make检查是否存在，所以假设目录下有同名的clean、test、deploy文件那将无法进行make执行命令，由于没有前置条件也不会发生修改所以make就会一直make: 'xxx' is up to date.但是引入.PHONY: clean test deploy 就不一样了，这里定义了clean test deploy 为伪目标（非文件）其实就是个标识,之后再使用make就不去检查文件是否存在了

```Makefile
.PHONY: clean test deploy

clean:
	rm -rf build/

test:
	pytest

deploy:
	rsync -avz ./build/ user@server:/var/www/

```

运用make可以自定义任何 以“下一键启动命令”

*   自动生成文档；
*   打包/发布项目；
*   部署服务器；
*   清理中间文件；
*   执行测试任务；
*   构建前端资源（配合 npm、webpack）；

最后介绍一个小技巧

```Makefile
source: file1 file2 file3
```

source 是一个伪目标，只有三个前置文件，没有任何对应的命令。make source会直接创建3个文件file1 file2 file3
[可以参考阮一峰博客了解更多细节](https://www.ruanyifeng.com/blog/2015/02/make.html)，个人感觉是一篇介绍makefile比较清晰的博客
## Dockerfile
Dockerfile 就是用来自定义镜像的，用于docker 进行构建images并推送到远程

### 基础镜像（FROM）
每个 Dockerfile 的第一条指令必须是 FROM，用于指定构建新镜像所基于的基础镜像。例如：


```dockerfile
FROM ubuntu:22.04
```
### 工作目录（WORKDIR）
进入容器内部时默认的目录就是工作目录
设置容器内的工作目录，后续的 COPY、RUN 等指令都会在这个目录下执行。例如：
```dockerfile
WORKDIR /app
```

可以在 Dockerfile 中多次使用 WORKDIR指令。每次使用都会更改当前的工作目录
如果指定的目录不存在，WORKDIR 会自动创建该目录

复制文件（COPY或ADD）
如果想复制多个文件到镜像中，可以使用多个 COPY 指令将本地文件或目录复制到镜像中。例如：

```dockerfile
# 复制当前目录的所有文件到镜像的 /app 目录
COPY . /app/

# 复制单个文件到镜像的 /app 目录
COPY package.json /app/
```

基本语法：

`COPY <src> <dest>`

功能：仅用于将文件或目录从构建上下文复制到镜像中

`特点：
简单直观：COPY 的行为非常明确，只执行复制操作
不支持 URL：不能用于从远程 URL 下载文件
不支持自动解压缩：不能自动解压缩压缩文件（如 .tar, .zip 等）`

除了复制文件或目录外，还支持从远程 URL 下载文件以及自动解压缩压缩文件

```dockerfile
# 从远程 URL 下载文件并复制到镜像中
ADD https://example.com/file.zip /app/

# 自动解压缩压缩文件
ADD file.tar.gz /app/

# 复制并自动解压缩
ADD source.tar.gz /app/
```
在编写Dockerfile时，推荐优先使用COPY而不是ADD，原因有几个：
COPY命令的功能非常明确，就是将文件从本地复制到Docker镜像中。而ADD命令除了复制文件外，还可以执行URL下载和解压操作，这可能会引入不必要的复杂性
ADD命令如果用于复制本地tar文件，会自动解压文件。这种行为可能会导致构建过程变得不那么可预测，尤其是当你不期望文件被解压时。而COPY命令总是按原样复制文件，不会进行任何额外的处理
由于ADD可以处理URL，如果URL中包含特殊字符，可能会造成构建失败。此外，如果URL指向的文件是tar文件，它会被自动解压，这可能会导致不可预见的结果
使用COPY可以使得Dockerfile更加清晰，对于其他开发者来说更容易理解。它减少了构建上下文的歧义，使得构建过程更易于维护
许多Docker最佳实践指南推荐使用COPY，因为它遵循了最小惊讶原则（Principle of Least Astonishment），即系统行为应该尽可能符合用户的预期
因此，除非你需要从URL下载文件或者自动解压tar文件，否则应该优先使用COPY。这样可以使 Dockerfile 更加简洁、明确和可靠

### 运行命令（RUN）
在镜像构建过程中执行命令，用于安装软件包或配置环境。例如：
```dockerfile
RUN apt-get update && apt-get install -y python3
```
### 设置环境变量（ENV）
定义环境变量，可以在后续的指令中使用。例如：

```dockerfile
ENV NAME World
RUN echo "Hello $NAME"
```
或者构建好了使用变量

```bash
docker run --rm myimage sh -c 'echo Hello $NAME'
```

### 暴露端口（EXPOSE）
如果想放行多个端口，可以使用多个EXPOSE指令声明容器运行时监听的端口。例如：

```dockerfile 
EXPOSE 80
```

### 容器启动时默认执行的命令（CMD或ENTRYPOINT）

| 指令           | 作用               | 是否可被覆盖                         | 常见用途       |
| ------------ | ---------------- | ------------------------------ | ---------- |
| `CMD`        | 定义容器启动时默认要运行的命令  | ✅ 可以被 `docker run` 后的命令覆盖      | 适合提供“默认行为” |
| `ENTRYPOINT` | 定义容器启动时固定要运行的主命令 | ⚠️ 一般不会被覆盖（除非加 `--entrypoint`） | 适合作为“执行入口” |

如下

```dockerfile
FROM alpine
CMD ["echo", "Hello World"]
```
运行docker run myimage 输出 Hello World;若是docker run myimage echo Hi 输出 Hi，进行了覆盖


```
FROM alpine
ENTRYPOINT ["echo", "Hello"]
```
运行 docker run myimage World 输出 Hello World；运行 docker run myimage echo hi 输出  Hello echo hi；所以这里只是传递参数不过会覆盖
修改只能这样操作

```bash
docker run --entrypoint ls myimage /
```
通常我们会采用 ENTRYPOINT + CMD 结合使用

```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```
另外无论是dockerfile里定义了多少ENTRYPOINT 和 CMD 都只会取最后一个

举一个完整的例子

```dockerfile
# 使用 ubuntu 22.04 作为基础镜像
FROM ubuntu:22.04

# 设置环境变量，防止交互式提示
ENV DEBIAN_FRONTEND=noninteractive

# 更新系统包列表
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /app

# 使用 COPY 指令复制本地 Python 脚本到容器的工作目录
COPY hello.py /app/

# 设置容器启动命令，运行 Python 脚本
CMD ["python3", "hello.py"]

```

最后编写完dockerfile 可以docker build 进行构建

```bash
docker build --tag ubuntu-python:1.0.0 ./
```

dockfile和makefile 看似没有关系，其实都是一种构建，其实我们也可以定义docker build 的make（一种有强迫症的封装)
```Makefile
.PHONY: dockerbuildimg

dockerbuildimg:
	docker build --tag ubuntu-python:1.0.0 ./

```









