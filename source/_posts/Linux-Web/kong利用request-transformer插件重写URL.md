---
title: kong利用request-transformer插件重写URL
date: 2020-09-10 12:59:58
tags: kong
categories: [Linux-Web, kong]
comments: true
copyright: true
---

## kong利用request-transformer插件重写URL

### 介绍

最近业务整合后有一个需求,将URL:www.abc.com/api/item/111 想重写成www.xyz.com/open/item/itemdetail?id=111,并且域名不变,不能发生302跳转.

使用Nginx的rewrite redirect指令可以实现URL重写需求,但是redirect会跳转到新域名,不符合需求.

刚好该业务的的前端是用Kong网关处理,所以研究kong的插件实现这个需求

---

### request-transformer介绍

**request-transformer**是Kong官方的插件,允许修改重写用户的请求,还可以使用正则表达式匹配URL,并将匹配到的字符串保存在变量中,然后使用模板将变量转换成用户的请求

简而言之:**就是重写用户的请求**,包括URL,args,headers,methods等等

官方地址: [reuqest-transformer官方地址](https://docs.konghq.com/hub/kong-inc/request-transformer/)

github项目地址: [request-transformer github](https://github.com/Kong/kong-plugin-request-transformer)

<!--more-->

---

### 配置方法

> kong使用的是2.1.3最新版本,试过使用Kong1.0版本插件无法生效

这里举2个例子说明

* 将URL:/v4/jkf/branch/qrcode&code=100006 重写为 /v4/jkf/branch/qrcode?code=100006.也就是将`&`转换为`?`

  - 首先配置Service和Route.具体配置方法略过,这里主要关心一下Route中的Path设置:

  ```
  /v4/jkf/branch/qrcode\&code=(?<code>\d+)
  ```

  该PATH表示:

  1.匹配/v4/jkf/branch/qrcode&code=任意长度数字.

  2.正则`\d`表示匹配数字,并且将匹配到的数字保存为`code`变量

  3.`\&`表示转义URL中的`$`符号

  > 关闭route中的script path可选项

  * 其次在该route下添加`request-transformer`插件,表示该插件只应用到此条route下.并且配置插件参数

    ```
    curl -X POST http://localhost:8001/routes/21292565-e7ae-40ff-a465-f449c16b7819/plugins \
          --data "name=request-transformer" \
          --data "config.replace.uri=/jkf/branch/qrcode" \
          --data "config.add.querystring=code:\$(uri_captures.code)"
    ```

    对于上面的命令行解释如下

    1. `21292565-e7ae-40ff-a465-f449c16b7819`就是刚才创建的路由ID
    2. `config.replace.uri`表示将route匹配到的PATH重写为`/jkf/branch/qrcode`
    3. `onfig.add.querystring`表示给URL添加args参数
    4. `code:\$(uri_captures.code)`.参数名是`code`,`uri_captures.code`表示获取route的PATH中code变量的值,由于命令行shell环境关系,所以要在变量符号`$`前进行转义.

    

    或者也可以使用konga的UI管理平台添加和编辑插件

![](https://img2.jesse.top/image-20200910160826719.png)

  

  

  ![image-20200910160949836](https://img2.jesse.top/image-20200910160949836.png)

  - 最后,`reload`Kong进程

  ```
  [root@docker ~]# docker exec kong kong reload
  ```

  **验证**

  本地访问网站:

  ```
   ✘ huangyong@huangyong-Macbook-Pro  ~  curl -XGET https://m.devapi.xxx.com/v4/jkf/branch/qrcode\&code\=100006
  ```

  Kong和后端nginx日志如下:

  ```
  172.18.0.1 - - [10/Sep/2020:16:52:11 +0800] "GET /jkf/branch/qrcode?code=100006 HTTP/1.1" 200 163 "-" "curl/7.54.0" "10.0.4.9, 172.18.0.2" 10.0.4.9, 172.18.0.2, 172.18.0.1 "2b7dcdc621d1928f456d561f31e95b25"0.065 0.065
  ```

  可以看到已经成功重写了URL

  ---

* 第二个例子,将/api/item/111 重写为/open/item/itemdetail?id=111

将URL后面的数字拼接到id的值,作为参数拼接成URL后,传递给后端

1.添加Service和Routes,Routes的PATH部分如下:

```
#该PATH只匹配/api/item/数字格式的URL.并且将\d+正则匹配到的数字保存为变量id
/api/item/(?<id>\d+)$
```

2.添加和配置插件

```
curl -X POST http://localhost:8001/routes/38fce1b7-2a36-42cb-9619-f30539889137/plugins \
      --data "name=request-transformer" \
      --data "config.replace.uri=/open/item/itemdetail" \
      --data "config.add.querystring=id:\$(uri_captures.id)"
```

3.重载kong

```
[root@docker ~]# docker exec kong kong reload
```

4.验证

```
 huangyong@huangyong-Macbook-Pro  ~  curl -XGET https://m.devapi.xxx.com/api/item/111
```

后端日志如下

```
172.18.0.1 - - [10/Sep/2020:17:11:43 +0800] "GET /open/item/itemdetail?id=111 HTTP/1.1" 200 160 "-" "curl/7.54.0" "10.0.4.9, 172.18.0.2" 10.0.4.9, 172.18.0.2, 172.18.0.1 "54ac589d36f90c1ef99ba6a43c4d488e"0.105 0.105
```



