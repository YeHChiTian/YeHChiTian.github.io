---
title: libcurl使用指定IP访问http资源
date: 2019-08-21 19:03:19
categories: 
- libcurl
tags: dns域名解析
---

# 前言

>由于业务场景需要，服务器需要访问本地的http资源，希望通过本地地址访问而不需要对http链接进行域名解析，项目使用libcurl完成http资源的访问请求，因此希望通过libcurl库，指定IP地址访问特定的http资源。

# 1、修改hosts文件

最容易想到的是修改本地**hosts文件**，但是此种做法不够好，因为这样修改将影响整个服务器对此域名的解析，而我们只是想要对特定的进程使用特定的IP访问http资源而已。(linux下hosts文件位于/etc/hosts目录下)，修改如下。

```bash
127.0.1.1       example.com
```

修改hosts文件之前，结果如下,可见将example.com域名解析为地址**93.184.216.34**。

```
$ping -c 2 example.com
PING example.com (93.184.216.34) 56(84) bytes of data.
64 bytes from 93.184.216.34: icmp_seq=1 ttl=128 time=263 ms
64 bytes from 93.184.216.34: icmp_seq=2 ttl=128 time=263 ms

--- example.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 263.669/263.810/263.952/0.532 ms
```

修改hosts文件之后，结果如下,可见将example.com域名解析为地址**127.0.0.1**。

```
#ping -c 2 example.com
PING example.com (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.034 ms

--- example.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1016ms
rtt min/avg/max/mdev = 0.034/0.038/0.043/0.007 ms
```

# 2、使用libcurl特定选项CURLOPT_RESOLVE 

程序使用libcurl访问http资源，因此希望在libcurl中寻找支持此功能的配置。很巧，发现libcurl有对此功能支持。选项介绍如下，提供特定域名到IP地址的转换。

>CURLOPT_RESOLVE - provide custom host name to IP address resolves

具体使用可参考官方例子，如下所示。

官方介绍：https://curl.haxx.se/libcurl/c/CURLOPT_RESOLVE.html

```cpp
CURL *curl;
struct curl_slist *host = NULL;
host = curl_slist_append(NULL, "example.com:80:127.0.0.1");
 
curl = curl_easy_init();
if(curl) {
  curl_easy_setopt(curl, CURLOPT_RESOLVE, host);
  curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");
 
  curl_easy_perform(curl);
 
  /* always cleanup */
  curl_easy_cleanup(curl);
}
 
curl_slist_free_all(host);
```

如果想增加多个域名到IP地址转换，**注意写法**。

```cpp
	host = curl_slist_append(NULL, "www.baidu.com:80:192.168.113.128");
	host = curl_slist_append(host, "example.com:80:127.0.0.1");
	curl_easy_setopt(curl_handle, CURLOPT_RESOLVE, host);
```

测试结果(有删减)如下。在libcurl中增加如下选项，即可看到测试打印消息。

> curl_easy_setopt(curl_handle, CURLOPT_VERBOSE, 1L);

```
* Added www.baidu.com:80:192.168.113.128 to DNS cache
* Added example.com:80:127.0.0.1 to DNS cache

.................
* Connected to www.baidu.com (192.168.113.128) port 80 (#0)
> GET / HTTP/1.1
Host: www.baidu.com
User-Agent: 4399cdn-p2p
Accept: */


* Connected to example.com (127.0.0.1) port 80 (#1)
> GET / HTTP/1.1
Host: example.com
User-Agent: 4399cdn-p2p
Accept: */*
```

访问www.baidu.com的时候，使用指定的IP地址192.168.113.128访问。

访问http://example.com时候，使用指定的IP地址127.0.0.1访问。



note: dns缓存

linux的域名解析缓存默认是关闭，需要启用nscd 服务，才会有dns缓存，在linux系统中dns缓存是以二进制保存在/var/cache/nscd/hosts，



# 3、总结

使用特定IP地址访问http资源，修改hosts文件是最简单但不是最好的办法， 如果使用libcurl库，可以使用CURLOPT_RESOLVE选项完成此功能。

# 4、 参考

（1）官方CURLOPT_RESOLVE选项介绍：https://curl.haxx.se/libcurl/c/CURLOPT_RESOLVE.html

（2）libcurl使用ip直连https在实际场景中的应用：https://my.oschina.net/mltong/blog/1554223

（3）[How to read the local DNS cache contents?](https://unix.stackexchange.com/questions/28553/how-to-read-the-local-dns-cache-contents)





