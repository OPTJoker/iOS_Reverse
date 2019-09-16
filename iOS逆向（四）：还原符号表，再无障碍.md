本文的部分理论支持，节选自[这里：iOS符号表恢复](http://blog.imjun.net/posts/restore-symbol-of-iOS-app/)。
## 前言
符号表历来是逆向工程中的“必争之地”，而iOS应用在上线前要裁去符号表，以避免被逆向分析。

这些可以通过配置xcode的编译选项来达到效果。具体操作请看这：[Xcode中和symbols有关的几个设置](https://www.jianshu.com/p/11710e7ab661)。

Xcode显示调用堆栈中符号时，只会显示符号表中有的符号。为了我们调试过程的顺利，我们有必要把可执行文件中的符号表恢复回来。

先来看一眼无符号表和有符号表的可执行文件调试区别：
![无符号表](https://img-blog.csdnimg.cn/20190915131056870.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
![有符号表](https://img-blog.csdnimg.cn/20190915131118132.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
符号表有多好用，一目了然。

## 什么是符号表？
我们要恢复符号表，首先要知道符号表是什么，他是怎么存在于 Mach-O 文件中的？由于这块涉及到的知识较多，还原符号表的作者讲解的很好，所以希望大家认真研究一下[这篇文章](http://blog.imjun.net/posts/restore-symbol-of-iOS-app/)。对 就是开篇提到的链接。

这里是：[项目开源地址](https://github.com/tobefuturer/restore-symbol)。
工具的配置和使用说明，也在作者的博客中。

有了这些工具，再回过头来研究一下，为什么我们在[第二篇文章](https://blog.csdn.net/u012241552/article/details/100778740)中重写了**doSearch**方法，却没有生效的原因。

在**doSearch**打断点的时候，**showLoading**函数已经被执行了，但我们点击搜索没有反应，所以应该是后面调用了`stopLoading`函数。这个函数正好在`FindContactSearchViewCellInfo`类里实现了（在class-dump导出的头文件里看到的）。所以我们就大胆尝试一下，通过`br -n "[FindContactSearchViewCellInfo stopLoading]"`命令，断点该函数。
再次在输入框输入0921，bingo，断点执行了。

从函数堆栈调用顺序（图片出了点问题，就不贴图了，后面我会附上其他app的函数调用栈效果截图），我们可以看到`doSearch`之后，执行了另一个函数：
`- (void)MessageReturn:(id)arg1 Event:(unsigned int)arg2`。
该函数也是`FindContactSearchViewCellInfo`类中的。
移步**IDA**工具，看该函数实现：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019091513381093.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
惊不惊喜，怪不得重写了网络请求的创建，依然搜不到想要的结果。而且我也曾试着重写过wx的`getSearchBarText`方法，仍然不起作用。
原来是消息回来之后，wx拿searchBar.text 跟 request的请求参数（userName就是）做了比对，最终丢掉了这次的请求包。最后又stopLoading了。

我怀疑这是wx来了一个新人，不知道该类有`getSearchBarText`方法，所以直接自己手动取值了。导致正常的微信用起来会有一个bug：点击搜索的瞬间，删除一位搜索框的内容。loading就停不下了。

哦对了，附上别人的符号表还原效果图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915134436814.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
我们断点到`stopLoading`以后，也是这种效果。

至此，调试三方app就跟调试自己的app一样了。

这篇可能有点烂尾，实在是符号表还原的理论知识太多，而且作者讲的我无法超越，只能虚心引用了。

**那么，加油吧！**

