---
title: 使用Docker搭建Ngrok服务器
date: 2016-06-14 14:41:31
tags:
  - docker
  - ngrok
---

### 准备工作
* 公网服务器一台并且[安装docker](https://docs.docker.com/linux/step_one/),我用的阿里云Centos7.0
* 域名一枚,本文使用 `tunnel.hteen.cn` 做Ngrok服务器域名
* [hteen/ngrok](https://hub.docker.com/r/hteen/ngrok/) Docker镜像

<!-- more -->

### 拉取镜像
```linux
[root@iZ25f738hs2Z ngrok]# docker pull hteen/ngrok
```

### 启动一个容器生成ngrok客户端,服务器端和CA证书
``` linux
[root@iZ25f738hs2Z ngrok]# docker run --rm -it -e DOMAIN="tunnel.hteen.cn" \
-v /data/ngrok:/myfiles hteen/ngrok /bin/bash /build.sh
```
挂载宿主机目录`/data/ngrok`到容器内`/myfiles`目录 ,之后会

```linux
Generating RSA private key, 2048 bit long modulus
...............+++
..........+++
.....
build ok !
[root@iZ25f738hs2Z ngrok]#
```

当看到`build ok !`的时候就成功了

```linux
[root@iZ25f738hs2Z ngrok]# ls -Al
-rw-r--r-- 1 root root 1679 6月  14 18:58 base.key
-rw-r--r-- 1 root root 1111 6月  14 18:58 base.pem
-rw-r--r-- 1 root root   17 6月  14 18:58 base.srl
drwxr-xr-x 4 root root 4096 6月  14 19:01 bin
-rw-r--r-- 1 root root  993 6月  14 18:58 device.crt
-rw-r--r-- 1 root root  899 6月  14 18:58 device.csr
-rw-r--r-- 1 root root 1675 6月  14 18:58 device.key
```
生成了我们要的客户端和服务端在`/data/ngrok/bin`目录下,包括

```linux
bin/ngrokd                  服务端
bin/ngrok                   linux客户端
bin/darwin_amd64/ngrok      osx客户端
bin/windows_amd64/ngrok.exe windows客户端
```

### 启动Ngrok server
直接挂载刚刚的`/data/ngrok`到容器即可启动服务

 ```linux
 [root@iZ25f738hs2Z ngrok]# docker run -idt --name ngrok-server \
 -v /data/ngrok:/myfiles \
 -p 80:80 \
 -p 443:443 \
 -p 4443:4443 \
 -e DOMAIN='tunnel.hteen.cn' hteen/ngrok /bin/bash /server.sh
 ```
这样我们就启动了一个ngrok服务端程序

### 域名解析
这里我们需要添加两条A记录到阿里云服务器

![域名解析](https://cdn2.hteen.cn/public/u1rwt.jpg)

这样我们才能将 `tunnel.hteen.cn` 和 `*.tunnel.hteen.cn` DNS解析到我们的服务器

### 客户端连接
下载我们生成的客户端,我这里以osx为例,其他平台一样

首先创建一个`ngrok.cfg`配置文件

```shell
server_addr: "tunnel.hteen.cn:4443"
trust_host_root_certs: false
```
然后在命令行执行

```linux
./ngrok -config ./ngrok.cfg -subdomain wechat 192.168.99.100:80
```
我这里是将`wechat.tunnel.hteen.cn`绑定的本地`192.168.99.100:80`

如果不指定`-subdomain`参数,每次启动客户端的时候会随机分配一个域名,随意并不方便

成功连接效果

![成功连接](https://cdn2.hteen.cn/public/k1c57.jpg)

### Nginx + Docker + Ngrok

由于ngrok默认使用80和443端口,我服务器已经运行了Nginx服务 ,
所以我这里启动Ngrok Server的时候并不是绑定的80和443端口,而是绑定的`8082`和`4432`

```linux
[root@iZ25f738hs2Z ngrok]# docker run -idt --name ngrok-server \
-v /data/ngrok:/myfiles \
-p 8082:80 \
-p 4432:443 \
-p 4443:4443 \
-e DOMAIN='tunnel.hteen.cn' hteen/ngrok /bin/bash /server.sh
```

启动之后需要在`nginx.conf` 添加两条反向代理配置

```nginx
server {
     listen       80;
     server_name  tunnel.hteen.cn *.tunnel.hteen.cn;

     location / {
             proxy_redirect off;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_pass http://10.24.198.241:8082;
     }

 }

 server {
     listen       443;
     server_name  tunnel.hteen.cn *.tunnel.hteen.cn;

     location / {
             proxy_redirect off;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_pass http://10.24.198.241:4432;
     }

 }

```
`10.24.198.241` 是我阿里云内网IP

### 结语

如果你暂时不准备搭建自己的ngrok服务器,也可以使用我目前的服务器

客户端在这里[下载](http://pan.baidu.com/s/1eSCwEFw)

^_^ Have a nice day
