---
title: nginx cache和gzip
date: 2018-07-06 21:40:15
---

昨天一个朋友做的商城项目上线，帮助他配置了一下nginx的gzip和cache，在这个过程中遇到的问题记录如下
<!-- more -->
### gzip

gzip用于静态文件(js、css、html)的压缩，一般可以压缩掉当前文件的2/3，加快页面渲染的速度，节省带宽

```
// ssh 登陆进入远程服务器，然后编辑nginx配置文件
vi /usr/local/nginx/conf/nginx.conf
// 打开gzip
{
  gzip on;
}
// reload nginx
/usr/local/nginx/sbin/nginx -t
/usr/local/nginx/sbin/nginx -s reload
```
配置完重启nginx以后，验证

```
curl -I -H "Accept-Encoding: gzip, deflate" "http://www.sinker.club"

HTTP/1.1 200 OK
Server: nginx/1.13.10
Date: Thu, 05 Jul 2018 05:48:56 GMT
Content-Type: text/html
Last-Modified: Wed, 04 Jul 2018 14:38:12 GMT
Connection: keep-alive
Vary: Accept-Encoding
ETag: W/"5b3cdbd4-5e50"
```
发现没效果，经过查资料发现gzip开启还需要配置一些其他的字段

```
gzip on;  // 开启gzip
gzip_min_length 1k; // 不压缩临界值，大于1K的才压缩，一般不用改
gzip_buffers 4 16k;
gzip_comp_level 2; 压缩级别，1-10，数字越大压缩的越好
gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php; // 进行压缩的文件类型
gzip_vary off;  // on的话会在Header里增加"Vary: Accept-Encoding"
```

重启配置完成以后继续执行curl命令，输出如下

```
Server: nginx/1.13.10
Date: Thu, 05 Jul 2018 05:48:56 GMT
Content-Type: text/html
Last-Modified: Wed, 04 Jul 2018 14:38:12 GMT
Connection: keep-alive
Vary: Accept-Encoding
ETag: W/"5b3cdbd4-5e50"
Cache-Control: max-age=259200
Content-Encoding: gzip
```
可以发现Content-Encoding: gzip 代表配置成功

### cache

接着我们配置缓存，配置之前我发现一个问题，我本来是一个默认的nginx配置文件，里面没有设置任何和缓存有关的东西，但是js和css都走了强缓存(from disk cache)，这个让我特别费解，后来查了很多文档，才发现cache-control默认就是private，当private的时候该文件就会被浏览器缓存到本地（并且是强缓存），但是具体过期时间文档中没有表明（我个人感觉是永远）

```
server {
  listen      80;
  server_name www.sinker.club;
  #charset koi8-r;

  #access_log  logs/host.access.log  main;
  root   /www/blog;
  index  index.html index.htm;
  location ~ .*\.(?:htm|html)$ {
    add_header Cache-Control "private, no-cache";
  }
  location ~ .*\.(?:js|css)$ {
    add_header Cache-Control "max-age=259200";
  }
  location ~ .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|  svgz|mp4|ogg|ogv|webm)$ {
    add_header Cache-Control "max-age=315360000";
  }
}
```
按照上述配置完成，然后重启nginx以后会发现html status 304、js css status from disk cache

这个时候我想测试一下，如果给html配置成强缓存可以吗？把上述配置文件修改一下

```
location ~ .*\.(?:htm|html)$ {
  add_header Cache-Control "max-age=30000";
}
```

reload以后查看，发现html还是走的304，按照上述配置js、css都可以为什么html不可以呢？查了一下论坛这个是因为chrome浏览器做的限制，不仅chrome、Safari、firfox等html文件都不会走强缓存，要不就是协商缓存，要不直接就是每次都发一次http请求。

即便html是走304协商缓存，但是建议大家还是在nginx里面给html设置上no-cache或者no-store。


#### 协商缓存

它的表现状态是http status为304，它需要每一次都发一次http请求去服务端验证该文件是否有更新，根据头部信息Last-Modified/If-Modified-Since、etag等做验证，如果没有更新重定向到本地读取本地缓存，它还有另外一个名字验证性缓存

#### 强缓存

它的表现状态为status: 200，size: from dist cache。它就比较霸道了，只要缓存到本地就不会和服务器交互，只有在你设置的max-age过期以后才会发送一次请求验证是否有最新的文件内容需要下载。如果想要设置强缓存的话，只需要给这个文件响应头加上cache-control: max-age=300（缓存时间自己决定）


### 其他

1. gzip一般不建议压缩图片，图片资源一般都是比较多如果压缩会占用服务器太多的资源，即便开启gzip其实对图片压缩的效果也特别小，基本上没有效果
2. 缓存时间的话，建议js、css一个月即可，图片为1年

### 一个异常场景
有一个场景是这样的：比如我项目上线好长时间了，有一些app已经访问过该项目，但是发现html没有配置不缓存，导致我们后期push的一些代码用户没有更新到（如果html缓存到客户端，即便后期html内的js的hash值或者版本号改变客户端也不会下载到，因为用户一直都在走老的html文件），这个时候我们再给nginx配置上html不缓存的话，后面一些新用户是不会出现上述问题了，但是在配置不缓存之前已经访问过的用户，即便你这次更改了也影响不到他们，他们还是会走老的html代码，这个时候只能让用户强制清除浏览器或者app缓存了


### 总结

1. gzip和缓存是我们前端做性能优化最重要的两个点，以前都是面试的时候简单背一点，或者在公司直接找运维配置，导致很多点都不是特别明白，人云亦云，趁着这次机会也算是梳理了一下，写篇文章mark一下
2. 我接触的场景用到协商的不多，大部分都是不缓存或者强缓存


