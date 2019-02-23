---
layout: post
title: Qt中的new和delete
categories: Programming
---

> Delete, or not delete, that is the question.

<!-- more -->

~~你已经是个成熟的系统了，该学会自己释放内存了。~~

## 从新建一个对话框开始
使用 new 新建一个对象，然后通过返回的指针对其进行操作，这是在 Qt 中十分常见的写法。例如我想要创建一个这样的对话框，最开始的代码是这样的：

![test_dialog](/public/image/test_dialog.png)
{: align="center"}

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

## 只有 new 没有 delete？
很明显，上面的代码中只有 new 而没有 delete，也就是说写的时候只管 new，new 完了就用，用完了就不管了，new 分配的内存留给操作系统在程序结束后自己去回收。  

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

## 一个 new 对应一个 delete？
修改完毕后编译运行，诶？关闭对话框后怎么提示 crashing 了，不是说好的一个 new 要对应一个 delete 吗？  
于是去查找 Qt 的内存管理相关知识，在[Object Trees & Ownership](https://doc.qt.io/qt-5/objecttrees.html)有提到，**Qt 中的 QObject 会根据代码中指定的 parent 组织成树形结构，当树中的一个节点被删除时，它会从父节点的记录中去掉自己，并自动删除自己的所有子节点。**因此一次 delete 操作会删除以该节点为根节点的整棵子树，同时也保证了每个节点只会被删除一次。  
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

## QHBoxLayot 和加入的 Widget 的关系？
那么问题来了，代码中的 QHBoxLayout 和使用 addWidget() 方法添加进去的 Widget 是什么关系呢？  
想知道 QHBoxLayout 和其它 Widget 的关系，可以直接通过 Qt 提供的方法查看 QObject 组成的树形结构，只需要在`dialog->exec();`之前加上几行：
```cpp
qDebug() << "--> dump dialog tree";
dialog->dumpObjectTree();
qDebug() << "--> dump dialogLayout tree";
dialogLayout->dumpObjectTree();
```
程序输出如下：

![dump_object_tree](/public/image/dump_object_tree.png)
{: align="center"}

可以看到，QHBoxLayout 和 QPushButton 同级，并且 QHBoxLayout 的子节点为空，所以删除 QHBoxLayout 并不会影响到加入其中的 Widget，[QBoxLayout Class](https://doc.qt.io/qt-5/qboxlayout.html)中的析构函数也明确标明了：
> **QBoxLayout::~QBoxLayout()**  
> Destroys this box layout.  
>   
> The layout's widgets aren't destroyed.

## 向 QHBoxLayout 加入的 QSpacerItem 怎么不见了？
那么问题又来了，代码中的 QSpacerItem 呢？输出的树形结构中找不到啊，它没有继承 QObject，不能指定 parent，它还需要在代码中手动 delete 吗？  
在搜索引擎上没有直接找到该问题的答案，于是我去查找了一下 Qt 里相关的源码。

[QHBoxLayout](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.h.html#QHBoxLayout) 的析构函数为空，于是直接去找它继承的[QBoxLayout](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.h.html#QBoxLayout)，析构函数 [~QBoxLayout()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayoutD1Ev) 如下：
```cpp
QBoxLayout::~QBoxLayout()
{
    Q_D(QBoxLayout);
    d->deleteAll();
}
```
在这里出现了一个`deleteAll()`函数，可以在 QBoxLayoutPrivate 的[这里](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN17QBoxLayoutPrivate9deleteAllEv)找到它的定义：
```cpp
inline void deleteAll()
{
    while (!list.isEmpty())
        delete list.takeFirst();
}
```
[QBoxLayoutPrivate](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#QBoxLayoutPrivate) 涉及到了 Qt 的 [D-Pointer](https://wiki.qt.io/D-Pointer)。简单来说，就是 QBoxLayout 把自己需要用到的数据结构放在了对应的 QBoxLayoutPrivate 中，QBoxLayout 中只保存类的各种方法和一个指向 QBoxLayoutPrivate 的指针，这样就可以保证在 Qt 更新时，底层的库虽然改变了，但编译的程序中对象的大小不变，不影响已编译程序的运行，从而实现二进制兼容。源码中常见的`Q_DECLARE_PUBLIC(), Q_DECLARE_PRIVATE(), Q_Q(), Q_D()`等宏定义均是与此有关，更多内容可以去了解 D-Pointer 的有关知识，这里只需要知道 QBoxLayout 的数据是存放在对应的 QBoxLayoutPrivate 中的就行了。 

这里的 list 就是 QBoxLayoutPrivate 中的`QList<QBoxLayoutItem *> list;`，那接下来就去看看里面存的内容。  

向 list 中加入内容的动作主要发生在向 QBoxLayout 中添加内容的时候，例如`addSpacing(), addStretch(), addSpacerItem(), addLayout(), addWidget()`几个函数，它们又都是调用的对应的`insertXXX()`函数，以[addSpacerItem()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayout13addSpacerItemEP11QSpacerItem)为例：
```cpp
void QBoxLayout::addSpacerItem(QSpacerItem *spacerItem)
{
    insertSpacerItem(-1, spacerItem);
}
```
对应的[insertSpacerItem()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayout16insertSpacerItemEiP11QSpacerItem)函数为：
```cpp
void QBoxLayout::insertSpacerItem(int index, QSpacerItem *spacerItem)
{
    Q_D(QBoxLayout);
    if (index < 0)                                // append
        index = d->list.count();
    QBoxLayoutItem *it = new QBoxLayoutItem(spacerItem);
    it->magic = true;
    d->list.insert(index, it);
    invalidate();
}
```
就是用传入的 QSpacerItem 新建了一个 QBoxLayoutItem 对象，然后将新建的 QBoxLayoutItem 对象插入到 list 中，那就再去看一下[QBoxLayoutItem](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#QBoxLayoutItem)：
```cpp
struct QBoxLayoutItem
{
    QBoxLayoutItem(QLayoutItem *it, int stretch_ = 0)
        : item(it), stretch(stretch_), magic(false) { }
    ~QBoxLayoutItem() { delete item; }
    
    /*** 其它内容 ***/

    QLayoutItem *item;
    int stretch;
    bool magic;
};

```
也就是说，创建 QBoxLayoutItem 对象时构造函数会把 QSpacerItem 的指针保存到它的 item 中，析构时则会调用 delete 进行释放，那么之前的`deleteAll()`函数就会释放 QHBoxLayout 中的 QSpacerItem 了。  

其它的`insertSpacing(), insertStretch(), insertLayout()`也都是类似的处理，所以继承自 QLayoutItem 的类不属于 QObject，它们在被加入到 QBoxLayout 后，会随着 QBoxLayout 的 delete 而被释放，不需要手动进行释放。  

那么还有一个[insertWidget()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qboxlayout.cpp.html#_ZN10QBoxLayout12insertWidgetEiP7QWidgeti6QFlagsIN2Qt13AlignmentFlagEE)呢？它的里面也有一个`list.insert()`，为什么QWidget不会被 delete 呢？  
```cpp
void QBoxLayout::insertWidget(int index, QWidget *widget, int stretch,
                              Qt::Alignment alignment)
{
    Q_D(QBoxLayout);
    if (!d->checkWidget(widget))
         return;
    addChildWidget(widget);
    if (index < 0)                                // append
        index = d->list.count();
    QWidgetItem *b = QLayoutPrivate::createWidgetItem(this, widget);
    b->setAlignment(alignment);
    QBoxLayoutItem *it = new QBoxLayoutItem(b, stretch);
    d->list.insert(index, it);
    invalidate();
}
```
可以看到，这里传递给 QBoxLayoutItem 的构造函数用来实例化的指针是`b`，`b`则是由`QLayoutPrivate::createWidgetItem(this, widget);`返回的，查找这里的[createWidgetItem()](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayout.cpp.html#_ZN14QLayoutPrivate16createWidgetItemEPK7QLayoutP7QWidget)：
```cpp
// Static item factory functions that allow for hooking things in Designer

QLayoutPrivate::QWidgetItemFactoryMethod QLayoutPrivate::widgetItemFactoryMethod = 0;
QLayoutPrivate::QSpacerItemFactoryMethod QLayoutPrivate::spacerItemFactoryMethod = 0;

QWidgetItem *QLayoutPrivate::createWidgetItem(const QLayout *layout, QWidget *widget)
{
    if (widgetItemFactoryMethod)
        if (QWidgetItem *wi = (*widgetItemFactoryMethod)(layout, widget))
            return wi;
    return new QWidgetItemV2(widget);
}
```
所以这里返回的是`QWidgetItemV2(widget)`，继续追踪[QWidgetItemV2](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayoutitem.h.html#QWidgetItemV2)，其继承自[QWidgetItem](https://code.woboq.org/qt5/qtbase/src/widgets/kernel/qlayoutitem.h.html#QWidgetItem)，主要查看其构造函数和析构函数，QWidgetItem 析构函数为空，带参构造函数只是把传进去的 QWidget 指针存到了 wid 里，因此重点在 QWidgetItemV2 的构造函数和析构函数中：
```cpp
QWidgetItemV2::QWidgetItemV2(QWidget *widget)
    : QWidgetItem(widget),
      q_cachedMinimumSize(Dirty, Dirty),
      q_cachedSizeHint(Dirty, Dirty),
      q_cachedMaximumSize(Dirty, Dirty),
      q_firstCachedHfw(0),
      q_hfwCacheSize(0),
      d(0)
{
    QWidgetPrivate *wd = wid->d_func();
    if (!wd->widgetItem)
        wd->widgetItem = this;
}
QWidgetItemV2::~QWidgetItemV2()
{
    if (wid) {
        auto *wd = static_cast<QWidgetPrivate *>(QObjectPrivate::get(wid));
        if (wd->widgetItem == this)
            wd->widgetItem = 0;
    }
}
```
构造函数是将传进来的 QWidget 指针对应对象的 widgetItem 指向了自己，而析构只是将其置零，并没有涉及 delete 操作。

因此整个流程是这样的：当调用`addWidget()`向布局中加入 QWidget 时，会去调用`insertWidget()`，然后新建一个 QWidgetItemV2 对象，QWidget 的一个 widgetItem 指针指向新建的 QWidgetItemV2 对象，QWidgetItemV2 的指针被加入到了 QBoxLayout 的 list 中。当 QBoxLayout 被释放时，它会 delete 自己 list 中的内容，即 QLayoutItem 被 delete，QWidgetItemV2 也被 delete，而 QWidgetItemV2 对应的 QWidget 只是把 widgetItem 指针置零，QWidget 并不会被释放。  

作为验证，我又往`dialog->exec();`之前加上了几行，直接使用`qDebug()`显示对象指针：
```cpp
qDebug() << "item 0         " << dialogLayout->itemAt(0);
qDebug() << "spacer         " << spacer;
qDebug() << "item 1         " << dialogLayout->itemAt(1);
qDebug() << "okButton       " << okButton;
qDebug() << "item 2         " << dialogLayout->itemAt(2);
qDebug() << "cancelButton   " << cancelButton;
```
结果如下：

![compare_item_pointer](/public/image/compare_item_pointer.png)
{: align="center"}

加入的 QSpacerItem 属于 QLayoutItem，因此它被直接添加到了 QHBoxLayout 的 list 中，会随着 QHBoxLayout 的 delete 而被一同释放。剩下的两个 QPushButton 则属于 QWidget，list 中加入的是它们各自对应的 QWidgetItemV2，QHBoxLayout 被 delete 时它们不会受到影响。


PS: 在线查看 Qt 源码的网站使用的[Code Browser](https://code.woboq.org/)体验不错。
