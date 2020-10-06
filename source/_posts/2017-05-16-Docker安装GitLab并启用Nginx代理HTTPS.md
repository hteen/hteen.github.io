---
title: Docker安装GitLab并启用Nginx代理HTTPS
date: 2017-05-16 15:57:45
tags:
    - docker
    - nginx
    - proxy
    - gitlab
    - https
---

<blockquote class="blockquote-center">
    基于docker安装gitlab, 并使用nginx代理实现https访问仓库
</blockquote>

<!-- more -->

## 准备工作
* 域名 `foo.bar.com`
* 域名SSL证书
* Nginx代理镜像 [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/)
* GitLab中文镜像 [twang2218/gitlab-ce-zh](https://hub.docker.com/r/twang2218/gitlab-ce-zh/)

## 创建Nginx代理容器

创建自定义网络, 仅仅是为了方便维护, 也可以不创建使用默认的

```linux
docker network create proxy
```

创建代理容器

```linux
docker run --name proxy --network proxy -d -p 80:80 -p 443:443 \
    -v /data/ssl:/etc/nginx/certs \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/nginx-proxy
```

> 需要将SSL证书挂载到容器内 `/etc/nginx/certs` 以启用域名HTTPS访问

## 创建GitLab容器
由于GitLab配置参数太多所以使用[docker-compose](https://docs.docker.com/compose/)

创建相应的数据目录和docker-compose.yml

```linux
mkdir -p /data/gitlab/{config,data,logs}
cd /data/gitlab
vim docker-compose.yml
```

docker-compose.yml 文件内容

```yml
version: '2'
services:
    gitlab:
      image: 'twang2218/gitlab-ce-zh'
      restart: unless-stopped
      hostname: 'foo.bar.com'
      environment:
        VIRTUAL_HOST: 'foo.bar.com'
        TZ: 'Asia/Shanghai'
        GITLAB_OMNIBUS_CONFIG: |
          external_url 'https://foo.bar.com' #启用HTTPS域名
          gitlab_rails['time_zone'] = 'Asia/Shanghai'
          # 需要配置到 gitlab.rb 中的配置可以在这里配置，每个配置一行，注意缩进。
          # 比如下面的电子邮件的配置：
          # gitlab_rails['smtp_enable'] = true
          # gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
          # gitlab_rails['smtp_port'] = 465
          # gitlab_rails['smtp_user_name'] = "xxxx@xx.com"
          # gitlab_rails['smtp_password'] = "password"
          # gitlab_rails['smtp_authentication'] = "login"
          # gitlab_rails['smtp_enable_starttls_auto'] = true
          # gitlab_rails['smtp_tls'] = true
          # gitlab_rails['gitlab_email_from'] = 'xxxx@xx.com'
          
          # 支持代理SSL需要增加以下配置
          nginx['listen_port'] = 80
          nginx['listen_https'] = false
          nginx['proxy_set_headers'] = {
            "X-Forwarded-Proto" => "https",
            "X-Forwarded-Ssl" => "on"
          }
      volumes:
        - /data/gitlab/config:/etc/gitlab
        - /data/gitlab/data:/var/opt/gitlab
        - /data/gitlab/logs:/var/log/gitlab
networks:
  default:
    external:
      name: proxy #容器创建时默认网络使用之前创建好的
```

> GitLab支持代理SSL[官方说明](https://docs.gitlab.com.cn/omnibus/settings/nginx.html#enable-https)

启动容器

```linux
docker-compose up -d
...
docker ps
CONTAINER ID     IMAGE                    PORTS                                      NAMES
23621d9fe059     twang2218/gitlab-ce-zh   22/tcp, 80/tcp, 443/tcp                    gitlab_gitlab_1
5a47c2dd8ebf     jwilder/nginx-proxy      0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   proxy
```

启动后初始化会有几分钟时间, 之后通过 `https://foo.bar.com` 访问就能看到GitLab的初始页面

设置好管理员账号后创建测试仓库就能看到仓库默认使用https

![](https://cdn2.hteen.cn/public/awrhk.jpg)

> 关于如何使用SSH访问仓库可以[查看文档](https://github.com/twang2218/gitlab-ce-zh#配置-ssh-端口)

