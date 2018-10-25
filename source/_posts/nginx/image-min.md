---
title: centos图片压缩配置方案
date: 2018-08-07 17:22:15
---

> 本方案是在webpack构建流程中新增的图片压缩，利用的是image-webpack-loader来处理的，测试服务器环境已经配置好，只需要各位同学安装loader和更新webpack.base.conf.js即可

<!-- more -->

### 安装loader

```
// 建议锁死版本3.6.0，高版本在某些情况会有问题
npm install image-webpack-loader@3.6.0 --save-dev
```
把下面的配置替换 webpack.base.conf.js内的url-loader的地方
```
{
    test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
    use: [
      `url-loader?name=${utils.assetsPath("img/[name].[hash:7].[ext]")}&limit=10000`,
      {
        loader: "image-webpack-loader",
        options: {
          mozjpeg: {
            progressive: false,
            quality: 80
          },
          // optipng.enabled: false will disable optipng
          optipng: {
            enabled: false
          },
          pngquant: {
            quality: "65-90",
            speed: 4
          },
          gifsicle: {
            interlaced: false
          }
        }
      }
    ]
 },
```

### 服务器配置（centos6.x）
按照上述配置完以后，在mac端执行npm run build的时候是没问题，但是到centos服务器的话就会报错，是因为image-webpack-loader压缩图片的话本质上是依赖于服务器底层的libpng-devel，centos默认是没有的所以需要手动安装一下

```
yum install libpng-devel
```
安装完libpng-devel以后还需要更新一下glibc，图片压缩需要该库的版本最低为2.14但是centos6.x默认版本为2.12所以需要手动更新一下，按照下述命令依次执行

```
wget http://ftp.gnu.org/gnu/glibc/glibc-2.15.tar.gz
 
wget http://ftp.gnu.org/gnu/glibc/glibc-ports-2.15.tar.gz
 
tar -xvf  glibc-2.15.tar.gz
 
tar -xvf  glibc-ports-2.15.tar.gz
 
mv glibc-ports-2.15 glibc-2.15/ports
 
mkdir glibc-build-2.15 
 
 
cd glibc-build-2.15
 
 
../glibc-2.15/configure  --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
 
 
make
make install
```

glibc更新到2.15以后，重新执行npm run build即可打包成功

### 服务器配置（centos7.x）

```
// 安装libpng-devel
https://centos.pkgs.org/6/centos-x86_64/libpng-devel-1.2.49-2.el6_7.x86_64.rpm.html
```
download下来以后在服务器上安装rpm文件，然后执行npm run build

### 备注
如果后续有压缩webp格式的需求，需要在服务器上再安装其他的第三方包
