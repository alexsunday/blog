---
layout: post
title: 为什么Tornado没有内置Session
keywords: Tornado, Session, Django, Python Web, Cookies
---

{{ page.title }}
================

`Tornado`我就不做科普了，[这里](http://www.tornadoweb.cn/)有介绍，[这里](http://www.tornadoweb.cn/documentation)有文档，文档也极尽简略，寥寥数语似乎就解决了这个性能飞快的Python Web框架。

纵观Tornado文档全部，也不会发现Sessions的踪影，为何在一个成熟的Web开发框架中，会缺少如此重要的功能呢？

##Session何许人
什么是Session，平日里用于判断用户是否登录，登录后台管理时管理员的判断？就像Cookies是浏览器为每个站点准备的4K大小的字符串并存放在用户Profile下的SQLite数据库一样，Session保存在哪里？

在服务端？PHP的的确[有个地方](http://www.php.net/manual/zh/session.configuration.php#ini.session.save-path)用于保存Session。但，仅仅如此么？

在客户端？PHP的童鞋同样可以看[这里](http://www.php.net/manual/zh/session.configuration.php#ini.session.name)。或者使用ASP或者.NET开发应用的人，对前端浏览器Cookies中的这个字段`ASPSESSIONIDSQRCRSTS`应该比较熟悉，像这样：

![ASP.NET Cookies]({{ site.baseurl}}images/tornado/asp_session.png)

所以，Session的实现，其实是 **浏览器Cookies + 服务端存储** 综合实现的。

##Session的缺陷
OK，实现一个Session并非难事，在服务端使用嵌入式的SQLite或者KV DB实现起来都不难，可为何Tornado选择不呢？

这里不得不提到的是**服务器单点问题**。

现代化的Web开发，早已不是那个靠着一台虚拟主机就能抗天下的时代了，互联网的普及带来了大规模用户的访问，对单台服务器的纵向扩展不足以满足日益增长的用户规模，分布式部署与Web服务器集群等技术的出现缓解了这一难题，却将当年**滥用Session**时所导致的问题，一一暴露。

考虑到使用了多台服务器的场景，多台服务器之间使用如Nginx或BigIP等负载均衡工具进行请求转发，问题便立即暴露，如下图所示：

![用户的第一次请求]({{ site.baseurl}}images/tornado/session1.png)

![用户的第二次请求]({{ site.baseurl}}images/tornado/session2.png)


用户第一次访问与第二次访问，若负载均衡服务器将两次请求『均衡』到了两台不同的服务器上，则第二次的请求极有可能因缺失服务器1的Session数据而导致失败。

HTTP协议应该是**无状态**的，这是协议的本质，即使我们引入了Cookies与Session，也应保证这一点。**传统Session本质的问题在于服务端并没有将Session数据进行分布式存储**，若Session数据本身就是分布式存储的，则无此扰，如Django，通过配置其`SESSION_ENGINE`到Memcached，可无痛解决此问题，详细的文档可参考[此处](https://docs.djangoproject.com/en/dev/topics/http/sessions/)。另外，[这里](http://michal.karzynski.pl/blog/2013/07/14/using-redis-as-django-session-store-and-cache-backend/)阐述了如何使用Redis做Django的Session后端。

是否，慢慢的发现，Session并不是如此简单的问题了，也能体会Tornado的开发者『放弃内置Session』的苦衷了吧，为什么Torando未内置Session，简言之，因为实现一个**足够正确的Session**很难，实现一个**满足所有人需求的Session**更难，故开发者选择了『无为而治』，放弃内置Session。

实际上通过Tornado实现Session的文章，网上已到处都是，随便摘抄都能解决手头的问题，或者干脆放弃使用Session这一概念，使用Tornado的[secure_cookie](http://www.tornadoweb.cn/documentation#cookie-cookie)。

##正确的Session
使用**Redis**或**Memcached（推荐）**解决Session问题，保持对大数据的敏感，保留充分的可扩展能力。这里有一份我曾经使用过的使用Redis实现的Session代码，极其简略，也许有bug，谨慎使用。

```python
class Session(object):
    '''secure_cookies_id == sessionid'''

    def __init__(self, sessionid, redisobj):
        self._expire = 60 * 60
        self._prefix = 'session_'
        self._id = sessionid
        self._redis = redisobj
        if self._id == '' or self._id is None:
            self._id = self.init_session()

    def get_sid(self):
        return self._id

    def init_session(self):
        while True:
            sid = self._prefix + md5(os.urandom(32) + str(time.time()))
            if not self._redis.exists(sid):
                break
        return sid

    def get(self, key):
        return self._redis.hget(self._id, key)

    def set(self, key, value):
        return self._redis.hset(self._id, key, value)

    def delkey(self, key):
        return self._redis.hdel(self._id, key)
```

这是使用方式

```python
class BaseHandler(tornado.web.RequestHandler):
    def initialize(self):
        sessionid = self.get_secure_cookie('SESSIONID')
        self.session = Session(sessionid, self.mc)
        self.set_secure_cookie('SESSIONID', self.session.get_sid())
```

