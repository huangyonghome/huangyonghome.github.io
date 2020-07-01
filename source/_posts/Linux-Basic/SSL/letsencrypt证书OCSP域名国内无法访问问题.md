---
title: Letsencrypt证书OCSP域名国内无法访问问题
date: 2020-07-01 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true
---

## Letsencrypt证书OCSP域名国内无法访问问题

### 背景

近期公司线上APP,小程序的h5页面在IOS移动设备上加载速度非常慢.安卓设备不受影响.调试显示 TCP SSL 相关时间在3s以上.

---

### 原因

经排查发现是国内Let's Encrypt免费SSL证书机构的OSCP域名 http://ocsp.int-x3.letsencrypt.org的CNAME解析在国内被DNS污染..安卓和chrome浏览器默认不在客户端检测OCSP,所以没有影响.但是移动IOS设备和Safari浏览器则会出现这种故障.

参考V2EX的论坛:https://www.v2ex.com/t/658605

使用国内的公众DNS服务器解析该域名结果:

```
 huangyong@huangyong-Macbook-Pro  ~  dig @114.114.114.114 ocsp.int-x3.letsencrypt.org

....略....

;; ANSWER SECTION:
ocsp.int-x3.letsencrypt.org. 31	IN	CNAME	ocsp.int-x3.letsencrypt.org.edgesuite.net.
ocsp.int-x3.letsencrypt.org.edgesuite.net. 31 IN CNAME a771.dscq.akamai.net.
a771.dscq.akamai.net.	31	IN	A	31.13.69.86

;; Query time: 15 msec
;; SERVER: 114.114.114.114#53(114.114.114.114)
;; WHEN: Wed Jul 01 16:40:14 CST 2020
;; MSG SIZE  rcvd: 158
```

这个`a771.dscq.akamai.net` 的CNAME域名解析的IP地址并不是akamai的IP地址.也无法访问这个iP

```
 huangyong@huangyong-Macbook-Pro  ~  ping 31.13.69.86
PING 31.13.69.86 (31.13.69.86): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
```

<!--more-->

正确的IP地址应该是:

```
 huangyong@huangyong-Macbook-Pro  ~  dig @8.8.8.8 ocsp.int-x3.letsencrypt.org

;; ANSWER SECTION:
ocsp.int-x3.letsencrypt.org. 4085 IN	CNAME	ocsp.int-x3.letsencrypt.org.edgesuite.net.
ocsp.int-x3.letsencrypt.org.edgesuite.net. 1783	IN CNAME a771.dscq.akamai.net.
a771.dscq.akamai.net.	19	IN	A	23.210.215.97
a771.dscq.akamai.net.	19	IN	A	23.210.215.80
```

---

### 解决办法

* 更换证书----使用阿里云的免费DV域名证书(赛门铁克)没有这个问题
* 服务器做ocsp stapling.让ocscp请求通过自己的服务器. (这个办法我尝试过很久都没有成功)
* 由于只是DNS被污染,.所以可以在nginx服务器绑定hosts.解析到正确的IP地址上.(我暂时使用这个办法)

---

### 服务器做ocsp stapling

百度上有大量的相关配置文档,这里列举3种:

**第一种**: 获取OCSP响应,配置ssl_stapling_file.点击查看[文档地址](https://holmesian.org/letsencrypt-ocsp-fix)

我在这一步的时候报错:

```
 openssl ocsp -no_nonce \
                 -respout /root/.acme.sh/meta.bi.doweidu.com/ocsp_res.der \
                 -issuer /root/.acme.sh/meta.bi.doweidu.com/ca.cer \
                 -cert /root/.acme.sh/meta.bi.doweidu.com/meta.bi.doweidu.com.cer \
                 -url http://ocsp.int-x3.letsencrypt.org/ \
                 -header "HOST" "ocsp.int-x3.letsencrypt.org"
```

报错内容如下,去掉`-header "HOST" "ocsp.int-x3.letsencrypt.org` 仍然会报错

```
Response Verify Failure
139961608943504:error:27069076:OCSP routines:OCSP_basic_verify:signer certificate not found:ocsp_vfy.c:92:
/root/.acme.sh/meta.bi.doweidu.com/meta.bi.doweidu.com.cer: good
	This Update: Jun 28 06:00:00 2020 GMT
	Next Update: Jul  5 06:00:00 2020 GMT
```

**第二种**: 配置根证书,中间证书文件.配置`ssl_trusted_certificate`: [文档地址](https://www.songhaifeng.com/razt/193.html)

但是我检测OCSP的时候,仍然提示`OCSP response: no response sent`:

```
openssl s_client -connect meta.bi.doweidu.com:443 -status -tlsextdebug < /dev/null 2>&1 | grep -i "OCSP response"
OCSP response: no response sent
```

**第三种**:直接配置nginx.文档地址: [文档地址](https://juejin.im/post/5ea12732f265da47d12933c5)

```
# 在Nginx的虚拟主机配置文件中加上以下几行

server {

......
ssl_stapling on;
ssl_stapling_verify on;
resolver 172.16.20.30;  #内部DNS服务器,配置了正确的解析
......
}
```

经过测试,这种方法也不起作用

我在Let's Encrypt官方论坛提交了一个topic,也没有收到有用的帮助信息:

https://community.letsencrypt.org/t/how-to-enable-ocsp-on-nginx-please/127288

---

### 临时解决方案

由于只是DNS收到污染,TCP链接和网络层面并没有被彻底封掉.所以只是简单在nginx的服务器上绑定hosts:

```
23.192.45.96 ocsp.int-x3.letsencrypt.org
23.192.45.96 a771.dscq.akamai.net
```

---

### 总结

还需要再观察一下该问题.目前尚不清楚是Akamai的整个CDN域名被污染,还是仅仅是Let's Entrypt的域名被污染.甚至或者是伟大的GFW原因.



