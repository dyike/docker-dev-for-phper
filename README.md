## 背景：

环境部署是所有团队都必须面对的问题，随着系统越来越大，依赖的服务也可能越来越多。有的时候新人入职配置环境一天就过去了，过程很坎坷！！

一般项目会用到下面的：

* Web服务器：Nginx
* Web程序：PHP
* 数据库：MySQL
* 缓存服务：Redis + Memcache
* 前端构建工具：npm + bower + gulp 等等
* PHP CLI工具：Composer + PHPUnit

因此团队的开发环境部署随之暴露出若干问题：

1.	依赖服务很多，本地搭建一套环境成本越来越高，初级人员很难解决环境部署中的一些问题
2.	服务的版本差异及OS的差异都可能导致线上环境BUG
3.	项目引入新的服务时所有人的环境需要重新配置

对于问题1，好解决，就是vagrant基于虚拟机的方案,整个团队共享一套开发环境镜像。
对于问题2，可以引入PHPBrew这样的版本PHP管理工具来解决。
对于问题3，让我想想，让我想想，让我想想。好像这两者都不能很好地解决，虚拟机镜像没有版本管理，总之不方便。。。

不妨，试试Docker。坑总是要挖，生产环境我们不跑docker，但是开发及测试还是可以考虑的。

直接来吧！

--------

## MySQL

### 获取mysql镜像，直接上官方的镜像， 输入命令：

```
docker pull mysql:5.7
```

### 运行MySQL

```
docker run -p 3306:3306 --name dev_mysql \
-v $PWD/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d \
--privileged=true mysql:5.7
```
### 命令说明：

* -p 3306:3306 （讲容器的3306端口映射到主机3306端口）
* --name (给容器指定别名)
* -v $PWD/mysql/data:/var/lib/mysql (将主机当前目录下的mysql/data文件夹挂载到容器的/var/lib/mysql 下，也就是说在mysql容器中产生的数据就会保存在本机mysql/data目录下）
* -e MYSQL_ROOT_PASSWORD=123456 （初始化root用户的密码）
* -d （后台运行容器）
* --privileged=true （可能会碰到权限问题，需要加参数）

### 执行以下命令进入mysql 运行环境

```
docker exec -it test_mysql bash
```
这样就进入mysql容器了，可以查看mysql 命令

```
mysql －u root -p
```

## PHP镜像，需要支持mysql扩展。用Dockerfile构建一个镜像。

 ```
 FROM  php:7.1-fpm
 RUN apt-get update && apt-get install -y \
 libfreetype6-dev \
 libjpeg62-turbo-dev \
 libpng12-dev \
 vim \
 && docker-php-ext-install pdo_mysql \
 && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
 && docker-php-ext-install gd \
 ```

### build镜像

```
docker build -t="php-fpm7.1" .
```

### 运行php环境的容器

```
docker run -d -p 9000:9000 \
-v /Users/ityike/Code:/var/www/html \
--name php-and-mysql \
--link dev_mysql:mysql \
--volumes-from dev_mysql \
--privileged=true \
php-fpm7.1
```

### 命令说明

* －v （将本地磁盘上的php代码挂载到docker 环境中，对应docker的目录是 /var/www/）
* --name （新建的容器名称 php-and-mysql）
* --link （链接的容器，链接的容器名称：在该容器中的别名，运行这个容器是，docker中会自动添加一个host识别被链接的容器ip）
* --volumes-from （来执行docker run，但需要注意的是不管是dev_mysql是否运行，都会起作用。只要有容器连接volume，就不会被删除）
* --privileged=true （权限问题）

### 进入容器：

```
docker exec -it php-and-mysql bash
cd /var/www && ls
```
将会看到本地机器上/User/ityike/Code/ 下的代码。


## 安装Nginx

在本地目录编辑nginx的配置文件 default.conf
绝对路径为/Users/ityike/Docker/nginx/conf/default.conf

```
server {
    listen       80;
    server_name  localhost;
    root /var/www/html;    #代码目录

    location / {
        root   /var/www/html;
        index  index.html index.htm index.php;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/html;
    }
    location ~ \.php$ {
        fastcgi_pass   phpfpm:9000;    # 修改为phpfpm容器
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name; # 修改为$document_root
        include        fastcgi_params;
    }
}
```

### 运行容器

```
docker run -d \
--link php-and-mysql:phpfpm \
--volumes-from php-and-mysql \
-p 80:80 \
-v /Users/ityike/Docker/nginx/conf/default.conf:/etc/nginx/conf.d/default.conf \
--name nginx-php \
--privileged=true nginx
```

### 命令说明：

* --link php-and-mysql:phpfpm （将php容器链接到nginx容器里来，phpfpm是nginx容器里的别名）
* --volumes-from php-and-mysql (将php-and-mysql 容器挂载的文件也同样挂载到nginx容器中)
* -v /Users/ityike/Docker/nginx/conf/default.conf:/etc/nginx/conf.d/default.conf \
--name nginx-php (将nginx 的配置文件替换，挂载本地编写的配置文件)
* --name nginx-php   (容器名）

### 进入容器

```
docker exec -it nginx-php bash
```

至此 nginx+php+mysql的连接基本完成

----------

如果还有redis服务器，我再搞一个容器，然后link过去，敲那么多命令，真的就没有一个好的解决方案吗？
当然不是了，主角docker-compose上场了。

## Docker-compose

使用Docker-compose 不再需要shell脚本来启动容器。在配置文件中，所有的容器通过services来定义，使用docker-compose脚本来启动，停止和重启。

### 安装docker-compose

```
pip install -U docker-compose
```

### 在一个目录文件夹加创建Compose的配置文件：docker-compose.yml

```
nginx:
  build: ./nginx
  ports:
    - "80:80"
  links:
    - "phpfpm"
  volumes:
    - /Users/ityike/Docker/code/:/var/www/html/
    - /Users/ityike/Docker/nginx/conf/default.conf:/etc/nginx/conf.d/default.conf

phpfpm:
  build: ./phpfpm
  ports:
    - "9000:9000"
  volumes:
    - ./code/:/var/www/html/
  links:
    - "mysql"
mysql:
  build: ./mysql
  ports:
    - "3306:3306"
  volumes:
    - /Users/ityike/Docker/mysql/data/:/var/lib/mysql/
  environment:
    MYSQL_ROOT_PASSWORD : 123456
```
这里没有添加redis，可以自己考虑动手试试。


### 查看上面说的Docker文件夹目录结构： tree Docker/
(忽略了一些文件，这里只列出主要的)
```
Docker
├── code
│   ├── index.php
│   └── mysql.php
│ 
├── docker-compose.yml
├── index.php
├── mysql
│   ├── data
│   │   ├── auto.cnf
│   │   ├── ibdata1
│   │   ├── ib_logfile0
│   │   ├── ib_logfile1
│   │   ├── mysql [error opening dir]
│   │   ├── performance_schema [error opening dir]
│   │   └── test_db [error opening dir]
│   └── Dockerfile
├── nginx
│   ├── conf
│   │   └── default.conf
│   └── Dockerfile
└── phpfpm
    ├── php.ini
    ├── composer.phar  
    └── Dockerfile
```

### 说说各个文件夹下的Dockerfile

#### mysql下的Dockerfile

```
FROM mysql:5.7
```

#### phpfpm下的Dockerfile

```
FROM  php:7.1-fpm

MAINTAINER ityike<yuanfeng634@gmail.com>

# Must install dependencies for your extensions, if need.
RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpng12-dev \
    vim \
    git \
    libxml2-dev \
    build-essential \
    openssl \
    libssl-dev \
    make \
    curl \
    libcurl4-gnutls-dev \
    libjpeg-dev \
    libreadline6 \
    libreadline6-dev \

    && docker-php-ext-install -j$(nproc) iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-mcrypt=/usr/include \
    && docker-php-ext-install gd \

# no dependency extension

    && docker-php-ext-install pdo_mysql sockets gettext opcache\

# Install PECL extensions

RUN apt-get install -y \
# for memcache
    libmemcache-dev \

# for memcached
    libmemcached-dev \

    && pecl install memcache && docker-php-ext-enable memcache \
    && pecl install memcached && docker-php-ext-enable memcached \
    && pecl install gearman && docker-php-ext-enable gearman \
    && pecl install yaf && docker-php-ext-enable yaf \

    && pecl install xdebug && docker-php-ext-enable xdebug \
    && pecl install redis && docker-php-ext-enable redis \
    && pecl install xhprof && docker-php-ext-enable xhprof \

    && apt-get clean; rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/* \
    && echo 'PHP 7.1 installed.'

# Other extensions (手动安装)
# RUN curl -fsSL 'https://xcache.lighttpd.net/pub/Releases/3.2.0/xcache-3.2.0.tar.gz' -o xcache.tar.gz \
#     && mkdir -p xcache \
#     && tar -xf xcache.tar.gz -C xcache --strip-components=1 \
#     && rm xcache.tar.gz \
#     && ( \
#         cd xcache \
#         && phpize \
#         && ./configure --enable-xcache \
#         && make -j$(nproc) \
#         && make install \
#     ) \
#     && rm -r xcache \
#     && docker-php-ext-enable xcache


# step1
RUN git clone https://github.com/laruence/yaf.git \
# step2
    && cd yaf \
# step3
    && phpize \
# step4
    && ./configure \
# step5
    && make && make install \
# step6
    && docker-php-ext-enable yaf


# open pid file
# RUN sed -i '/^;pid\s*=\s*/s/\;//g' /usr/local/etc/php-fpm.d/www.conf \
RUN sed -i '/^;pid\s*=\s*/s/\;//g' /usr/local/etc/php-fpm.conf

# add php-fpm to service

    && cp /usr/src/php/sapi/fpm/init.d.php-fpm.in /etc/init.d/php-fpm && chmod +x /etc/init.d/php-fpm

# && chkconfig --add php-fpm

# composer tool
# download from https://getcomposer.org/download/   and then add to php-tools/

ADD ./php-tools/composer.phar /usr/local/bin/composer
RUN chmod 755 /usr/local/bin/composer

WORKDIR "/var/www"

# Volumes

VOLUME ["/var/www"]

# extends from parent
# EXPOSE 9000
# CMD ["php-fpm"]

```

上面Dockerfile中我安装的依赖比较多，按照自己的实际需求入坑！

--------

官方的镜像与直接使用apt-get install 安装php不同，官方的镜像是下载源码包编译安装，安装目录，相关配置也不相同。所以安装扩展的方式也不同。

官方镜像自带了当前版本的php源码包，下载的源码包在`/usr/src/php.tar.xz`。

可使用docker官方提供的 `docker-php-source extract` 来快速解压它，解压目录 `/usr/src/php/`。

同样可以用 `docker-php-source delete` 来快速删除`/usr/src/php/` 目录。

核心扩展是自带在php源码包里面的，解压后在`/usr/src/php/ext`

使用 docker 官方提供的 `docker-php-ext-install [gettext]` 来快速安装并启用扩展。

特别需要注意的是，有的扩展是需要安装依赖的！！！例如 gd、mcrpy的扩展。

根据`docker-php-ext-install` 命令的提示可直接本地编译安装的扩展有

```
bcmath bz2 calendar ctype curl dba dom enchant exif fileinfo filter ftp gd gettext gmp hash iconv imap interbase intl json ldap mbstring mcrypt mysqli oci8 odbc opcache pcntl pdo pdo_dblib pdo_firebird pdo_mysql pdo_oci pdo_odbc pdo_pgsql pdo_sqlite pgsql phar posix pspell readline recode reflection session shmop simplexml snmp soap sockets spl standard sysvmsg sysvsem sysvshm tidy tokenizer wddx xml xmlreader xmlrpc xmlwriter xsl zip
```

##### 其他扩展的安装：（需要手动安装，比如上面Dockerfile中注释的xcache）

下面示例安装一下yaf扩展：

```
# step1
git clone https://github.com/laruence/yaf.git
# step2
cd yaf
# step3
phpize
# step4
./configure
# step5
make && make install
# step6
docker-php-ext-enable yaf
# step7(这个不重要，可以忽略)
rm -r ./yaf
```

##### 有一些需要注意的事

* 	官方的镜像使用的是9000端口来监听的，没有启用 Unix sock，配置nginx 站点时要注意使用 fastcgi_pass [host]:9000;。 需要的话可以在/usr/local/etc/php-fpm.d/www.conf 配置 listen。
* 默认没有写pid文件，也需要在 /usr/local/etc/php-fpm.conf 启用

##### 将 php-fpm 加入 service
将 php-fpm 加入 service 可以更方便的管理(重启、停止php-fpm等)。首先要在php-fpm配置中启用pid文件。 pid文件的默认位置在`/usr/local/php/var/run/php-fpm.pid`。
拷贝php官方提供的service文件到`/etc/init.d`
```
cp /usr/src/php/sapi/fpm/init.d.php-fpm.in /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
```
编辑`/etc/init.d/php-fpm`，加上正确的配置：
```
prefix=@prefix@
exec_prefix=@exec_prefix@

php_fpm_BIN=@sbindir@/php-fpm
php_fpm_CONF=@sysconfdir@/php-fpm.conf
php_fpm_PID=@localstatedir@/run/php-fpm.pid
```

配置后：
```
prefix=/usr/local
exec_prefix=/usr/local/bin/php

php_fpm_BIN=/usr/local/sbin/php-fpm
php_fpm_CONF=/usr/local/etc/php-fpm.conf
php_fpm_PID=$prefix/var/run/php-fpm.pid
```

现在就可以使用 service php-fpm [status|start|reload]等命令了。



#### nginx下的Dockerfile

```
FROM nginx:latest
RUN apt-get update && apt-get install -y vim
```
执行命令：build镜像：
```
docker-compose up --build   or    docker-compose build
```

## 现在三个容器运行只需要使用命令：

```
docker-compose up -d
```
------------

## 最后想要补充的，最后补充的不是不重要。【这一块我现在还没有过多地深入】

`docker pull` 和 `docker build` 创建一个新的Docker image 。
每一个layer就是一个cache，都是使用磁盘空间的。仍然保留原先的版本layer dangling

#### remove untagged image

* macOS用户（xargs命令没有 “-r” 选项）
```
docker images --no-trunc | grep '<none>' | awk '{ print $3 }' \
    | xargs docker rmi
```

#### docker 运行默认的容器，通过日志或退出状态可以查看，同时还存储aufs文件系统的更改，这样就可以将容器commit为新的image


如果以后不需要检查容器，需要使用docker run --rm来标识， 这个标识不适用于后台容器（-d）

```
docker ps --filter status=dead --filter status=exited -aq \
  | xargs docker rm -v
```


#### `docker rm` 不会删除容器创建的卷。

镜像是树型结构，最后一个引用被删除，才是最终的删除
但你需要使用 -v 选项来删除容器中的卷。

* 默认情况下，保存到容器中的磁盘的所有内容都保存在aufs层中。如果你需要清理未使用的容器和镜像，这不会产生问题。
* 如果从主机上挂载文件或者目录（`docker run -v /host/path:/container/path`
）文件将存储在主机文件系统中，因此很容易跟踪它们
* 第三种方式是docker的卷。这些是映射到主机上的`/var/lib/docker/volumes/path`中的特殊目录的特殊路径。很多image使用卷在容器之间共享文件（使用volumes-from选项）或保留数据，以便在进程退出后不会丢失它们（仅数据容器模式）。

那是不是没有办法列出volumes和他们的状态呢？

下面我谷歌到的，我试了一下

```bash
#!/bin/bash

# remove exited containers:
docker ps --filter status=dead --filter status=exited -aq | xargs docker rm -v

# remove unused images:
docker images --no-trunc | grep '<none>' | awk '{ print $3 }' | xargs docker rmi

# remove unused volumes:
find '/var/lib/docker/volumes/' -mindepth 1 -maxdepth 1 -type d | grep -vFf <(docker ps -aq | xargs docker inspect | jq -r '.[] | .Mounts | .[] | .Name | select(.)') | xargs rm -fr
```


## 附属参考文章：

* http://www.yzone.net/blog/120
* https://lebkowski.name/docker-volumes/
* http://www.tanhui.bid/docker/2016/10/19/使用Docker-docker-compose-搭建nginx+php+mysql-环境






















