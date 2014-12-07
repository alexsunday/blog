---
layout: post
title: 写一个VPN
tags: [pyvpn, TCP/IP, tun/tap, VPN]
keywords: pyvpn, TCP/IP, tun/tap, VPN
description: VPN很神奇有木有，自己写一个了不起有木有
---

- VPN释义
- 常见用途
- 当下各种典型VPN
- 各自的痛点
- 我们要做什么
- 代码

VPN（英语：Virtual Private Network，简称VPN）本意为「虚拟专用网络」，维基百科释义为「一种常用于连接中、大型企业或团体与团体间的私人网络的通讯方法」。

从以上定义看出，VPN用于「连接中、大型企业团体」的一种通讯方式。比如日常生活中的例子，当员工出差后，使用酒店内的网络接入互联网后，却并不能如往常一样访问位于公司内部的ERP等功能，而当其连入公司搭设的VPN后，则如同重新回到公司的网络环境一般，照常访问公司内网的其他机器，考勤、财务、等，连打印机都不放过。

更简单的描述是，VPN是一种让员工下班回家后继续办公的一种「资本主义压迫劳动者」的技术。

VPN开始的时候的确如此，但这项技术在中国，却变成了一种跨越围墙的技术，甚至有大量的商(pi)业(bao)公司提供付费的VPN技术。

常用的VPN协议有
- L2TP
- PPTP
- IPsec
- OpenVPN

还有一些未列出的如Cisco等，均有自己的VPN实现，但与前三者类似，都是一些寡头标准而已，只是由于L2TP以及PPTP有一些开源实现。而OpenVPN则是一种完全不同思路的开源实现，通过使用Linux内核中的TUN/TAP设备，联合OpenSSL的加密函数，编写的应用层VPN，不与前述的寡头标准VPN兼容，安全性最好，部署也最为灵活，但win32操作系统中默认不包含客户端，需要下载使用专用客户端。

L2TP与PPTP等对网络要求高，需要特定的网络结构方能部署，其他如Cisco等则需要专用设备，部署安装都不够方便；OpenVPN在绝大多数的情况下都是比较好的选择，但繁琐到令人费解的公钥私钥配置、路由配置等，尚有待改进；另外很多情况下，我们并不需要如此强大的密钥加密方式，只需要简单的用户名密码登录VPN或者甚至连用户名密码都可省略的情况，也是有的。 另外针对提供付费VPN的公司，虽然部署不是难题，但其数据加密需求却不是很强烈，来往数据的加密解密，却产生了大量不必要的CPU运算，使得性能低下。

如果这些都是问题，其实我们可以看看vTun这个项目，似乎不错。

本文将使用python，制作一个足够简化的vpn，包含其客户端与服务端实现，总代码行数控制在100行以内。

```python
#!/usr/bin/env python
# encoding: utf-8
import os
import time
import socket
import select
import logging
import struct
import subprocess
import fcntl  # @UnresolvedImport
import sys
logger = logging.getLogger('vpn')
logger.addHandler(logging.StreamHandler())
logger.setLevel(logging.DEBUG)


def make_tun():
    TUNSETIFF = 0x400454ca
    IFF_TUN = 0x0001
    IFF_NO_PI = 0x1000
    tun = open('/dev/net/tun', 'r+b')
    ifr = struct.pack('16sH', 'tun%d', IFF_TUN | IFF_NO_PI)
    fcntl.ioctl(tun, TUNSETIFF, ifr)
    return tun


def client():
    tundev = make_tun()
    tunfd = tundev.fileno()
    logger.info(u'TUN dev OK')
    time.sleep(1)
    subprocess.check_call('ifconfig tun0 192.168.10.2/24 up', shell=True)
    subprocess.check_call('route add -net 192.168.0.1/24 gw 192.168.10.1 tun0',
                          shell=True)
    time.sleep(1)

    sock = socket.socket()
    addr = ('heruilong1988.oicp.net', 23456)
    sock.connect(addr)
    logger.info(u'SOCK dev conn OK')
    sock.setblocking(False)
    sockfd = sock.fileno()

    buflen = 65536
    fds = [tunfd, sockfd, ]
    while True:
        rs, _, _ = select.select(fds, [], [], 0.1)
        for fd in rs:
            if fd == tunfd:
                logger.info('TUN recv DATA')
                os.write(sockfd, os.read(tunfd, buflen))
            elif fd == sockfd:
                logger.info(u'SOCK recv DATA')
                os.write(tunfd, os.read(sockfd, buflen))


def server():
    buflen = 65536
    tundev = make_tun()
    tunfd = tundev.fileno()
    logger.info(u'TUN dev OK')
    subprocess.check_call('ifconfig tun0 192.168.10.1/24 up', shell=True)
    time.sleep(1)

    sock = socket.socket()
    laddr = ('192.168.0.192', 23456)
    sock.bind(laddr)
    sock.listen(socket.SOMAXCONN)
    logger.info(u'Sock Listen OK')
    sock.setblocking(False)
    sockfd = sock.fileno()

    fds = [tunfd, sockfd, ]
    while True:
        rs, _, _ = select.select(fds, [], [], 0.1)
        for fd in rs:
            if fd == sockfd:
                cs, ca = sock.accept()
                logger.info(u'Remote sock addr: [%s:%d]' % ca)
                fds.append(cs.fileno())
            elif fd == tunfd:
                logger.info(u'TUN dev recv DATA, rs:[%r]' % rs)
                for client_fd in fds:
                    if client_fd not in [tunfd, sockfd]:
                        os.write(client_fd, os.read(tunfd, buflen))
            else:
                logger.info(u'SOCK dev recv DATA')
                os.write(tunfd, os.read(fd, buflen))


if __name__ == '__main__':
    if sys.argv[1] == '-s':
        sys.exit(server())
    elif sys.argv[1] == '-c':
        sys.exit(client())
    else:
        print u'Usage: pyvpn [-s] [-c]'
```

