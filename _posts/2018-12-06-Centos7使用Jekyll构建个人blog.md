---
layout: post
title: Centos7使用Jekly构建个人blog
modified: 2018-12-6
categories: web
tags: 
  - CentOS
  - 建站
comments: true
---


最近趁着`Black Friday`,撸了个北美小鸡.在小鸡上建立了个人的博客.

本blog使用的是`jekyll` 和github上的blog是一样的.当然你也可以使用`hexo`.

域名购买的话,笔者是在`gandi`上买的域名,但是大家更加推荐去`namesilo`.

使用`jekyll` ,推荐使用rvm(http://www.rvm.io) 去安装`ruby`,再用`ruby`安装`jekyll`

<!-- more -->

**安装`rvm`**

```shell
#配置key
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
#安装
\curl -sSL https://get.rvm.io | bash -s stable
#重新加载一下rvm的运行环境,确保rvm可以运行
source ~/.rvm/scripts/rvm
#在命令行中输入,检测是否安装成功
rvm -v
```



 **安装`ruby`**

```shell
rvm install ruby
rvm --default use ruby
```

 **安装`jekyll`**

```shell
#这里还需要另外一个bundler
gem install jekyll bundler
```

理论上`jekyll`是安装完成了,如果你使用的是国内的云服务器,遇到网络错误,可以修改`gem`的源

```shell
#查看gem源
gem sources -l 
#gem删除原来默认的源
gem sources -r https://rubygems.org
#gem 添加 一个https源(注意 ,如果https还是不行,需要将s去掉)
gem source -a https://gems.ruby-china.com
```

我们选择一个目录作为web的root目录

```shell
#创建目录
mkdir -p /www
#进入 jinji目录
cd /www
#创建jekyll
jekyll new myblog
#在后台启动
bundle exec jekyll serve --detach
#退出
```

退回到root目录,wget 一下,主要是为了检查本地的`jekyll`是否已经启动成功了

```shell
cd ~
wget http://127.0.0.1:4000
#如果一切是行的通的话,这里是会显示jekyll默认的首页内容的
#然后删掉
rm -rf index.html
```



到了这里 jekyll已经结束了,为了让别人能够访问到你的网站 ,就需要去配置域名解析(DNS)

顺便配置下https.接下来就继续配置`Nginx`,而在配置`nginx`

**配置DNS**

配置dns非常简单.

在gandi的域解析记录添加以下内容

![Alt text](https://jinji.io/images/cname_config.png)



使用nginx包装

```shell
#安装nginx 
yum update -y
yum install nginx 
```

在安装过程中 出现一个问题` No package nginx available`,这个问题是因为`centos`所提供的原始软件包安装器没有这个`nginx`所导致的,安装一个`epel-release`替换即可

```shell
#重命名原来的,对其进行备份
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
#使用aliyun的
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
#清除缓存
yum makecache
#安装epel-release(提供软件包用的)
yum install epel-release
yum install nginx
```

**配置nginx服务器配置**

`/etc/nginx/nginx.conf ` 是`nginx `默认的配置文件,当他被使用时会加载像conf.d中的service内容,每一个service相当于一个小的主机,也可以在里面直接配置https的信息(下面会写)

找到nginx中的server 80端口部分(38~57行)

```shell
#以防万一 备份!
cp /etc/nginx/nginx.conf  /etc/nginx/nginx.conf.bak
vim /etc/nginx/nginx.conf
#显示下行号
:set number 
#请将myblog.com更换为你的域名
#请注意 每行需要以分号结尾
--------------------------------------------
server {
    listen 80;
    server_name myblog.com;
    location / {
       #如果你的blog目前没有添加https,直接做跳转即可
        proxy_pass http://localhost:4000;
    }
}
--------------------------------------------
#保存
:wq!
#重启 nginx
systemctl restart nginx
```

重启的时候可能会提示错误

最常见的就是`nginx: bind() to 0.0.0.0:80 failed (98: Address already in use) `

使用这个命令结束掉80端口的问题,再次运行`systemctl restart nginx` 即可

```shell
fuser -k 80/tcp
```

这个时候其实网站还是不能访问的,因为默认情况下服务器的防火墙会关闭80端口

```shell
firewall-cmd --zone=public --add-port=80/tcp --permanent 
#可能会提示FirewallD is not running
#开启它!
systemctl start firewalld
#看到绿色文字 running
#80端口开启
firewall-cmd --zone=public --add-port=80/tcp --permanent 
#如果有https的话,你还需要开启443
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --reload 
```

到这里,你就可以通过域名访问到你的个人blog了.

**添加https**

- https://certbot.eff.org/lets-encrypt/centosrhel7-nginx
- 去国内的云服务商那里申请免费的证书

注册免费的https有两种方法 ,很遗憾.使用自动签名的方式,在我这里并不起作用(原因始终无法找到)后来我使用了`aliyun`的免费证书 .在aliyun 搜索`ssl证书`  即可找到注册的页面,选择`免费版DV SSL` 其要求你在dns解析的网站上添加一条域名解析内容,点击验证后,等待两个小时左右会有一条验证通过的短信,下载证书的`pem`和`key` 上传到服务器的某个目录下笔者放在了 `/etc/nginx` 下建立了一个`cert`目录,并将这两个文件放入, 配置在`ssl_certificate`和`ssl_certificate_key`下.下图是在gandi中配置验证的部分

![Alt text](https://jinji.io/images/cname_https_auth.png)

对nginx进行修改

```shell
#443端口
server {
    listen 443 ssl default_server ;
    listen [::] ssl default_server;
    #修改这里的myblog.com 为你的域名
    server_name myblog.com;
    #你的pem文件
    ssl_certificate   your.pem;
    #你的key文件
    ssl_certificate_key  your.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location /{
        #你网站的路径, root是你本地网站的位置,请检查root目录下是否有对应的index.html,index.htm等文件
        #否则会出现403(Forbidden)错误
        root /www;
        index index.html;
    }
}
#80端口
server {
    listen 80;
    server_name myblog.com;
    location / {
	   #如果有https ,不要写上面这条, 
       #proxy_pass http://localhost:4000;
	   #即只写这一条
       #意思是如果有访问的话,就会作直接跳转
       #由原来的http 直接跳转为https,这个需要
       #配合443端口的内容
        rewrite ^(.*)$  https://myblog.com;
    }
}
#重启nginx
systemctl restart nginx
```

笔者曾经在这里遇到`403错误` ,经过排查是443端口中,`root` 目录和`index`配置错误(路径错误) 导致的.



**配置git与hooks**

- git基础设置

```shell
#安装git
yum install git -y
#验证 
git -v
#配置git
git config --global user.email "your_email@example.com"
git config --global user.name "Your Name"
#添加服务器的git账号 用于维护git 
sudo adduser git
#输入git密码
passwd git 

#切换到git账号
su  git
#添加git免输密码的ssh-key 验证
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
#将客户端生成的~/.ssh/id_rsa.pub  添加到 git用户目录下 
#1) .ssh目录的权限必须是700 
#2) .ssh/authorized_keys文件权限必须是600
touch authorized_keys 
chmod 600 authorized_keys
cat your.pub >>  ~/.ssh/authorized_keys
#在你的客户端上验证,123123123是你的服务器ip.
#会有一个让你输入yes的对话，在输入后
#不需要输入密码，即说明这个免密码设置已经成功！！
ssh -t git@123.123.123
```

- `git pull` 和`git fetch`区别在哪里？(为什么我要写这个？因为我因为一大堆git错误折腾了好久
  - **commit-id**:在每次本地工作完成后，都会做一个`git commit `操作来保存当前工作到本地的`repo`， 此时会产生一个`commit-id`，这是一个能唯一标识一个版本的序列号。 在使用`git push`后，这个序列号还会同步到远程仓库。
  - **FETCH_HEAD**: 版本链接，记录在本地的一个文件中，指向着目前已经从远程仓库取下来的分支的末端版本
  - **git fetch**：这将更新git remote 中所有的远程仓库所包含分支的最新`commit-id`, 将其记录到`.git/FETCH_HEAD`文件中  .
  - **git pull** : 基于本地`FETCH_HEAD`记录，比对本地的`FETCH_HEAD`记录与远程仓库的版本号，`git fetch` 获得当前指向的远程分支的后续版本的数据，利用`git merge`将其与本地的当前分支合并。所以可以认为`git pull`是`git fetch`和`git merge`两个步骤的结合。  
- 建立git服务器

 ```shell
mkdir  -p /home/git/repo/www.git
cd /home/git/repo/www.git
#初始化
git init --bare
cd /hooks
vim post-receive
--------------------------
#!/bin/sh
git --work-tree=/www --git-dir=/home/git/repo.git checkout -f
--------------------------
chmod +x post-receive

 ```

设置完登录部分就可以对`/www` 目录进行配置 

```shell
git init 
#/repo 这里就是你的git路径
git remote add orgin /home/git/repo/www.git
git fetch
#切换到主分支
git checkout master
```



此时，你的git地址为

```shell
git@123.123.123.123:/home/git/repo/www.git
#在你的计算机上进行操作
git clone git@123.123.123.123:/home/git/repo/www.git
touch README 
git commit -m "go!"
git push
```

出现了`error: unable to create file README (Permission denied)`

这个解决的话，我找了下 ，最后还是用最暴力的方式（总觉得这个方法不好，但是找不到其他方案）

```shell
sudo chmod 777 /wwww
```





至此，网站便可以顺利运行了。（我终于可以关掉一堆网页了！







参考资料:

vultr的centos 7创建jekyll教程

https://www.vultr.com/docs/creating-a-jekyll-blog-on-centos-7

生成https签名有两种方式

https://certbot.eff.org/lets-encrypt/centosrhel7-nginx

配置git-hook

https://ourai.ws/posts/deployment-with-git-hooks/

git pull 于fetch的区别

https://blog.csdn.net/riddle1981/article/details/74938111 
