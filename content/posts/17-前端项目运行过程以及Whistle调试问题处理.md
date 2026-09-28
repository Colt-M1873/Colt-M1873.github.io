---
title: "前端项目运行过程以及Whistle调试问题处理"
date: 2023-08-10T17:26:43+08:00
draft: false
ShowToc: true
TocOpen: false
---

## 前端项目运行过程

新拿到一个Vue项目，我们需要 nodejs环境、vue-cli构建工具和npm的墙内镜像。

首先安装node，安装完后命令行执行`npm -v`有结果即表明安装成功。

命令行执行以下代码设置npm代理为华为镜像
```bat
npm config rm proxy
npm config rm http-proxy
npm config rm https-proxy
npm config set no-proxy .huawei.com
npm config set registry http://cmc-cd-mirror.rnd.huawei.com/npm
```
安装vue-cli和webpack
```bat
npm install -g vue-cli
npm install webpack -g
```
然后在工程的vue项目目录(一般是vue.config.js文件存放的位置)下安装项目依赖，即执行：

```bat
npm install
```
随后

```bat
npm run dev
```
理论上就能将前端项目跑起来。

## whistle的使用

由于项目依赖可能较为复杂，很多时候也会用到通过直接拦截服务器的请求、将服务器资源替换为本地项目资源来进行调试的方式：
这时候就要用到**whistle**和**SwitchyOmega**
通过配置代理来显示本地前端资源，以达到测试前端项目的目的。

SwitchyOmega是一个chrome插件，直接商店搜索下载即可。

安装whistle执行`npm install -g whistle`即可。

随后在cmd中执行`w2 start`，如果有显示即表明whistle安装成功并且正在运行，默认的运行端口是8899，本地访问` http://127.0.0.1:8899/`即可打开whistle


在whistle的rule设置中写入规则，将服务器上的前端工程地址，替换成你本地的工程地址，以使用本地资源替换前端资源。

举例如下，其中`C:\path\to\xxWeb\xxweb\dist` 即是你执行 `npm run build`构建项目的地址
```whistle rules
$https://*.xxxxxxxxx.com/xxxweb/  file://C:\path\to\xxWeb\xxweb\dist/index.html includeFilter://m:GET  
https://*.xxxxxxxxx.com/xxxweb/js/  file://C:\path\to\xxWeb\xxweb\dist/js/
https://*.xxxxxxxxx.com/xxxweb/img/  file://C:\path\to\xxWeb\xxweb\dist/img/
https://*.xxxxxxxxx.com/xxxweb/css/  file://C:\path\to\xxWeb\xxweb\dist/css/
https://*.xxxxxxxxx.com/xxxweb/static/fonts/  file://C:\path\to\xxWeb\xxweb\dist/fonts/
```

随后保存规则，将HTTPS设置中的 **Capture TUNNEL CONNECTS** 和 **Enable HTTP/2** 全部勾选。

而后在 SwitchOmega中新建情景模式
代理协议选HTTP，代理服务器和端口即是你本地跑whistle的端口，按照whistle默认设置则填写 `127.0.0.1`和`8899`

执行`npm run build`构建你的项目，构建完成后打开SwitchOmega代理，正常的话访问`https://*.xxxxxxxxx.com/xxxweb/`就能看到你已经修改过的内容。

可以从whistle的Network页面查看当前的抓包结果，点击即可查看某个请求是否经过了你在rules中设置的规则。

whistle常用命令
打开 `w2 start`
关闭 `w2 stop`
重启 `w2 restart`

在开着whistle的时候使用`npm run build`的话，很容易出现无权访问的错误
**即 EPERM: operation not permitted, lstat**
该错误是由于whistle正在监控项目构建的路径，正在使用其中的各个文件，因此执行`npm run build`会出现访问权限冲突。

在使用`npm run build:watch`进行实时持续构建的时候，使用whistle抓包也容易出现xxx资源无法获取的冲突，这两者都是由于whistle和npm同时访问了同一个文件导致的。

因此日常还是在执行`npm run build`之前关闭whistle以避免出错

## 参考
[npm install 出错:npm ERR! code ETIMEDOUT](http://3ms.huawei.com/km/blogs/details/9151503)
[【前端调试】JS调试常用技巧--微服务调试利器Whistle](http://3ms.huawei.com/km/blogs/details/8211185?l=zh-cn)
[whistle本地项目代理](http://3ms.huawei.com/hi/group/2692283/wiki_6563246.html)
