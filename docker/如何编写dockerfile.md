### from语法

选择基础镜像

eg: **FROM ubuntu:latest**

#### 选择基础镜像的三个原则

- 官方镜像优于非官方的镜像
- 固定版本的Tag,而不是每次都是用latest
- 功能满足，选择体积小的镜像

<br/>

### Run指令

run指令可以执行Shell指令，包括下载文件、安装软件、配置环境.....都是可以的,但是一个run会叠加一层镜像，所以在使用多个run命令时，尽量使用 && \进行连接，则可只使用一个run命令，只生成一层image layer

<br/>

### Dockerfile的文件操作

制作镜像的时候，经常需要向镜像里添加文件。在Dockerfile中有两个命令可以向镜像中添加文件COPY和ADD。

#### 用copy命令构建镜像

COPY和ADD命令，在复制普通文件的时候，并没有什么太大的不同，两个命令都可以把本地文件，复制到镜像里。

```
FROM node:alpine3.14
COPY index.js  /app/index.js
```

#### 用add命令构建镜像

ADD 构建镜像和COPY最直观的一点不同，是ADD命令可以直接解压gzip压缩文件，这当我们有很多文件要上传操作的时候，就会变的简单很多。

```
FROM node:alpine3.14
ADD index.tar  /app/
```

#### 切换工作目录workdir

在写Dockerfile文件时，默认的操作目录，是镜像的根目录。但有时候需要拷贝很多内容到镜像里是二级目录，就可以使用WORKDIR命令。把工作目录切换到二级，WORKDIR命令像我们操作linux下的cd命令。

```
FROM node:alpine3.14
WORKDIR /app
ADD index.tar  index.js
```

### Dockerfile中的arg和env

ARG 和ENV 是经常容易被混淆的两个Dockerfile语法，它们都可以用来设置一个“变量”。但其实两个语法在细节上有很多不同

#### ENV定义变量

```
FROM ubuntu:latest
ENV VERSION=2.0.1
RUN apt-get update && \
    apt-get install -y wget && \
    wget https://github.com/ipinfo/cli/releases/download/ipinfo-${VERSION}/ipinfo_${VERSION}_linux_amd64.tar.gz && \
    tar zxf ipinfo_${VERSION}_linux_amd64.tar.gz && \
    mv ipinfo_${VERSION}_linux_amd64 /usr/bin/ipinfo && \
    rm -rf ipinfo_${VERSION}_linux_amd64.tar.gz
```

#### ARG定义变量

```
FROM ubuntu:latest
ARG VERSION=2.0.1
RUN apt-get update && \
    apt-get install -y wget && \
    wget https://github.com/ipinfo/cli/releases/download/ipinfo-${VERSION}/ipinfo_${VERSION}_linux_amd64.tar.gz && \
    tar zxf ipinfo_${VERSION}_linux_amd64.tar.gz && \
    mv ipinfo_${VERSION}_linux_amd64 /usr/bin/ipinfo && \
    rm -rf ipinfo_${VERSION}_linux_amd64.tar.gz
```

#### ARG和ENV的不同

总的来说ARG和ENV有两点不同，第一点是声明变量的作用域不同，第二点是ARG声明后，可以在构建时修改变量。

- ARG是构建环境，ENV可带到镜像中
- ARG可以在构建镜像时改变变量值,当镜像创建后，ARG参数将不再生效

### CMD容器启动命令

当设置好基础环境，安装完对应软件，处理完文件后。有时候需要启动某个默认命令。CMD用来设置容器启动时默认会执行的命令。

#### CMD命令的三个基本特性

- 容器启动时默认执行的命令
- 如果docker run 启动容器时指定了其他命令，则CMD命令无效
- 如果定义多个CMD，只有最后一个CMD生效

#### CMD写法

```
CMD ['IPINFO','version']
```

### ENTRYPOINT命令的使用

ENTRYPOINT 命令很容易和 CMD命令混淆。ENTRYPOINT也可以设置容器启动时要执行的命令。

a. 只使用cmd

	同上

b.只使用entrypoint

	启动容器时想要执行的命令会被转换为entrypoint的参数
	
	eg:
	from ubuntu:21.04
	entrypoint ["ls","-a"]
	
	docker build ./ -t test .
	
	docker run test -l
	
	则会执行ls -a -l
	其中-l被追加到ls -a 后面作为参数

c.混合使用

```
FROM ubuntu:21.04
ENTRYPOINT [ "echo"]
CMD []
```

echo作为想要执行的命令，启动容器时传入的会被作为参数

### VOLUME数据持久化

容器删除掉后，里边的数据也会跟着删除。数据的保存和重复可用这是最基本的要求，也就是常说的数据持久化。在写Dockerfile的时候可以用VOLUME命令，指定数据持久化。