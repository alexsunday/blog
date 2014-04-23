---
layout: post
title: 非root构建隔离的Python应用程序环境
---

{{ page.title }}
================ 

在Linux中，尤其是服务器操作系统，Python的版本总是低于日常使用的版本，譬如最常见的服务器发行版CentOS，其6.5版本上的Python为2.6.6，但当前python.org上的版本则是2.7.6/3.4.1，相差还是很大的。一般地，我们可以使用virtualenv，这一强大组件使得我们可以构建隔离的环境，但virtualenv仍然是基于原有python版本，进行软链接并修改PYTHONPATH变量实现。基于如下理由，我们需要构建一个**用户层隔离的Python环境**。

- `yum等系统原有组件依赖特定python版本` yum、ibus等系统应用程序对内置版本有依赖，基于此，我们不能简单configure make && make install替换操作系统原有的Python。当然，你可以通过ln -s 并同时[hack yum](http://blog.csdn.net/jcjc918/article/details/11022345)来实现，我总觉得自己水平不够，不到万不得已不想去hack 系统原有组件。
- `无root权限` 无root权限时，想通过easy_install或pip安装第三方库，可以使用virtual env实现，不过使用的是操作系统的旧版Python。
- `不想将python库安装到系统空间` 有洁癖的管理员还是有很多的，有不少管理员固执的认为pypi不同于yum的源，不愿意直接将pypi的库安装到系统中。
- `需要尝试不同的python版本` 甚至是尝试不同的库版本，如SQLAlchemy等，处于快速开发中的库往往跨版本有很大差异，需要尝试不同的版本以重现生产bug是很常见的需求。
- `重新编译一个方便很多` 对的，重新编译一个会方便很多，自动获得Python对应版本的头文件，Build C extension无压力。

如何即不影响系统原有的流程，又能使用上python世界的最新特性，便是本文的主题，本文可以看作是对virtualenv的一个补充，不影响virtualenv的继续使用。

我们知道，Linux中，控制shell启动应用程序，由**PATH环境变量**控制，处于PATH环境变量指定目录中的可执行程序均可被直接执行，如bash中启动vim之类命令:
``` Bash
$ which python
>>/usr/bin/python
$ echo $PATH
>>/opt/java/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/home/bigdata/bin
```

除了PATH环境变量，有如下环境变量需要配置:

- `PATH` 必配项，一般而言我们可配置为prefix下的bin目录，有些你或许需要将sbin也加上，如nginx、spark等
- `LD_LIBRARY_PATH` 必配项，控制应用程序动态库查找路径顺序，通过指定此路径
- `MANPATH` 可选项，顾名思义，控制man命令查找其文档的路径
- `PKG_CONFIG_PATH` 可选项，编译安装一些较复杂系统组件时[使用](http://www.chenjunlu.com/2011/03/understanding-pkg-config-tool/)

我们同时还知道，Linux中编译安装多数为固定三步走，如下:

``` Bash
./configure
make
make install 
```

在第一步的configure配置中，有一个至关重要的参数prefix，指明安装路径，make install 会将系统拷贝(install, not cp)至你指定的目录。
``` Bash
./configure --prefix=$HOME/local
```

所以，构建一个用户态的Python极为简单，简单到就是『纸老虎』一只。简言之，配置以PATH为首的一系列环境变量即可，并将其持久化到用户配置文件如 `$HOME/.bash_profile` 或 `$HOME/.bashrc` 中，一般而言，桌面用户选择后者即可，参考此处[关于Login Shell与Non Login Shell的区别](https://wido.me/sunteya/understand-bashrc-and-profile)。

这里有我常使用的一个脚本，通常的用法是复制并在如[XShell](http://www.netsarang.com/products/xsh_overview.html)之类的工具上直接按`Shift-Insert`即可粘贴到终端中，如下：

``` Bash
mkdir $HOME/local/{bin,lib,include,sbin,etc,package} -pv
cd $HOME/local/package
wget -c http://mirrors.sohu.com/python/2.7.6/Python-2.7.6.tgz
tar xvfz Python-2.7.6.tgz
cd Python-2.7.6
./configure --prefix=$HOME/local
make
make install

cat >>~/.bash_profile <<EOF
PATH=\$HOME/local/bin:\$PATH
LD_LIBRARY_PATH=\$HOME/local/lib:\$LD_LIBRARY_PATH

export PATH
export LD_LIBRARY_PATH
EOF

source $HOME/.bash_profile
cd $HOME/local/package
wget -c http://pypi.douban.com/packages/source/s/setuptools/setuptools-2.0.1.tar.gz
tar xvfz setuptools-2.0.1.tar.gz
cd setuptools-2.0.1
python setup.py install

easy_install -i http://pypi.douban.com/simple/ ipython
easy_install -i http://pypi.douban.com/simple/ django
easy_install -i http://pypi.douban.com/simple/ SQLAlchemy
easy_install -i http://pypi.douban.com/simple/ virtualenv
```
以上就是全部，有几点需要留意:

- `mkdir $HOME/local/xxx......`我个人一般习惯于在HOME目录下创建一个『伪根目录结构』，将所有用户态的应用，都安装到$HOME下。
- 使用了sohu.com的源，你可以改为163的或者官方的均可
- 修改用户bash_profile文件，对于桌面用户，此处可能需要改为 $HOME/.bashrc
- 留意环境变量的改动，只简单的修改了PATH与LD\_LIBRARY\_PATH变量，对于部分应用，你也许需要如CLASSPATH(Java)，MANPATH等
- source命令使环境变量立即生效，对于服务器用户而言使用.bash_profile是正确的，对于桌面用户而言，可能需要修正为$HOME/.bashrc
- 此处使用了豆瓣源，安装一些常见包，并演示常见包安装用法，可通过配置easy\_install来实现无需每次easy\_install时都要指定pypi源

嗯，执行python试试:

``` Bash
$ python
Python 2.7.6 (default, Jan 11 2014, 19:15:27) 
[GCC 4.4.7 20120313 (Red Hat 4.4.7-4)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

是吧，已经切换为了最新的2.7.6版本了，也可以试试ipython命令：

``` Bash
$ ipython
Python 2.7.6 (default, Jan 11 2014, 19:15:27) 
Type "copyright", "credits" or "license" for more information.

IPython 1.1.0 -- An enhanced Interactive Python.
?         -> Introduction and overview of IPython's features.
%quickref -> Quick reference.
help      -> Python's own help system.
object?   -> Details about 'object', use 'object??' for extra details.

In [1]: 
```

是否已在非root用户下，成功的构建了一个全套的python环境了，即使是现在，你仍然可以使用virtualenv之类的第三方隔离工具，在用户环境下进一步隔离，以上就是鄙人平时开发中的一点小小的心得，现分享并整理。

