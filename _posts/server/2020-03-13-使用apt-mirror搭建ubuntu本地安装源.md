---
layout:     post
title:      使用apt-mirror搭建ubuntu本地安装源
subtitle:   
date:       2020-03-13
author:     skai
header-img: 
catalog: true
categories: 服务器搭建
tags:
    - 服务器搭建
---
# 使用apt-mirror搭建ubuntu本地安装源

### 一. 安装mirror

apt-get install apt-mirror

### 二. 创建源存放目录

我的源安装目录为/mnt/e/mirror-data

在mirror-data中建立对应的目录，比如我需要14.04、16.04和ros的源

16.04目录为/mnt/e/mirror-data/16.04, 在此目录下新建mirro , var, skel目录

14.04目录为/mnt/e/mirror-data/14.04, 在此目录下新建mirro , var, skel目录

ros indigo目录为: /mnt/e/mirror-data/ros/ubuntu/indigo, 在此目录下新建mirro , var, skel目录

ros kenetic目录为: /mnt/e/mirror-data/ros/ubuntu/kinetic, 在此目录下新建mirro , var, skel目录

### 三. 配置mirror

**拷贝四份mirror.list文件, 进行修改**

```
cp /etc/apt/mirror.list /etc/apt/mirror.list.14.04
cp /etc/apt/mirror.list /etc/apt/mirror.list.16.04
cp /etc/apt/mirror.list /etc/apt/mirror.list.ros.indigo
cp /etc/apt/mirror.list /etc/apt/mirror.list.ros.kinetic 
```

以16.04为例mirror.list修改如下


```
############# config ##################
#
 set base_path    /mnt/e/mirror-data/ubuntu/16.04
#
 set mirror_path  $base_path/mirror
 set skel_path    $base_path/skel
 set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
 set defaultarch  amd64
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     100
set _tilde 0
#
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
clean http://mirrors.aliyun.com/ubuntu
```
ubuntu各版本对应如下:

网上找了下，各版本对应如下

| 版本号 | Codename |
| ------ | -------- |
| 11.04  | natty    |
| 11.10  | oneiric  |
| 12.04  | precise  |
| 12.10  | quantal  |
| 13.04  | raring   |
| 13.10  | saucy    |
| 14.04  | trusty   |
| 14.10  | utopic   |
| 15.04  | vivid    |
| 15.10  | wily     |
| 16.04  | xenial   |
| 16.10  | yakkety  |

### 四. 开始下载apt-mirror源

建立mirrorcron.sh脚本，内容如下:

```
#!/bin/bash
/bin/cp -rf /etc/apt/mirror.list.16.04 /etc/apt/mirror.list
/usr/bin/apt-mirror
/bin/cp -rf /etc/apt/mirror.list.14.04 /etc/apt/mirror.list
/usr/bin/apt-mirror
/bin/cp -rf /etc/apt/mirror.list.ros.indigo /etc/apt/mirror.list
/usr/bin/apt-mirror
/bin/cp -rf /etc/apt/mirror.list.ros.kinetic /etc/apt/mirror.list
/usr/bin/apt-mirror
```

运行脚本./mirrorcron.sh进行更新



### 五. 设置定时更新(可忽略)

vi /etc/crontab

增加一行，每天凌晨1点开始同步（需建立对应的日志目录）

0  1    * * *   root    /etc/apt/mirrorcron.sh &>/var/log/mirror/cron.log 2>&1



### 六. 安装apache2

输入以下命令安装:

```
apt-get install apache2
```

**修改apache2的端口**

/etc/apache2/ports.conf 中修改为Listen 8080

/etc/apache2/sites-enabled/000-default.conf 中80修改为8080 

**将/mnt/e/mirror-data映射到/var/www/html**

```
ln -s /mnt/e/mirror-data  /var/www/html/mirror
```

重启apache2 ：

```
sudo service apache2 restart
```

输入http://ip:8080/即可看到mirror目录



### 七. 安装nginx

因apache2实际源的路径太长，所以使用Nginx进行代理

```
apt-get install nginx
```

/etc/nginx/nginx.conf修改为
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 768;
}
http {
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;


        gzip on;
        gzip_disable "msie6";

        upstream apt-yum
        {
          server 192.168.42.248:8080;
        }

        server {
            listen 8082;
            server_name localhost;
            root /var/www/html;
            location / {
            root html;
            index index.html index.htm;
            }
       
           location /ubuntu/14.04 {
           proxy_pass  http://apt-yum/mirror/ubuntu/14.04/mirror/mirrors.aliyun.com/ubuntu/;
           proxy_redirect off;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
           }
           location /ubuntu/16.04 {
                   proxy_pass  http://apt-yum/mirror/ubuntu/16.04/mirror/mirrors.aliyun.com/ubuntu/;
                   proxy_redirect off;
                   proxy_set_header Host $host;
                   proxy_set_header X-Real-IP $remote_addr;
                   proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
           }
           location /ros/indigo {
                   proxy_pass  http://apt-yum/mirror/ros/ubuntu/indigo/mirror/packages.ros.org/ros/ubuntu/;
                   proxy_redirect off;
                   proxy_set_header Host $host;
                   proxy_set_header X-Real-IP $remote_addr;
                   proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
           }
           location /ros/kinetic {
                   proxy_pass  http://apt-yum/mirror/ros/ubuntu/kinetic/mirror/packages.ros.org/ros/ubuntu/;
                   proxy_redirect off;
                   proxy_set_header Host $host;
                   proxy_set_header X-Real-IP $remote_addr;
                   proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
           }
       
           }
   }
```
### 八.使用源

**ubuntu 16.04源**

```
deb http://192.168.42.248:8082/ubuntu/16.04/ xenial main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/16.04/ xenial-security main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/16.04/ xenial-updates main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/16.04/ xenial-backports main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/16.04/ xenial-proposed main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/16.04/ xenial main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/16.04/ xenial-security main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/16.04/ xenial-updates main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/16.04/ xenial-backports main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/16.04/ xenial-proposed main restricted universe multiverse 
```

**ubuntu 14.04源**

```
deb http://192.168.42.248:8082/ubuntu/14.04/ trusty main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/14.04/ trusty-security main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/14.04/ trusty-updates main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/14.04/ trusty-backports main restricted universe multiverse 
deb http://192.168.42.248:8082/ubuntu/14.04/ trusty-proposed main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/14.04/ trusty main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/14.04/ trusty-security main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/14.04/ trusty-updates main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/14.04/ trusty-backports main restricted universe multiverse 
deb-src http://192.168.42.248:8082/ubuntu/14.04/ trusty-proposed main restricted universe multiverse
```

**Ros 14.04源**

```
sudo sh -c 'echo "deb http://192.168.42.248:8082/ros/indido `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver [keyserver.ubuntu.com](http://keyserver.ubuntu.com/) --recv-keys F42ED6FBAB17C654
```

**Ros 16.04源**

```
sudo sh -c 'echo "deb http://192.168.42.248:8082/ros/kinetic `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver [keyserver.ubuntu.com](http://keyserver.ubuntu.com/) --recv-keys F42ED6FBAB17C654
```

将以上内容覆盖/etc/apt/source.list

再执行sudo apt-get update就可以