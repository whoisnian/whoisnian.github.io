---
layout: post
title: Qt中的new和delete
categories: Programming
---

> Delete, or not delete, that is the question.

<!-- more -->

~~你已经是个成熟的系统了，该学会自己释放内存了。~~

使用 new 新建一个对象，然后通过返回的指针对其进行操作，这是在 Qt 中十分常见的写法。例如我想要创建一个这样的对话框，最开始的代码是这样的：
![test_dialog](/public/image/test_dialog.png)
```cpp
#include <QApplication>
#include <QDebug>
#include <QDialog>
#include <QHBoxLayout>
#include <QLayoutItem>
#include <QPushButton>
#include <QSpacerItem>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    QDialog *dialog = new QDialog;
    QHBoxLayout *dialogLayout = new QHBoxLayout;
    QSpacerItem *spacer = new QSpacerItem(0, 0, QSizePolicy::Expanding);
    QPushButton *okButton = new QPushButton("确定");
    QPushButton *cancelButton = new QPushButton("取消");

    QObject::connect(okButton, SIGNAL(clicked()), dialog, SLOT(accept()));
    QObject::connect(cancelButton, SIGNAL(clicked()), dialog, SLOT(reject()));

    dialogLayout->addSpacerItem(spacer);
    dialogLayout->addWidget(okButton);
    dialogLayout->addWidget(cancelButton);

    dialog->setLayout(dialogLayout);
    dialog->setMinimumWidth(400);

    dialog->exec();
    return 0;
}
```
写的时候只管 new，new 完了就用，用完了就不管了，new 分配的内存留给操作系统在程序结束后自己去回收。  

在这里这么写不是大问题，因为对话框关闭后程序会立即结束，new 分配的内存并不会占用太长时间，很快就由操作系统回收了。  

那么如果在其它地方创建一个这样的对话框，关闭对话框后程序并没有立即结束呢？  
new 分配的内存就会一直得不到释放，占用的内存被白白浪费，造成了内存泄漏（memory leak）。  

[维基百科](https://en.wikipedia.org/wiki/New_and_delete_(C%2B%2B))上有提到，每一个 new 操作必须要有相应的 delete 操作，否则将会引起内存泄漏：  
> Every call to `new` must be matched by a call to `delete`; failure to do so causes memory leaks.

那么我就尝试在`return 0;`之前加上几行：
```cpp
delete dialog;
delete dialogLayout;
delete spacer;
delete okButton;
delete cancelButton;
```
编译运行，诶，关闭对话框后怎么提示 crashing 了，不是说好的一个 new 要对应一个 delete 吗？于是去查找 Qt 的内存管理相关知识，在[Object Trees & Ownership](https://doc.qt.io/qt-5/objecttrees.html)有提到，Qt 中的 QObject 会根据代码中指定的 parent 组织成树形结构，当树中的一个节点被删除释放时，它会从父节点的记录中去掉自己的记录，并依次删除释放自己的子节点，从而删除释放以自己为根节点的整棵子树。  
> When QObjects are created on the heap (i.e., created with new), a tree can be constructed from them in any order, and later, the objects in the tree can be destroyed in any order. When any QObject in the tree is deleted, if the object has a parent, the destructor automatically removes the object from its parent. If the object has children, the destructor automatically deletes each child. No QObject is deleted twice, regardless of the order of destruction.

所以上面的对话框正确的写法应该是这样的：
```cpp
#include <QApplication>
#include <QDebug>
#include <QDialog>
#include <QHBoxLayout>
#include <QLayoutItem>
#include <QPushButton>
#include <QSpacerItem>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    QDialog *dialog = new QDialog;
    QHBoxLayout *dialogLayout = new QHBoxLayout(dialog);
    QSpacerItem *spacer = new QSpacerItem(0, 0, QSizePolicy::Expanding);
    QPushButton *okButton = new QPushButton("确定", dialog);
    QPushButton *cancelButton = new QPushButton("取消", dialog);

    QObject::connect(okButton, SIGNAL(clicked()), dialog, SLOT(accept()));
    QObject::connect(cancelButton, SIGNAL(clicked()), dialog, SLOT(reject()));

    dialogLayout->addSpacerItem(spacer);
    dialogLayout->addWidget(okButton);
    dialogLayout->addWidget(cancelButton);

    dialog->setLayout(dialogLayout);
    dialog->setMinimumWidth(400);

    dialog->exec();

    delete dialog;
    return 0;
}
```
使用 new 新建对象时需要指明它的 parent，然后只需要 delete 树形结构的根节点即可。  

那么问题又来了，代码中的 QHBoxLayout 和使用 addWidget() 方法添加进去的部件是什么关系呢？  
想知道 QHBoxLayout 和其它 Widget 的关系，可以查看 QObject 组成的树形结构，需要在`dialog->exec();`之前加上几行：
```cpp
qDebug() << "--> dump dialog tree";
dialog->dumpObjectTree();
qDebug() << "--> dump dialogLayout tree";
dialogLayout->dumpObjectTree();
```
程序输出如下：
![dump_object_tree](/public/image/dump_object_tree.png)
可以看到，QHBoxLayout 和 QPushButton 同级，所以删除 QHBoxLayout 并不会影响里面的 Widget，[QBoxLayout Class](https://doc.qt.io/qt-5/qboxlayout.html)的析构函数也明确标明：
> **QBoxLayout::~QBoxLayout()**  
> Destroys this box layout.  
>   
> The layout's widgets aren't destroyed.

那么问题又来了，代码中的 QSpacerItem 呢？树形结构里没见啊，它也不是继承于 QObject，也没有设定 parent，它还需要手动 delete 吗？  
在搜索引擎上没有直接找到该问题的答案，于是我去查找了一下 Qt 里相关的源码。

未完待续。。。

[QBoxLayout::~QBoxLayout()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayoutD1Ev)  
[void deleteAll()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN17QBoxLayoutPrivate9deleteAllEv)  
[void QBoxLayout::insertSpacerItem(xxx)](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayout16insertSpacerItemEiP11QSpacerItem)  
[void QBoxLayout::insertWidget(xxx)](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayout12insertWidgetEiP7QWidgeti6QFlagsIN2Qt13AlignmentFlagEE)  
[struct QBoxLayoutItem](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#QBoxLayoutItem)  
[QWidgetItem *QLayoutPrivate::createWidgetItem(xxx)](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayout.cpp.html#_ZN14QLayoutPrivate16createWidgetItemEPK7QLayoutP7QWidget)  
[QWidgetItemV2::QWidgetItemV2(xxx)](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayoutitem.cpp.html#_ZN13QWidgetItemV2C1EP7QWidget)  
[QWidgetItemV2::~QWidgetItemV2()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayoutitem.cpp.html#_ZN13QWidgetItemV2D1Ev)  

（答案是不需要，删除layout时它会自动删除自己的 QLayoutItem）
