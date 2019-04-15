<!--
 * @Description: 
 * @Author: Damon.chen
 * @LastEditors: Damon.chen
 * @Date: 2019-04-04 11:36:10
 * @LastEditTime: 2019-04-15 14:52:55
 -->
# Docker 实战 -- meteor OHIFViewer

> docker采用的是c/s架构，Client通过接口与Server进程通信实现容器的构建，运行和发布。docker比较重要的三个核心概念如下：

>  - 镜像（images）：一个只读的模板，可以理解为应用程序的运行环境，包含了程序运行所依赖的环境和基本配置，镜像可以按照层级（从基础镜像开始）来构建，每一层包含特定的环境。

>  - 仓库（repository）：一个用于存放镜像文件的仓库，如果你对git的仓库熟悉，应该很容易理解，对，就是那个。有私有仓库和公有仓库之分。

>  - 容器（container）：一个运行应用程序的虚拟容器，在我们运行镜像时产生。容器包含自己的文件系统+隔离的进程空间和包含其中的进程。

## 环境准备
- 安装docker，未安装的同学，请根据自己的开发环境采用不同的安装方式去安装，具体操作参考教程，不做赘述。

- 安装成功后，可以通过docker -v查看版本号（尽量使用最新的稳定版本）。

## 项目准备
- 在项目根目录下，添加Dockerfile文件，此文件用来配置我们自定义一个镜像所需要指定的依赖项、环境以及执行的命令等。内容格式如下：
```
# 指定我们的基础镜像是node，版本是v8.0.0
FROM node:8.0.0
# 指定制作我们的镜像的联系人信息（镜像创建者）
MAINTAINER EOI
 
# 将根目录下的文件都copy到container（运行此镜像的容器）文件系统的app文件夹下
ADD . /app/
# cd到app文件夹下
WORKDIR /app
 
# 安装项目依赖包
RUN npm install
RUN npm rebuild node-sass --force
 
# 配置环境变量
ENV HOST 0.0.0.0
ENV PORT 8000
 
# 容器对外暴露的端口号
EXPOSE 8000
 
# 容器启动时执行的命令，类似npm run start
CMD ["npm", "start"]

```
具体本项目配置
```
# First stage of multi-stage build
# Installs Meteor and builds node.js version
# This stage is named 'builder'
# The data for this intermediary image is not included
# in the final image.
FROM node:8.10.0-slim as builder

RUN apt-get update && apt-get install -y \
	curl \
	g++ \
	git \
	python \
	build-essential

RUN curl https://install.meteor.com/ | sh

# Create a non-root user
RUN useradd -ms /bin/bash user
USER user
RUN mkdir /home/user/Viewers
COPY OHIFViewer/package.json /home/user/Viewers/OHIFViewer/
ADD --chown=user:user . /home/user/Viewers

WORKDIR /home/user/Viewers/OHIFViewer

ENV METEOR_PACKAGE_DIRS=../Packages
ENV METEOR_PROFILE=1
RUN meteor npm install
RUN meteor build --directory /home/user/app
WORKDIR /home/user/app/bundle/programs/server
RUN npm install --production

# Second stage of multi-stage build
# Creates a slim production image for the node.js application
FROM node:8.10.0-slim

RUN npm install -g pm2

WORKDIR /app
COPY --from=builder /home/user/app .
COPY dockersupport/app.json .

ENV ROOT_URL http://localhost:3000
ENV PORT 3000
ENV NODE_ENV production

EXPOSE 3000

CMD ["pm2-runtime", "app.json"]

```

- 配置文件中关键字解读
 - FROM
 ```
 语法：FROM <image>[:<tag>]
 解释：设置要制作的镜像基于哪个镜像，FROM指令必须是整个Dockerfile的第一个指令，如果指定的镜像不存在默认会自动从Docker Hub上下载。
 ```
 - MAINTAINER
 ```
 语法：MAINTAINER <name>
 解释：MAINTAINER指令允许你给将要制作的镜像设置作者信息。
 ```
 - ADD
 ```
 语法：ADD <src> <dest>
 解释：ADD指令用于从指定路径拷贝一个文件或目录到容器的指定路径中，<src>是一个文件或目录的路径，也可以是一个url，路径是相对于该Dockerfile文件所在位置的相对路径，<dest>是目标容器的一个绝对路径。
 ```
 - WORKDIR
 ```
 语法：WORKDIR /path/to/workdir
 解释：WORKDIR指令用于设置Dockerfile中的RUN、CMD和ENTRYPOINT指令执行命令的工作目录(默认为/目录)，该指令在Dockerfile文件中可以出现多次，如果使用相对路径则为相对于WORKDIR上一次的值，例如WORKDIR /data，WORKDIR logs，RUN pwd最终输出的当前目录是/data/logs。
 ```
 - RUN
 ```
 语法：① RUN <command>   #将会调用/bin/sh -c <command>
      ② RUN ["executable", "param1", "param2"] #将会调用exec执行，以避免有些时候shell方式执行时的传递参数问题，而且有些基础镜像可能不包含/bin/sh
 解释：RUN指令会在一个新的容器中执行任何命令，然后把执行后的改变提交到当前镜像，提交后的镜像会被用于Dockerfile中定义的下一步操作，RUN中定义的命令会按顺序执行并提交，这正是Docker廉价的提交和可以基于镜像的任何一个历史点创建容器的好处，就像版本控制工具一样。
 ```
 - ENV
 ```
 语法：ENV <key> <value>
 解释：ENV指令用于设置环境变量，在Dockerfile中这些设置的环境变量也会影响到RUN指令，当运行生成的镜像时这些环境变量依然有效，如果需要在运行时更改这些环境变量可以在运行docker run时添加–env <key>=<value>参数来修改。
 注意：最好不要定义那些可能和系统预定义的环境变量冲突的名字，否则可能会产生意想不到的结果。
 ```
 - EXPOSE
 ```
 语法：EXPOSE <port> [ ...]
 解释：EXPOSE指令用来告诉Docker这个容器在运行时会监听哪些端口，Docker在连接不同的容器(使用–link参数)时使用这些信息。
 ```
 - CMD
 ```
 语法： ① CMD ["executable", "param1", "param2"]    #将会调用exec执行，首选方式
      ② CMD ["param1", "param2"]        #当使用ENTRYPOINT指令时，为该指令传递默认参数
      ③ CMD <command> [ <param1>|<param2> ]        #将会调用/bin/sh -c执行
 解释：CMD指令中指定的命令会在镜像运行时执行，在Dockerfile中只能存在一个，如果使用了多个CMD指令，则只有最后一个CMD指令有效。当出现ENTRYPOINT指令时，CMD中定义的内容会作为ENTRYPOINT指令的默认参数，也就是说可以使用CMD指令给ENTRYPOINT传递参数。
 注意：RUN和CMD都是执行命令，他们的差异在于RUN中定义的命令会在执行docker build命令创建镜像时执行，而CMD中定义的命令会在执行docker run命令运行镜像时执行，另外使用第一种语法也就是调用exec执行时，命令必须为绝对路径。
 ```
 - 其中还有其他的一些关键字：USER、ENTRYPOINT、VOLUME、ONBUILD等，如果你有兴趣可以自行研究。
- 在项目根目录下添加.dockerignore文件，此文件的作用类似.gitignore文件，可以忽略掉添加进镜像中的文件，写法、格式和.gitignore一样，一行代表一个忽略。本项目添加的忽略如下：
```
.idea/
.meteor/local
.meteor/meteorite
.meteor/dev_bundle
*/.meteor/dev_bundle
node_modules
.npm
npm-debug.log
Packages/active-entry/helloworld/
LesionTracker/tests/nightwatch/reports/
package-lock.json
docs/_book
docs/
img/
test/
LesionTracker/
StandaloneViewer/
```

## 构建镜像
- 查看目前本地docker的镜像
```
> sudo docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```
- cd 到项目目录，执行以下命令
```
> sudo docker build -t ohifviewer:1.0 .

    Sending build context to Docker daemon  1.436GB
  .... 此处省略1000个字符。
  Successfully built d8f0875e967b
  Successfully tagged ohifviewer:1.0
```
ohifviewer是镜像名，1.0是镜像的版本号，到此你已经成功构建了一个新的镜像，你可以通过sudo docker images，查看你的镜像。

```
> docker images
 REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
 ohifviewer              1.0                 d8f0875e967b        3 minutes ago        2.11GB
```
- 启动镜像，测试是否成功。
```
> docker run -d -p 9000:8000 deploy:1.0
8aec5ee037bb253901d2c2e02c7be546580546c493576139f3789fb660f3401d

> docker ps -a
CONTAINER ID    IMAGE        COMMAND          CREATED           STATUS         PORTS                  NAMES
8aec5ee037bb    deploy:1.0   "npm start"     57 seconds ago    Up 56 seconds  0.0.0.0:9000->8000/tcp amazing_bassi
```
docker run -d -p 9000:8000 deploy:1.0中-d表示后台运行，-p 9000:8000表示指定本地的9000端口隐射到容器内的8000端口。 deploy:1.0为我们要运行的镜像。通过docker ps -a查看docker的进程（容器的运行本身就是一种特殊的进程）运行情况，发现我们的容器已经在运行。本地可以访问localhost:9000。

通过docker logs可以查看我们容器内应用进程的运行日志。docker logs <CONTAINER ID>

```
> docker logs 8aec5ee037bb
  npm info it worked if it ends with ok
  npm info using npm@5.0.0
  npm info using node@v8.0.0
  npm info lifecycle newlook-deploy@1.0.0~prestart: newlook-deploy@1.0.0
  npm info lifecycle newlook-deploy@1.0.0~start: newlook-deploy@1.0.0
  
  > newlook-deploy@1.0.0 start /app
  > node ./server/index.js
  
  Server listening on 0.0.0.0:8000
   DONE  Compiled successfully in 9310ms06:55:56
  
  > Open http://0.0.0.0:8000
```
- 常用命令
```
docker stop <CONTAINER ID>可以停止容器运行

 docker start <CONTAINER ID>可以启动容器运行

 docker restart <CONTAINER ID>可以重启容器

 docker rm <CONTAINER ID> -f可以强制删除在运行的容器
```

## 生产部署

- 在开发环境打包，docker save <namespace>/<name:tag> <name>.tar
```
> docker save lzqs/deploy:1.0 > deploy.tar
```
这里ls会发现目录下生成了deploy.tar的文件。部署时将此文件copy到生产环境服务器上。
- 确保生产服务器上已经安装了docker，若没装，请参考相关文档，若不装，对不起小生也无力了，然后在服务器上加载上传的镜像包deploy.tar。
```
> docker load < deploy.tar

  007ab444b234: Loading layer [==================================================>] 129.3 MB/129.3 MB
  4902b007e6a7: Loading layer [==================================================>] 45.45 MB/45.45 MB
  bb07d0c1008d: Loading layer [==================================================>] 126.8 MB/126.8 MB
  ecf5c2e2468e: Loading layer [==================================================>] 326.6 MB/326.6 MB
  7b3b4fef39c1: Loading layer [==================================================>] 352.3 kB/352.3 kB
  677f02386f07: Loading layer [==================================================>] 137.2 kB/137.2 kB
  7333bb4665b8: Loading layer [==================================================>] 55.66 MB/55.66 MB
  e292e64ffb88: Loading layer [==================================================>] 3.757 MB/3.757 MB
  ee76d0e6f6d9: Loading layer [==================================================>] 1.436 GB/1.436 GB
  33dca533c6e5: Loading layer [==================================================>] 331.8 kB/331.8 kB
  24630015679d: Loading layer [==================================================>] 35.18 MB/35.18 MB
  Loaded image: lzqs/deploy:1.0
```

- 运行lzqs/deploy镜像，成功后，在外部访问服务器的9000端口， <服务器的IP>:9000
```
> docker run -d -p 9000:8000 lzqs/deploy
1d0db9a5d0c8826171e501b0e86afd444fca8144b1105e63dae8d621bdda7a77

> docker ps -a
CONTAINER ID  IMAGE           COMMAND      CREATED              STATUS             PORTS                    NAMES
1d0db9a5d0c8  lzqs/deploy:1.0 "npm start"  About a minute ago   Up About a minute  0.0.0.0:9000->8000/tcp goofy_curran
docker exec -it <CONTAINER ID> /bin/bash 可以进入容器中执行，方便我们查看内部文件和调试

> docker exec -it 1d0db9a5d0c8 /bin/bash

root@1d0db9a5d0c8:/app#
```
战功，访问部署的docker应用，<服务器的IP>:9000，效果如下图：


```
#更新apt-get源 使用163的源
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
    echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >/etc/apt/sources.list && \
    echo "deb http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list
```
