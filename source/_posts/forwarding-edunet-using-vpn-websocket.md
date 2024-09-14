---
title: 使用校园VPN服务配合wstunnel实现校园网内网穿透
date: 2024-09-12 00:19:51
tags: 
    - wstunnel
    - 内网穿透
category: 杂物
---
## 起因

最近题也不写了，课也不听了，在宿舍摆烂的时候突然想到能不能利用学校提供的`aTrust VPN`服务访问到学校内网的`Minecraft`服务器，这样就不用再去付费买那些不太靠谱的内网穿透服务了
<!-- more -->
## 网络环境

学校提供的校园网是有IPv6地址的。`240c`开头，`test-ipv6`测试也是满分，但是在外网访问不到，也就是在校园网外`ping`不通，但是在校内是通的。
IPv4地址似乎没什么~~利用价值~~用，其他设备怎么`ping`都`ping`不通。
另外不知道为什么学校提供的运营商代拨服务不给你提供IPv6地址。就算连接Wifi的时候给你分配了IPv6，也会在选择代拨上网后失效，~~这点一直想吐槽~~

## 冒出想法

哎🤓👆，我有个想法

学校的VPN服务可以访问到学校内网的服务，那是不是也可以用`IPv6`访问到内网的任意主机呢？
![image](2024/09/06/welcome/ng-edunet-using-vpn-websocket/image.png)
然后就被这个不支持IPv6的前端网页气笑了

但是别急，我们还有一种曲线救国的方式

## 配置自定义域名解析

在`aTrust VPN`里访问`https://www.baidu.com`后，浏览器弹出了一个新标签页，URL形似`https://www-baudu-com-s.vpn.xxxx.edu.cn`

很显然，URL前面的`www-baudu-com-s`其实就是把代理域名`www.baidu.com`中的`.`换成了`-`，https链接会在后面额外加上`-s`，那么我们只需要拥有一个域名，设置一个`AAAA`解析指向自己的电脑IPv6地址，理论上就可以访问内网主机上的http资源了

我这里直接在本地开了一个80端口的hexo server
```bash
hexo server -p 80
```

在`cloudflare`上添加一条`AAAA`记录，指向自己的IPv6地址。然后将自己的域名按上述方式处理一下并尝试访问
![image](2024/09/06/welcome/ng-edunet-using-vpn-websocket/image-1.png)
可以看到我的blog已经成功加载出来了，也就是说`aTrust VPN`不仅可以访问学校的服务，还可以访问任意内网主机

## VPN授权方式

观察代理网站的`Cookie`列表

![image](2024/09/06/welcome/ng-edunet-using-vpn-websocket/image-2.png)

发现可疑的`Cookie`仅有一项，`sdp_user_token`，其他的`Cookie`应该都是被代理的网站的`Cookie`（名称中有网站域名）

删除这个`Cookie`，刷新页面发现网站跳转到了登录界面。那么基本可以确定是这个`sdp_user_token`负责标识用户，也就是只需要在请求中带上这个`Cookie`就可以通过代理访问网站。~~（还是挺简单的）~~

## 配合wstunnel进行端口转发

既然能够进行http访问，而且也能顺利通过授权，那么就可以请出神器[wstunnel](https://github.com/erebe/wstunnel)了

设置好域名解析后，在本地运行`wstunnel server`：
```bash
wstunnel server wss://[::]:443
```

然后在需要远程访问的主机上，运行`wstunnel client`：
```bash
wstunnel client wss://wstunnel-xxxx-xxx-s.vpn.xxx.edu.cn:8081/ -H Cookie:sdp_user_token=xxx-xxx-xxx -L tcp://xxxxx:localhost:xxxxx
```

我这里转发了一个`Minecraft`服务器，在`1000km`外的地方进行访问，延迟在`60~80ms`左右，相当优秀

## 持久化

这个VPN服务是有超时自动下线功能的，也就是只要一段时间内没有操作就会被强制下线。而且重新登录还需要手机验证码，非常不利于持久化的服务 ~~（接好几次手机验证码太烦了）~~。

经过测试，大约一小时以内没有操作就会下线，而不管有没有还在传输数据的`WebSocket`。

但是如果定时进行一个`HTTP`请求，似乎就不会被踢下线

（挂了两天了还没寄，等我再挂一段时间）