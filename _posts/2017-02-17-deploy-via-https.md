---
layout: post
title: 博客的迁移及自动化部署并全站https化
tags: https
categories: Others
date: 2017-02-17
---
过完年来想把博客做一个迁移，放到自己购买的服务器上，并实现自动化部署，并启用全站HTTPS

## hexo本地部署
这一步骤网上有很多教程，这里不再多说了

## 服务器自动化部署
大体的流程就是，我们通过`hexo g`命令在本地生成静态文件以后，通过git push到我们的远程仓库(这里我用的是GitHub)，然后由于我们事先在项目库中配置了webhooks，由它post到你的服务器一个请求链接，我们的服务器收到请求后，对应执行我们提前写好的脚本，再将push的内容同步到我们的服务器，从而更新了服务器的内容。

## 服务器环境配置
我们通过ssh(windows用户可以通过putty登录)登录到我们的服务，我这里用的是Ubuntu系统，安装好nodejs,git,nginx后，将我们的文件从远程仓库拉下来
```
mkdir blog
cd blog
git init
git remote add origin https://github.com/yuxingxin/yuxingxin.github.io.git
git pull origin master
```

### 配置nginx
这个配置之前写的有相关文章，不明白的可以看[这里](http://www.jianshu.com/p/3531a011b7b6)，不过这里的系统是Ubuntu，所以nginx的安装路径也不太一样(`/etc/nginx`)，默认我们需要在`/etc/nginx/conf.d/`目录下添加配置文件`blog.conf`，**注意这里的后缀名一定是`.conf`**
```
vim /etc/nginx/conf.d/blog.conf
```
然后配置上我们的域名，端口和映射地址
```
server {
    listen       80;  #修改这里为其他端口如8081
    server_name  yuxingxin.com www.yuxingxin.com; # 这里是你的域名
    location / {
        root   /root/blog/; #修改这里的路径为自己的路径
        index  index.html index.htm;
    }
}
```
然后重启nginx通过域名就可以访问我们的博客了
```
nginx -s reload
```

### webhooks配置
也就是人们常说的钩子，是一个很有用的工具。你可以通过定制 Webhook 来监测你在 Github.com 上的各种事件，最常见的莫过于 push 事件,如果你设置了一个监测 push 事件的 Webhook，那么每当你的这个项目有了任何提交，这个 Webhook 都会被触发，这时 Github 就会发送一个 HTTP POST 请求到你配置好的地址。如此一来，你就可以通过这种方式去自动完成一些重复性工作；比如，你可以用 Webhook 来自动触发一些持续集成（CI）工具的运作，比如 Travis CI；又或者是通过 Webhook 去部署你的线上服务器。
Webhook 的配置是十分简单的。首先进入你的 repo 主页，通过点击页面上的按钮 [settings] -> [Webhooks & service] 进入 Webhooks 配置主页面。在Payload URL配置链接，比如：
```
http://xxxxx.com:8246/webhooks/push/deploy
```
这样一来，仓库的配置就算好了，接下来看服务器端如何响应
### 服务器配置webhooks响应
首先我们在我们的博客目录下创建一个js文件，用来启动我们的监听服务，端口就是我们在仓库配置那里的端口地址：8246
```
var http = require('http')
var exec = require('child_process').exec
http.createServer(function (req, res) {
    if(req.url === '/webhooks/push/deploy'){
        exec('sh ./deploy.sh',function(error,stdout,stderr){
	     if(error){
		  console.log(error.stack);
		  console.log('Error code:'+ error.code);
	     }else{
	          console.log('success');
	     }
	})
    }
    res.end()
}).listen(8246)
```
这段代码就能启动一个nodejs服务，监听8246端口，当请求过来的url完全匹配的时候，执行deploy.sh。
再新建一个deploy.sh文件处理部署相关脚本：
```
git pull origin master
```
### 运行nodejs服务
```
node ./webhooks.js
```
如果你使用上面的命令运行nodejs服务，nodejs服务会在前台运行，可以使用[pm2](https://www.npmjs.com/package/pm2)使nodejs运行在后台。
```
# 安装
npm install pm2 -g

# 启动服务
pm2 start webhooks.js

# 停止服务
pm2 stop webhooks.js

# 重启服务
pm2 restart webhooks.js

# 实时查看pm2的日志服务
pm2 logs
```
到此为止我们的自动化部署就全部完成了，以后我们只需在本地将文件push到远程仓库，就会自动同步到我们的服务器上

## 启用全站HTTPS
这里简单总结下在 Nginx 配置 HTTPS 服务器：主要签署第三方可信任的证书和配置HTTPS

### 关于证书
SSL证书需要向国际公认的证书证书认证机构（简称CA，Certificate Authority）申请。
CA机构颁发的证书有3种类型：
1. 域名型SSL证书（DV SSL）：信任等级普通，只需验证网站的真实性便可颁发证书保护网站；
2. 企业型SSL证书（OV SSL）：信任等级强，须要验证企业的身份，审核严格，安全性更高；
3. 增强型SSL证书（EV SSL）：信任等级最高，一般用于银行证券等金融机构，审核严格，安全性最高，同时可以激活绿色网址栏。

关于证书服务，市面上大体的都是收费的证书，当然也有部分是免费的，比如Let's Encrypt  刚刚又拍云也上线了免费的 SSL [证书](http://weibo.com/ttarticle/p/show?id=2309404057245016043005#_0)，另外StartSSL也提供免费证书，有效期3年；另外还有腾讯云和阿里云都有相关的免费证书，这里我使用的是Let's Encrypt ,这也是目前最知名的开源SSL证书。

### 证书申请
申请 Let's Encrypt 证书不但免费，还非常简单，虽然每次只有 90 天的有效期，但可以通过脚本定期更新，配好之后一劳永逸。Let's Encrypt 的证书签发过程使用的就是 ACME 协议,这里也推荐一个小工具就是[acme-tiny](https://github.com/diafygi/acme-tiny),它可以帮助我们简化创建证书的流程。
#### 创建帐号
创建一个目录存放私钥证书等各种文件，然后进入后创建我们的RSA账户私钥
```
mkdir ssl
cd ssl
openssl genrsa 4096 > account.key
```
#### 创建 CSR 文件
这一步生成我们的证书签名文件，即CSR，首先要创建RSA私钥
```
openssl genrsa 4096 > domain.key
```
接下来就可以生成我们的证书文件了，单个域名和多个域名生产的参数还不太一样.
```
//单个域名
openssl req -new -sha256 -key domain.key -subj "/CN=yuxingxin.com" > domain.csr
//多个域名（比如yuxingxin.com和www.yuxingxin.com）
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yuxingxin.com,DNS:www.yuxingxin.com")) > domain.csr
```
#### 配置验证服务
CA 在签发 DV（Domain Validation）证书时，需要验证域名所有权，而大部分都是通过邮件验证的方式，Let's Encrypt 则是在你的服务器上生成一个随机验证文件，再通过创建 CSR 时指定的域名访问，如果可以访问则表明你对这个域名有控制权。
首先创建用于存放验证文件的目录
```
mkdir ~/www/challenges/
```
然后配置一个 HTTP 服务，以 Nginx 为例：
```
server {
    listen       80;
    # 这里改成自己的域名
    server_name yuxingxin.com www.yuxingxin.com;

    location /.well-known/acme-challenge/ {
        alias /root/www/challenges/;
        try_files $uri =404;
    }
}
```
#### 生成网站证书
先把acme-tiny脚本保存到之前的ssl目录
```
wget https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py
```
指定账户私钥、CSR 以及验证目录，在ssl目录下执行脚本
```
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /root/www/challenges/ > ./signed.crt
```
如果一切正常，当前目录ssl下就会生成一个 signed.crt，这就是申请好的证书文件。
这里遇到一个错误,大致如下：
```
ValueError: Wrote file to /root/www/challenges/oJbvpIhkwkBGBAQUklWJXyC8VbWAdQqlgpwUJkgC1Vg, but couldn't download http://www.yuxingxin.com/.well-known/acme-challenge/oJbvpIhkwkBGBAQUklWJXyC8VbWAdQqlgpwUJkgC1Vg
```
网上查了一些方案，觉得有一个比较靠谱，也得到了解决，大致就是原来库做了一个验证导致有些情况通不过，这里有人fork源库修改了部分代码，地址在[这里](https://github.com/frezbo/acme-tiny)，如果出现上述错误，可以获取这个库的脚步然后在执行上面那条命令。
#### 下载中间证书
```
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
```
#### 为了后续能顺利启用 OCSP Stapling，我们再把根证书和中间证书合在一起：
```
wget -O - https://letsencrypt.org/certs/isrgrootx1.pem > root.pem
cat intermediate.pem root.pem > full_chained.pem
```
#### 修改nginx证书配置
```
server {
    listen 443;
    #修改成自己的域名
    server_name yuxingxin.com www.yuxingxin.com;

    ssl on;
    #这里注意证书路径
    ssl_certificate /root/ssl/chained.pem;
    ssl_certificate_key /root/ssl/domain.key;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /root/ssl/dhparam.pem;
    ssl_prefer_server_ciphers on;
    resolver 8.8.8.8;
    ssl_stapling on;
    ssl_trusted_certificate /root/ssl/signed.crt;
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;preload";

    location / {
        # 这里要改成自己存放博客静态网页的目录
        root  /root/blog;
        index  index.html index.htm;
    }
}
server {
    listen       80;
    # 这里改成自己的域名
    server_name yuxingxin.com www.yuxingxin.com;
    ssl_certificate /root/ssl/chained.pem;
    ssl_certificate_key /root/ssl/domain.key;

    location / {
        return 301 https://$host$request_uri;
    }

    location /.well-known/acme-challenge/ {
        alias /root/www/challenges/;
        try_files $uri =404;
    }

    #error_page  404              /404.html;
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```
### 配置自动更新
Let's Encrypt 签发的证书只有 90 天有效期，推荐使用脚本定期更新
创建脚本文件并赋予执行权限
```
mkdir shell
cd shell
vi renew_cert.sh
```
并复制下面内容
```
#!/bin/bash
cd /root/ssl/
python acme_tiny.py --account-key account.key --csr domain.csr --acme-dir /root/www/challenges/ > signed.crt || exit
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
service nginx reload
```
这里借助crontab来定时执行任务，它是一个可以用来根据时间、日期、月份、星期的组合来调度对重复任务的执行的守护进程。
在终端执行：
```
crontab -e
```
然后添加如下内容：
```
0 0 1 * * /root/shell/renew_cert.sh >/dev/null 2>&1
```
这样以后证书每个月都会自动更新，一劳永逸。
另外这里也推荐一个网站，可以监测你的证书的有效期[https://letsmonitor.org](https://letsmonitor.org/)
这样的话就算完了，但是有几点需要注意下：
1. dhparam的生成
```
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```
2. 强制HTTPS
```
#另外还有两种其他的配置方式，可以自行Google
 location / {
        return 301 https://$host$request_uri;
    }
```
3. 关于HSTS
HTTP Strict Transport Security的缩写，即：“HTTP严格传输安全”。假设一个用户从来没有访问过我的网站，并且他第一次访问的时候访问的是 http://yuxingxin.com ，在正常的情况下，我的服务器就会给这位用户返回一个 301 跳转到 https://yuxingxin.com ，并且带上上面配置的HSTS头，在用户下次访问我的博客时，只要 HSTS 还在有效期中，浏览器就会直接跳转到相对应的 https 页面，并且这是不需要经过数据传输的，直接在本地浏览器进行的处理。
目前大部分浏览器对HSTS的支持已经相当完美，具体各浏览器和版本的支持情况可以在[这里](http://caniuse.com/#search=HSTS)上查看。 但是HSTS是有缺陷的，第一次访问网站的客户端，HSTS并不工作。 要解决这个问题，就需要我们下面要讲解的HSTS preload list。它是一个站点的列表，并通过硬编码写入 Chrome 浏览器中，列表中的站点将会默认使用 HTTPS 进行访问，此外，Firefox 、Safari 、IE 11 和 Edge 也同样用一份 HSTS 站点列表，它申请加入需要一些条件：
4. 有一张有效的证书（如果是使用了 SHA-1 证书签名算法的必须在 2016 年前失效）
5. 重定向所有的 HTTP 流量到 HTTPS （ HTTPS ONLY ）
6. 全部子域名的流量均通过 HTTPS ，如果子域名的 www 存在的话也同样需要通过 HTTPS 传输。
* 在相应的域名中输出 HSTS 响应头
1 过期时间至少大于 18 周（10886400 秒）
2 必须声明 includeSubdomains
3 必须声明 preload
4 跳转过去的那个页面也需要有 HSTS 头
点击[这里](https://hstspreload.org/)开始申请，申请成功后，你的域名就会加入到[这个列表](https://code.google.com/p/chromium/codesearch#chromium/src/net/http/transport*security*state_static.json)
![ssl.png](http://upload-images.jianshu.io/upload_images/450566-d5d44b46903e8552.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果大多数浏览器都已经更新到新的列表，那么针对国内的 VPS ，不打开80端口，只打开 443 ，浏览器同样会跳转过来，这样就可以免备案了，不过好像这样对搜索引擎就不太友好。

## 测试
所有这些工作完成以后，我们可以对证书进行检测。这里是[检测地址](https://www.ssllabs.com/ssltest/index.html) ，下面是我的域名的检测报告。
![report.jpeg](http://upload-images.jianshu.io/upload_images/450566-082a2a529b747654.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果是A+,则说明你的配置是好着的。另外这里也给出一个国内的[检测网站](https://www.chinassl.net/ssltools/ssl-checker.html)，除了以上这些步骤，可能觉得有些繁琐，就有人写了一个[脚本](https://github.com/xdtianyu/scripts/blob/master/lets-encrypt/README-CN.md),它是一个快速获取/更新 Let's encrypt 证书的 shell script，使用该脚本可以简化上面的流程。
