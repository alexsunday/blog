---
layout: post
title: C++/Qt与PyQt
tags: [Qt, C++, PyQt, python]
keywords: Qt, C++, PyQt, python
---

{{ page.title }}
================ 

##举个栗子
这里是一个简单的C++的Qt应用例子:

```c++
#include <QApplication>
#include <QtGui>
#include <QtCore>

int main(int argc, char *argv[])  
{  
   QApplication app(argc, argv);  
   QSplitter *splitter = new QSplitter;//QSplitter用户分割两个widget  
   QFileSystemModel *model = new QFileSystemModel;  
   model->setRootPath(QDir::currentPath());  
   QTreeView *tree = new QTreeView(splitter);  
   tree->setModel(model); //为视图设置模型  
   tree->setRootIndex(model->index(QDir::currentPath()));  
   QListView *list = new QListView(splitter);  
   list->setModel(model);   
   list->setRootIndex(model->index(QDir::currentPath()));  
   splitter->setWindowTitle("Two views onto the same file system model");  
   splitter->show();  
   return app.exec();  
}  
```

对应的Python代码应该是

{% highlight python %}
from PyQt4 import QtGui
from PyQt4 import QtCore
import sys

def main(argv):
    app = QtGui.QApplication(argv)
    splitter = QtGui.QSplitter()
    model = QtGui.QFileSystemModel()
    model.setRootPath(QtCore.QDir.currentPath())
    tree = QtGui.QTreeView(splitter)
    tree.setModel(model)
    tree.setRootIndex(model.index(QtCore.QDir.currentPath()))
    qlv_list = QtGui.QListView(splitter)
    qlv_list.setModel(model)
    qlv_list.setRootIndex(model.index(QtCore.QDir.currentPath()))
    splitter.setWindowTitle("Two views onto the same file system model")
    splitter.show()
    return app.exec_()

if __name__ == '__main__':
    main(sys.argv)
{% endhighlight %}

运行时界面如图
![PyQt Demo]({{site.baseurl}}images/cplus_qt_to_py_qt.png)

##变量生存周期
- C++使用RTTI进行内存管理，或者手动管理
- Python使用引用计数，自动管理内存
- Qt内使用其树模型管理内存，父节点析构时，自动析构所有子节点

是故，使用C++/Qt编程时，几乎所有的QObject实例对象构造时，都会记得加上其parent的引用以防止内存泄漏；同样，PyQt中依然 如此，倒不是防止其内存泄漏，因为Python会根据引用计数自动对类实例进行管理，一旦变量的超出其作用域而没有任何引用时会被自动析构，但生存于Qt 树模型中的对象却能幸免.

##其他异同
- 在PyQt中，所有的新对象均使用类似于栈对象的形式出现，但要留意其对象的生存周期，留意何时对象会被『析构』
- Python没有 『栈对象』与『堆对象』之分，故你可几乎无视C++的指针与引用之间的区别
- 拜上一点所赐，顺带可以无视C++的 实例引用(.) 与 指针引用(->) 之间的区别，在Python中，全部使用 (.) 点号替代
- Python的静态方法调用与实例方法调用方式都是用 (.) 点号，而C++用 命名空间符 (::)，在PyQt中仍然全部使用 (.) 点号替代
- Python中没有C/C++中的enum（枚举）型，故枚举型的引用方式有所差异
- Python使用包、库的概念，而C++使用头文件概念，Qt没有使用C++的 namespace（命名空间）
- 留意部分特殊对象的生成方式如QIcon，这货大部分情况下在C++/Qt里都是栈对象，并非生存于Qt的树模型中
- 部分特殊对象如C++的QString与Python的String以及对应的Unicode之间的相互转换关系，请弄清楚平台编码(一般而言，Win32使用GB系列，*Nix使用UTF-8)、命令行显示编码(指win32的cmd，一般使用GB系列)、终端编码(如果你使用Putty或者XShell之类的工具，请留意终端编码与shell的$LANG环境变量)、文件编码、内存编码(如有可能，内存中的string，请全部使用 unicode)
- 标准库的对象转换关系，如QList与std::list与Python的list，我已经开始有点晕了，以此衍生的如 QListView，QStringList等，另外std::map，QMap，dict for python就不细说了，还是细看文档吧，Qt的文档很棒。
- 留意那些非QObject子类的对象如Qt的图形视图框架中的QGraphicsItem，别被坑到了
- Python2.7的关键字exec，PyQt中替换为exec_，在QDialog中亦是如此，这在Py3K中得到了改进


