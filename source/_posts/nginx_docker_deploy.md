---
title: 记一次使用 nginx 和 docker 部署
tags: 
  - Linux
  - nginx
author:
  nick: Lovae
categories: nginx

cover: 
date: 2018-7-02

subtitle: 一次踩坑记录

---
## 记一次部署项目的惨痛经历

一天，小明买了一台服务器，要部署一个小网页。

我一想，这还不简单？考虑到以后可能要添加新服务，于是乎安装了 docker，docker-compose，nginx。

### nginx

管他三七二十一，先起个 nginx 服务  `systemctl start docker`，再 `curl 127.0.0.1`，是 nginx 的欢迎页面，OK！

再用浏览器访问服务器 IP 地址。。。无响应！？抓个包看看：

![](/images/linux/no_conn.png)

Emmm…去看下 nginx 错误日志，`cat /var/log/nginx/error.log`

```
[notice] 63762#0: signal process started
```

没毛病啊！再看看访问日志：`cat /var/log/nginx/error.log`。。。空的！

首先 nginx 服务是没毛病的，因为内网能访问；浏览器访问不到，那就是防火墙的问题了。因为默认使用 80 端口，所以配置 iptables 规则，接收所有 80 的访问。

```bash
/sbin/iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

再访问，果然 ojbk(垃圾运营商默认封掉了 80 端口，不厚道)

### docker

再起个 docker 服务 `systemctl start docker`

```
Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
```

Wtf?! 好吧，看看什么情况：`systemctl status docker.service`

```Bash
Jul 02 15:03:50 level=warning msg="could not change group /var/run/docker.sock to docker: group docker not found"
Jul 02 15:03:50 level=info msg="libcontainerd: new containerd process, pid: 3981"
Jul 02 15:03:52 Error starting daemon: SELinux is not supported with the overlay2 graph driver on this kernel. Either boot into a newer kernel or disab...abled=false) ## 在这行
Jul 02 15:03:52 docker.service: main process exited, code=exited, status=1/FAILURE
Jul 02 15:03:52 Failed to start Docker Application Container Engine.
Jul 02 15:03:52 Unit docker.service entered failed state.
Jul 02 15:03:52 docker.service failed.
```

SELinux 不支持 docker 的 overlay driver ？可是 SELinux 是个啥？[看这里](https://zh.wikipedia.org/wiki/%E5%AE%89%E5%85%A8%E5%A2%9E%E5%BC%BA%E5%BC%8FLinux)和[这里](https://www.jianshu.com/p/15f842095de8)。查了一下，基本上是这样：

> 解决方法有两个，要么启动一个新内核，要么就在docker里禁用selinux，--selinux-enabled=false
>
> 重新编辑docker配置文件： `vim /etc/sysconfig/docker`，在 —selinux-enable 后加上 =false：
>
> ```bash
> OPTIONS='--selinux-enabled=false --log-driver=journald --signature-verification=false'
> ```

再启动 docker ，OK！

docker 起一个 nginx Server，把网页放上去。

```bash
docker run --name example -p 127.0.0.1:8000:80 -v /html:/usr/share/nginx/html:ro nginx
```

### 反向代理

上面 docker 服务不希望使用 IP 端口访问，在 127.0.0.1 上，还要再开个反向代理。

在 `/etc/nginx/conf.d` 里 `vi example.com.conf`，复制下面这段：

```bash
server {
    server_name example.com;
    listen 80;
    location / {
        proxy_pass  http://127.0.0.1:8000; #这里端口号和 docker 的 nginx 服务对应
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
    }
}
```

由于主配置文件 `/etc/nginx/nginx.conf`  里会引入所有  `/etc/nginx/conf.d`  的 .conf 文件，所以这个改动是有效的。重启 nginx：`nginx -s reload` 。

本来到这里应该结束了，浏览器访问 example.com 就能访问页面了。然而。。。

![](/images/linux/502.png)

？？？wtf ？！不应该啊！

* 难道 docker 挂了？用 `curl 127.0.0.1:8000` 试一下，没毛病，是正常的页面！

* 那难道是主 nginx 挂了？防火墙？浏览器访问 IP 主机 IP 地址：

  ![](/images/linux/nginx_server.png)

  也没毛病啊！

* 难道反向代理配置没写对？检查了 server_name，没写错，`cat /var/log/nginx/error.log`，没有错误日志啊！

到这里我就彻底懵逼了，乱试了很久，无果。

等等！！！会不会是 SELinux ？！不管了，先把 SELinux 关掉再说！修改配置文件 `/etc/selinux/config`和 `/etc/sysconfig/selinux`（`/etc/sysconfig/selinux` 是一个符号链接，真正的配置文件为：`/etc/selinux/config`）

```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disable ### 这里改为禁用(重启系统才能生效)，不过先不建议这样做。
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

再访问 example.com ，OK！

终于知道是什么原因了，不过这个解决方案让我不太满意，因为 disable 意味着系统安全性降低，有没有代价小一点的方案呢？有！

在这个例子中 SELinux 主要限制了网络访问，所以单独解除这个限制：

```bash
setsebool -P httpd_can_network_connect 1
```

再访问，就 OK 了。

### 总结

整个过程还是比较痛苦的，主要是排查问题所在的过程和思路不够清晰，花了挺长时间。还有就是对系统本身不够熟悉，一些特性知道，导致深坑。