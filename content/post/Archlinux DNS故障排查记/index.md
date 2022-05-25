---
title: "Archlinux DNS故障排查记"
slug: 
date: 2021-05-25T12:07:07+08:00
draft: true
image: cover.jpg
categories:
    - 编程

tags: 
    - 网络
---
# 起因
新装的arch没用几天，印象里除了例行的yay -Syu滚动更新以外， 没有做其他的设置。突然某一天，konsole中ping不通google和github了，甚至于baidu也不能ping，奇怪的是，浏览器可以正常访问。
# 排查
于是开启了漫长的排查和搜索。首先想到的是DNS出问题了，因为konsole中ping IP是能ping通的。接着尝试了以下方法：
- 手动设置8.8.8.8等DNS服务器到/etc/resolv.conf中。随后发现该文件每次都会被网络托管的NetworkManager覆盖回去。遂再一次修改之后，执行chattr +i /etc/resolv.conf，禁止被修改。-未果
- 添加rc.conf文件，手动添加网卡进去interface=eth0 -未果
- 在/etc/hosts中添加本地DNS映射 -方向错误，不能解决这次的问题
再尝试以上全部失败后，一次尝试安装一款DNS分析软件过程中，意外的发现连不到任何的镜像服务器。于是开始怀疑是cgproxy配置的全局代理导致的问题。systemctl stop cgproxy 之后终于ping通了baidu。
# 解决
虽然关掉cgproxy之后可以正常ping了，但是全局代理不能不要哇。再重复阅读Arch安装文档后发现一个关键点：
> (重要）如果启用了 udp 的透明代理（dns 也是 udp），则给 v2ray 二进制文件加上相应的特权：`sudo setcap "cap_net_admin, cap_net_bind_service=ep" /usr/bin/v2ray `
否则 udp 的透明代理可能会出问题。
如果每次更新了 v2ray 二进制文件，都需要重新执行此命令。

重点就是每次更新v2ray二进制文件后都需要重新加特权，重启之后一起恢复正常。
# 复盘
1. 文档很重要。在后续查看文档之后，了解到v2ray在选中某些配置后（DNS拦截，udp透明代理）会将DNS托管给v2ray内置的DNS服务器。猜测是滚动更新连带更新了v2ray却没有赋权导致内部DNS服务异常。
2. 遇到问题应该一开始就查看相应的文档，而不是google，网上其他人的解决方案没有详细的上下文，往往并不适用。
3. 找准方向，一击破之。
4. 及时快照保存当前稳定的系统。