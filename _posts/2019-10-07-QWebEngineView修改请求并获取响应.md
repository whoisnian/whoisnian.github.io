---
layout: post
title: QWebEngineView修改请求并获取响应
categories: programming
---

> 在自己的Qt程序中想通过QQ邮箱的通讯录来获取好友列表，考虑到登录过程的不确定性，希望效果是在浏览器中打开QQ邮箱登录页面，用户手动登录后浏览器窗口自动关闭，然后程序再请求需要的内容。

<!-- more -->

## 好友列表获取过程
打开浏览器控制台，登录QQ邮箱后可以在网络请求中找到一个请求地址为`https://mail.qq.com/cgi-bin/laddr_lastlist`的请求，响应内容中的有效信息包括好友邮箱地址、昵称、备注、分组。  

测试发现url中`t=addr_datanew&category=hot&sid=0123456789abcdef`和cookie中的`sid`以及header中的`Referer`是必需的，其他的内容都可以省略。  

响应内容并不是常用的json格式，观察后发现更像是用js代码声明了一个对象放在括号里，双引号会被转义为`\x26quot;`，其他的emoji表情，unicode字符也是被转义为HTML实体后用`\x26`代替`&`，`\`会被转义为`\x5c`。  

## 为什么要用QWebEngineView？
Qt涉及到网络请求的话最先想到的应该是[QNetworkAccessManager](https://doc.qt.io/qt-5/qnetworkaccessmanager.html)，现有程序中的下载队列用的就是这个。  

但是概述中提到了”登录过程的不确定性“，具体来说就是我使用QQ帐号登录的时候，最理想的情况是点击登录后就登录成功，但实际情况下偶尔还会弹出滑块验证码，异地登录还有手机短信验证码，只是单次登录不需要记住密码的话直接扫二维码会更方便，因此使用QNetworkAccessManager就会有各种各样的情况要考虑，而且还不知道是否有情况遗漏。  

于是就想到了之前在Qt中用过的[WebEngine](https://doc.qt.io/qt-5/qtwebengine-index.html)。Qt中的WebEngine是基于Chromium项目的，加上它感觉给项目内置了个浏览器进去，不过考虑到私有项目就自己一个人用，体积大点也不会有太大影响。  

## 如何用QWebEngineView？
参考[Qt WebEngine Widgets Module](https://doc.qt.io/qt-5/qtwebengine-overview.html#qt-webengine-widgets-module)的结构图：  

![dictht](/public/image/QtWebEngineWidgetsModule.svg)  
{: align="center"}

[QWebEngineView](https://doc.qt.io/qt-5/qwebengineview.html)处于最上面的一层，提供的API也比较少，功能比较丰富的API在[QWebEnginePage](https://doc.qt.io/qt-5/qwebenginepage.html)这层。  

根据前面的好友列表获取过程，理想的情况就是`弹出浏览器窗口 -> 用户登录成功 -> 页面跳转 -> 浏览器截取目标网络响应 -> 关闭浏览器`，页面跳转可以通过QWebEngineView的`urlChanged()`事件检测到，但没找到合适的方法准确地截取某个请求的响应，只能换一个思路：`弹出浏览器窗口 -> 用户登录成功 -> 页面跳转 -> 隐藏浏览器窗口 -> 使浏览器主动发送目标请求 -> 获取浏览器当前内容 -> 关闭浏览器`，主动发送请求可以调用QWebEngineView的`load()`，获取浏览器当前内容则是用QWebEnginePage的`toHtml()`。  

但在实现时还是遇到了坑。  

## QWebEngineHttpRequest无法设置Referer
主动发送目标请求时需要构造QWebEngineHttpRequest对象，url中的sid可以从登录后跳转url获取，同一个QWebEngineView对象的cookie会自动维护，不需要手动设置，因此就剩下了header中的Referer需要处理。通过QWebEngineHttpRequest提供的`setHeader()`方法设置Referer，结果请求还是提示”禁止GET方法调用“。本地`nc -l -p 8000`进行监听，代码中改为请求本地，发现设置的Referer无效，请求头中并没有Referer项，换成随意的其他项，如`setHeader("abc", "123")`，却可以正常携带该项。感觉像是bug或者WebEngine自身限制，但未找到具体说明。  

不过在找相关说明的过程中找到了一个解决办法[Referrer HTTP Header no longer ignored when set via RequestInterceptor](https://code.qt.io/cgit/qt/qtwebengine.git/commit/?id=1a8e93c95de92f6a00bdf3768c5315dd032513c0)，通过QWebEnginePage的`setUrlRequestInterceptor()`设置QWebEngineUrlRequestInterceptor拦截发出的目标请求，在这里加上Referer，可以成功发送并得到响应。

## toHtml()返回空字符串
获取当前页面上的内容，QWebEnginePage提供了`toHtml()`和`toPlainText()`两个函数，`toHtml()`读取原始html，`toPlainText()`会忽略掉所有html格式。需要注意的是这两个函数的用法有点特殊，他们都是异步函数，返回值为空，需要传入一个回调函数，通过回调函数获得从当前页面读取到的内容，最方便的写法就是在这里使用lambda表达式，例如：  
```c++
this->page()->toHtml([](const QString &content) {
    qDebug() << content;
}
```
但是运行之后发现`qDebug()`的输出一直为空，在stackoverflow看到一个例子是将QWebEnginePage的`loadFinished()`事件绑定到了一个包含`toHtml()`的函数上，尝试之后发现有效。也就是说`toHtml()`并不会判断页面是否加载完毕，需要在合适的时间调用才可以成功读取。  
修改完毕后，发现`toHtml()`偶尔还是会返回空字符串，于是尝试在接收到`loadFinished()`事件后使用QTimer设定一个定时器，在100ms后再读取内容，此时不再出现空字符串的情况。

## 示例代码
WebEngineView.h
```c++
#ifndef WEBENGINEVIEW_H
#define WEBENGINEVIEW_H

#include <QTimer>
#include <QWebEngineView>
#include <QWebEngineProfile>
#include <QWebEngineCookieStore>
#include <QWebEngineUrlRequestInterceptor>

class MyInterceptor : public QWebEngineUrlRequestInterceptor
{
    Q_OBJECT
public:
    MyInterceptor(QObject *p = nullptr):QWebEngineUrlRequestInterceptor(p){}
    void interceptRequest(QWebEngineUrlRequestInfo &info) {
        if(info.requestUrl().toString().contains("https://mail.qq.com/cgi-bin/laddr_lastlist"))
            info.setHttpHeader("Referer", "https://mail.qq.com/");
    }
};

class WebEngineView : public QWebEngineView
{
    Q_OBJECT
public:
    WebEngineView(QWidget *p = nullptr):QWebEngineView(p){}
    void run(void) {
        this->page()->profile()->cookieStore()->deleteAllCookies();
        MyInterceptor *interceptor = new MyInterceptor(this);
        this->page()->setUrlRequestInterceptor(interceptor);
        connect(this, SIGNAL(urlChanged(const QUrl &)),
                this, SLOT(urlChangedSlot(const QUrl &)));
        connect(this, SIGNAL(loadFinished(bool)),
                this, SLOT(loadFinishedSlot(bool)));
        this->load(QUrl("https://mail.qq.com/"));
        this->show();
    }

private slots:
    void urlChangedSlot(const QUrl &url) {
        QString u = url.toString();
        if(u.contains("https://mail.qq.com/cgi-bin/frame_html")) {
            int from = u.indexOf("sid=") + 4;
            QString sid = u.mid(from, u.indexOf("&", from) - from);
            this->load(QUrl("https://mail.qq.com/cgi-bin/laddr_lastlist?sid="
                            + sid + "&t=addr_datanew&category=hot"));
        }
    }
    void loadFinishedSlot(bool ok) {
        if(ok&&this->url().toString().contains("https://mail.qq.com/cgi-bin/laddr_lastlist"))
            QTimer::singleShot(100, this, SLOT(getContentSlot()));
    }
    void getContentSlot() {
        this->page()->toPlainText([](const QString &content) {
            qDebug() << content;
        });
    }
};

#endif // WEBENGINEVIEW_H
```

main.cpp
```c++
#include <QApplication>
#include "WebEngineView.h"

int main(int argc, char* argv[]) {
    QApplication a(argc, argv);
    WebEngineView w;
    w.run();
    return a.exec();
}
```

test.pro
```Makefile
QT      += core gui widgets webenginewidgets
CONFIG  += c++11
SOURCES += main.cpp
HEADERS += WebEngineView.h
```