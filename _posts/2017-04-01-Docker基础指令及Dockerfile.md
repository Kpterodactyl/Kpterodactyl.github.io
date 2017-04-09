---
layout:     post
title:      Docker基础知识
subtitle:   Docker基础指令及Dockerfile
date:       2017-04-01  08:52:08 +0800
author:     Memory
header-img: img/post-bg-js-module.jpg
catalog: Docker
tags:
    - DockerFile
    - php-fpm
    - Centos
---


Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 linux 机器上，也可以实现虚拟化。Docker可以保证在软件开发的时候的环境一致性，以及环境迁移的便捷性。  


- Docker镜像：在运行之后吧、会变成容器，启动Container的速度非常快，docker镜像采用了分层技术，基于一个基础镜像（docker run）  


- Docker Registry：Registry是Docker镜像的中央存储仓库 (docker pull/push)   



基础镜像可以用于创建Docker容器。镜像可以非常基础，仅仅包含操作系统比如CentOS7.1；当你在使用Dockerfile构建镜像的时候，每一个命令都会在前一个命令的基础上形成一个新层。   

## **DockerFile指令：** ##

 
1. From 首先要有基础父镜像如 `FROM centos：centos 7.1.1503`
2. MAINTAINER 维护者的信息
3. ENV 指定一个环境变量，会被后续 RUN 指令使用，并在容器运行时保持，如配置时区 
4. ADD 复制，可以是Dockerfile所在目录的一个相对路径；
如 {% highlight ruby %}ADD supervisord.conf /etc/supervisord.conf {% endhighlight %}   也可以是一个URL；还可以是一个tar文件（自动解压为目录），有解压的功能。
5. RUN 用来安装基础功能如 mkdir pip install yun install   
6. EXPOSE 提供容器暴露端口号，如 EXPOSE 22
7. ENTRYPOINT  容器启动后执行该命令，每个DockerFile只会有最后一条ENTERPOINT生效。 
8. VOLUME 在建立mysql时或需要保持数据时使用，创建一个可以从本地和容器的映射，当容器删除后数据库仍然保留
## **Docker指令：** ##
在写好Dockerfile之后，就可以通过**docker build**开始创建镜像啦

{% highlight ruby %}
docker build -t csphere/centos:7.1 ./path 
{% endhighlight %} 
其中  . 表示当前目录，在执行时会看到会有step 1 2 3 ... 每执行一条都会在镜像中生成一层
![](http://i.imgur.com/dJG0lQr.png)


**docker images** 可以查看当前镜像
**Docker run** 可通过镜像生成容器 


- -it 交互模式 
- -d 在后端启动，返回容器的长ID号
- -p 2222:2222 将容器的22端口映射到主机的2222端口 
- -P 22 将容器的22端口映射到主机随机端口
- --name="base": 为容器指定一个名称
实例：  

![](http://i.imgur.com/5lz2Tg4.png)

**docker ps** 只会显示running状态的容器   
**docker ps -a** 会显示所有
{% highlight ruby %}
docker exec -it website /bin/bash  
{% endhighlight %} 
进入容器指令 ,其中website是容器名称 /bin/bash是选择进入的容器目录
这里我用centos作为基础镜像，之后安装中间件镜像（php-fpm、MySQL）来形成应用镜像。  

创建php-fpm镜像的Dockerfile
{% highlight ruby %}
FROM       csphere/centos:7.1
MAINTAINER KUN

# Set environment variable
ENV	APP_DIR /app

RUN	yum -y --skip-broken install nginx php-cli php-mysql php-pear php-ldap php-mbstring php-soap php-dom php-gd php-xmlrpc php-fpm php-mcrypt && \ 
    yum clean all   
    
RUN sed -i "s/default_server//" /etc/nginx/nginx.conf
RUN echo "daemon=off;" >> /etc/nginx/conf.d/default.conf
ADD	nginx_default.conf /etc/nginx/conf.d/default.conf

ADD	php_www.conf /etc/php-fpm.d/www.conf
RUN	sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/' /etc/php.ini

RUN	mkdir -p /app && echo "<?php phpinfo(); ?>" > ${APP_DIR}/info.php

EXPOSE	80 443

ADD	supervisor_nginx.conf /etc/supervisor.conf.d/nginx.conf
ADD	supervisor_php-fpm.conf /etc/supervisor.conf.d/php-fpm.conf

ONBUILD ADD . /app
ONBUILD RUN chown -R nginx:nginx /app

{% endhighlight %} 

## 可能遇到的问题 ##
Error: fakesystemd conflicts with systemd-219-30.el7_3.7.x86_64 
可以尝试的解决办法
在centos 7 和fakesystend发生了冲突，可能是版本的问题，更新基础镜像
$ sudo docker pull centos:lastest 
或 $ sudo docker pull centos:7 

我根据错误提示添加了--skip-broken 
![](http://i.imgur.com/oGZEolJ.png)
也可以绕过这个问题，不过在镜像和容器创建之后，进入容器之后会报如下错误：
![](http://i.imgur.com/gxHh2Ud.png)
然后在容器中通过 yum -y install php-fpm重新安装后php-fpm就恢复了正常
![](http://i.imgur.com/liJINHN.png)

