### 使用docker搭建wordpress

link

https://docs.docker.com/install/linux/docker-ce/centos/#os-requirements

https://docs.docker.com/compose/install/



#####装系统

1. 首先阿里云建立一个服务器，买了个很便宜的服务器，装上centos7.4 64位



#####装docker

```shell
//首先删除docker
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
  
sudo yum install docker-ce

sudo systemctl start docker

sudo docker run hello-world             
```



#####开始装wordpress，因为wordpress实际上需要一系列的组件来共同工作，所以我们还是先来安装docker-compose

```shell
//docker-compose实际上就是docker的脚本文件，用来同时对多个容器进行启动和停止，非常的实用

sudo curl -L https://github.com/docker/compose/releases/download/1.20.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

//optional install command completition
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.20.1/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose

docker-compose --version
```



#####装完了docker-compose之后就可以开始装wordpress了

##### 安装过程中遇到了一个问题，就是上传文件的大小限制，具体做法是将本地的文件映射到wordpress里面去，这样的话重启 的时候就可以使用最新的设置了。https://github.com/docker-library/wordpress/issues/10

```shell
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - dbdata:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "80:80"
     restart: always
     volumes:
       - /root/my_wordpress/uploads.ini:/usr/local/etc/php/conf.d/uploads.ini
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
    dbdata:
```

上面是docker-compose的创建文件，其中uploads.ini的内容如下：

```shell
file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
```



##### 现在就可以开始进行启动了

启动命令如下：

```shell
//启动wordpress
docker-compose up -d

//关停wordpress
docker-compose  down

//下面的命令是同时清除磁盘，造成wordpress数据丢失
docker-compose  down --v 

```

