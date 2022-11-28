---
title: CTFd+CTFd-Whale部署实践
categories:
  - 杂记
tags:
  - CTF
abbrlink: b16d8429
date: 2022-11-28 20:06:27
updated: 2022-11-28 20:06:27
---
#CTF 
# 引言

放暑假了，想着搭建一个ctf靶场训练下新生，但遇到的问题比我想象中的多得多。。。写个blog记录一下。

# 参考文章

  

- [ctfd使用ctfd-whale动态靶机插件搭建靶场指南](https://vaala.cat/2020/09/21/ctfd%E4%BD%BF%E7%94%A8ctfd-whale%E5%8A%A8%E6%80%81%E9%9D%B6%E6%9C%BA%E6%8F%92%E4%BB%B6%E6%90%AD%E5%BB%BA%E9%9D%B6%E5%9C%BA%E6%8C%87%E5%8D%97/)

- [手把手教你如何建立一个支持ctf动态独立靶机的靶场（ctfd+ctfd-whale）](https://blog.csdn.net/fjh1997/article/details/100850756)

- [CTFd-Whale 推荐部署实践](https://www.zhaoj.in/read-6333.html/comment-page-1#_Direct_Frp)

  

~~三篇文章所使用的方法基本上差不多，但是部署的ctfd版本和插件版本不尽相同，第一篇文章中使用的是~~[~~frankli0324~~](https://github.com/frankli0324/ctfd-whale)~~师傅修改后的ctfd-whale，但是部署上之后测试发现，renew功能无法使用，不知道是不是我部署错误的问题，而第二篇则是无法正确的映射题目的端口，所以最后还是选择了官方的部署教程。~~

经过更新后，[frankli0324](https://github.com/frankli0324/ctfd-whale)师傅修改后的ctfd-whale已经可以正常使用。

# 部署

## 环境准备

  

- 安装docker

- 安装docker-compose

## 配置swarm

```php

docker swarm init

docker node update --label-add='name=linux-1' $(docker node ls -q)

```

## 安装

  

1. 下载ctfd

```php

git clone https://github.com/CTFd/CTFd --depth=1

cd CTFd

```

  

2. 修改docker-compose.yml

```python

version '2' -> version '3'

```

接着`docker-compose up -d`，对CTFd进行初始配置。

  

3. 配置frps，在docker-compose-yml中配置如下：

```php

services:

    ...

    frps:

        image: glzjin/frp

        restart: always

        volumes:

            - ./conf/frp:/conf

        entrypoint:

            - /usr/local/bin/frps

            - -c

            - /conf/frps.ini

        ports:

            - 10000-10100:10000-10100  # 映射direct类型题目的端口

            - 8001:8001  # 映射http类型题目的端口

        networks:

            default:  # 需要将frps暴露到公网以正常访问题目容器

            frp_connect:

              ipv4_address: 172.1.0.3

  

networks:

    ...

    frp_connect:

        driver: overlay

        internal: true

        ipam:

            config:

                - subnet: 172.1.0.0/16

```

  

3. 创建目录 ./conf/frp，并创建`frps.ini`文件，填写：

```dockerfile

[common]

# 下面两个端口注意不要与direct类型题目端口范围重合

bind_port = 7987  # frpc 连接到 frps 的端口

vhost_http_port = 8001  # frps 映射http类型题目的端口

token = your_token

subdomain_host = node3.buuoj.cn  # 访问http题目容器的主机名，需要配置泛解析域名

```

**之后部署时需要把注释内容全部删除，否则会报错。**

  

4. 配置frpc，同样修改docker-compose.yml

```bash

services:

    ...

    frpc:

        image: glzjin/frp:latest

        restart: always

        volumes:

          - ./conf/frp:/conf/

        entrypoint:

          - /usr/local/bin/frpc

          - -c

          - /conf/frpc.ini

        depends_on:

          - frps #frps需要先成功运行

        networks:

            frp_containers:  # 供frpc访问题目容器

            frp_connect:  # 供frpc访问frps, CTFd访问frpc

                ipv4_address: 172.1.0.4

  

networks:

    ...

    frp_containers:

        driver: overlay

        internal: true  # 如果允许题目容器访问外网，则可以去掉

        attachable: true

        ipam:

            config:

                - subnet: 172.2.0.0/16

```

  

5. 创建./conf/frp/frpc.ini，填写：

```python

[common]

token = your_token

server_addr = 172.1.0.3

server_port = 7987  # 对应 frps 的 bind_port

admin_addr = 172.1.0.4  # 请参考“安全事项”

admin_port = 7400

```

  

6. 检查frp配置是否正确

```bash

docker-compose up -d

docker-compose logs frpc

```

查看日志是否正确，正确情况如下：

```python

[service.go:224] login to server success, get run id [******], server udp port [******]

[service.go:109] admin server listen on ******

```

  

6. 配置CTFd

```dockerfile

services:

    ctfd:

        ...

        volumes:

            - /var/run/docker.sock:/var/run/docker.sock

        depends_on:

            - frpc #frpc需要先运行

        networks:

            ...

            frp_connect:

```

照着修改docker-compose.yml即可。

  

7. 下载CTFd-Whale

```dockerfile

git clone https://github.com/frankli0324/CTFd-Whale CTFd/plugins/ctfd-whale --depth=1

docker-compose build # 需要安装依赖

docker-compose up -d

```

  

8. 配置ctfd-whale

   1. Auto Connect Network：ctfd_frp_containers

   2. Http Domain Suffix：设置好的泛解析域名

   3. Http Port：同frps.ini的vhost_http_port

   4. Direct IP Address：公网ip

  

设置完成后理论上应该就成功了。

完整的docker-compose.tml:

```dockerfile

version: '3'

  

services:

  ctfd:

    build: .

    user: root

    restart: always

    ports:

      - "8000:8000"

    environment:

      - UPLOAD_FOLDER=/var/uploads

      - DATABASE_URL=mysql+pymysql://ctfd:ctfd@db/ctfd

      - REDIS_URL=redis://cache:6379

      - WORKERS=1

      - LOG_FOLDER=/var/log/CTFd

      - ACCESS_LOG=-

      - ERROR_LOG=-

      - REVERSE_PROXY=true

    volumes:

      - .data/CTFd/logs:/var/log/CTFd

      - .data/CTFd/uploads:/var/uploads

      - .:/opt/CTFd:ro

      - /var/run/docker.sock:/var/run/docker.sock

    depends_on:

      - db

      - frpc

    networks:

        default:

        internal:

        frp_connect:

          ipv4_address: 172.1.0.5

  

  nginx:

    image: nginx:1.17

    restart: always

    volumes:

      - ./conf/nginx/http.conf:/etc/nginx/nginx.conf

    ports:

      - 80:80

    depends_on:

      - ctfd

  

  db:

    image: mariadb:10.4.12

    restart: always

    environment:

      - MYSQL_ROOT_PASSWORD=ctfd

      - MYSQL_USER=ctfd

      - MYSQL_PASSWORD=ctfd

      - MYSQL_DATABASE=ctfd

    volumes:

      - .data/mysql:/var/lib/mysql

    networks:

        internal:

    # This command is required to set important mariadb defaults

    command: [mysqld, --character-set-server=utf8mb4, --collation-server=utf8mb4_unicode_ci, --wait_timeout=28800, --log-warnings=0]

  

  cache:

    image: redis:4

    restart: always

    volumes:

    - .data/redis:/data

    networks:

        internal:

  frps:

        image: glzjin/frp

        restart: always

        volumes:

            - ./conf/frp:/conf

        entrypoint:

            - /usr/local/bin/frps

            - -c

            - /conf/frps.ini

        ports:

            - 10000-10100:10000-10100  # 映射direct类型题目的端口

            - 8001:8001  # 映射http类型题目的端口

        networks:

            default:  # 需要将frps暴露到公网以正常访问题目容器

            frp_connect:

              ipv4_address: 172.1.0.3

  frpc:

        image: glzjin/frp:latest

        restart: always

        volumes:

          - ./conf/frp:/conf/

        entrypoint:

          - /usr/local/bin/frpc

          - -c

          - /conf/frpc.ini

        depends_on:

          - frps #frps需要先成功运行

        networks:

            frp_containers:  # 供frpc访问题目容器

            frp_connect:  # 供frpc访问frps, CTFd访问frpc

                ipv4_address: 172.1.0.4

  

networks:

    default:

    internal:

        internal: true

    frp_connect:

        driver: overlay

        internal: true

        ipam:

            config:

                - subnet: 172.1.0.0/16

    frp_containers:

        driver: overlay

        internal: true  # 如果允许题目容器访问外网，则可以去掉

        attachable: true

        ipam:

            config:

                - subnet: 172.2.0.0/16

```

# 遇到的问题

## 部署时提示找不到python和python-dev

```dockerfile

ERROR: unable to select packages:

    required by: world[python]

  python-dev (no such package):

    required by: world[python-dev]

```

修改python和python-dev为python3和python3-dev

## 编译时提示cannot execute 'cc1plus'

Dockerfile中添加安装g++

```dockerfile

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \

    apk update && \

    apk add \

        ...

        g++ \

        ...

```

## docker ps正常但无法进入网站

docker logs 容器id显示

```bash

/usr/local/lib/python3.7/site-packages/tzlocal/unix.py:158: UserWarning: Can not find any timezone configuration, defaulting to UTC.

  warnings.warn('Can not find any timezone configuration, defaulting to UTC.')

Starting CTFd

/usr/local/lib/python3.7/importlib/_bootstrap.py:219: RuntimeWarning: greenlet.greenlet size changed, may indicate binary incompatibility. Expected 144 from C header, got 152 from PyObject

  return f(*args, **kwds)

/usr/local/lib/python3.7/importlib/_bootstrap.py:219: RuntimeWarning: greenlet.greenlet size changed, may indicate binary incompatibility. Expected 144 from C header, got 152 from PyObject

  return f(*args, **kwds)

/usr/local/lib/python3.7/importlib/_bootstrap.py:219: RuntimeWarning: greenlet.greenlet size changed, may indicate binary incompatibility. Expected 144 from C header, got 152 from PyObject

  return f(*args, **kwds)

[2020-10-11 12:31:30 +0000] [1] [INFO] Starting gunicorn 19.9.0

[2020-10-11 12:31:30 +0000] [1] [INFO] Listening at: http://0.0.0.0:8000 (1)

[2020-10-11 12:31:30 +0000] [1] [INFO] Using worker: gevent

[2020-10-11 12:31:30 +0000] [21] [INFO] Booting worker with pid: 21

[2020-10-11 12:31:31 +0000] [23] [INFO] Booting worker with pid: 23

[2020-10-11 12:31:32 +0000] [25] [INFO] Booting worker with pid: 25

[2020-10-11 12:31:34 +0000] [27] [INFO] Booting worker with pid: 27

[2020-10-11 12:31:35 +0000] [29] [INFO] Booting worker with pid: 29

[2020-10-11 12:31:36 +0000] [31] [INFO] Booting worker with pid: 31

[2020-10-11 12:31:37 +0000] [33] [INFO] Booting worker with pid: 33

[2020-10-11 12:31:39 +0000] [35] [INFO] Booting worker with pid: 35

[2020-10-11 12:31:40 +0000] [37] [INFO] Booting worker with pid: 37(一直在加)

```

删除requirements.txt中gevent的版本号。

## 无法更新ctfd-whale配置

这个问题很奇怪，我只遇到过一次，然后放了一晚上之后他就好了

所以，解决方法是，建议重启docker