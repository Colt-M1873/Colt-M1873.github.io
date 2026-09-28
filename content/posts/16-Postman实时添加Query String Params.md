---
title: "Postman实时添加Query String Params"
date: 2023-08-10T16:21:23+08:00
draft: false
ShowToc: true
TocOpen: false
---

很多接口会以url string的形式传递信息，特别是GET请求的时候，因为不能带payload，查询的参数全靠Query String Params。

在Postman中添加固定的Query String Params很方便，只要在Query Params中一项一项添加key和value值就可以了

但有些接口会要求在url中携带一些实时生成的参数以进行校验：

例如接口要求在每次请求时都要携带 `_t=1691656231331` 这样的参数，即当前时间。

这样的操作一般是在网页的前端工程中，每次发送请求前使用`new Date().getTime()`给url加入当前时间。

为了在Postman中模拟这样的操作，要使用到Pre-request Script这一项，顾名思义即在每次请求前都会执行一次的脚本
![Pre-request Script](./images/1691656586695_image.png)

在Pre-request Script中写入：
```javascript
timestamp = new Date().getTime()
console.log(timestamp)
pm.environment.set("timestamp",timestamp)
```
即为Postman添加了名为timestamp的环境变量。该环境变量的作用和使用方式与 Postman Enviroments设置中的环境变量是一样的。

在Query Params中填入此环境变量即可实现每次发送当前时间作为`_t`值。
![](./images/1691656902685_image.png)

在每次运行后，点击左下角的Console即可查看到json运行的输出和请求响应的完整信息。


复习一下HTTP请求传递参数的三种形式：Query String Params、Request Payload和FormData


## 参考
[postman请求参数如何获取到当前时间](https://blog.csdn.net/weixin_41639638/article/details/121743607)
[postman使用教程3-全局(Global)变量和环境(Environment)变量的使用 ](https://www.cnblogs.com/yoyoketang/p/14736733.html)
