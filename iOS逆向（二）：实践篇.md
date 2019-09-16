## 1、先定一个小目标：修改查找好友功能
比如，我们想修改的功能是：在查找好友页面，输入特定字符串  (如: 0921)，然后我们在请求发起之前，改成 "130 xxxx xxxx"，再去进行真实的好友手机号请求。别问我为什么hook这个功能，问就是玄学（某些教育类公司就给招生老师装过有类似功能的wx）。

#### 1.1 查找目标入口
通过XCode 自带的View Capture，我们可以查看UI层级：顺便根据类名，向要hook的目标类靠近。
![UI层级一目了然](https://img-blog.csdnimg.cn/20190913192228279.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)

首先看到的关键字，就是**WCSearchController**，貌似搜索逻辑是在这里进行的吧？移步**class-dump**导出的头文件，看看有没有相关功能的函数：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190913192330840.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
然而，看完函数名很失望。。显然wx的封装程度，和代码的优秀程度，不是那么简单的往VC里堆叠逻辑，他们有着复杂的设计。不要紧，我们还有两条线索：
1：**查看其他VC**：UI界面还有个VC叫**AddFriendEntryViewController**
2、**猜测** 函数名：搜索按钮被点击，应该有个**searchButtonClicked:** 类似的方法回调吧？
所以，两条路都试一下：先看AddFriendVC，再在头文件中全局搜searchButtonClicked关键字。
结果如下：
![AddFriendVC](https://img-blog.csdnimg.cn/20190912184912558.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
在这个VC里，我们看到它引用了Class **FindContact...** 大致意思不就是：“**查找联系人**”嘛？先记住这个人。再去看关键字检索：
![关键字检索](https://img-blog.csdnimg.cn/20190912184929364.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
哦吼~ 有点意思了，它实现了这个方法。
那我们就去好好看看 **FindContactSearchViewCellInfo**这个嫌疑类：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912190444284.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
哇，在关键词search的帮助下，我们很快就找到了一些猎物。

像CPU命中了**cache**一样快乐，这个类里实现了搜索按钮点击回调、有搜索命令的实现、有get搜索框文案的实现。

猜的这么准，有什么根据么？有的，请看**杨潇玉**的博客：[如何在逆向工程中Hook得更准](http://yulingtianxia.com/blog/2017/03/06/How-to-hook-the-correct-method-in-reverse-engineering/)
（**PS:** 该同学很厉害。上学期间，自学iOS，反汇编分析很多代码，runtime玩的相当6，其中苹果开源的代码故意隐藏掉一行很重要的函数调用，都被他发现了。后来拿到BAT所有offer，最终去了腾讯安全部门。然而他还是觉得自己当时（实习期间）向身边神一样的牛人们学到了很多东西。哦对了，他还是咱们逆向工具作者**AloneMonkey**的朋友，慢慢的你会发现，你读的很多优秀博客，他们之间都有着~~不可告人~~ 额不，是千丝万缕的联系）

言归正传，我们hook一下这个类试试：
哦对，我们上篇文章提到用CaptainHook，作者的demo里已经有使用方法和基本语法了。
xxxDylib.h 就是声明hook类的地方
xxxDylib.m 就是写你要hook方法的实现。
可以看出，我们的注入代码最终是以动态链接库的原理，注入到wx进程中，等dyld动态链接完成之后，生效的。
验证方法：`(lldb) image list `命令 可以查看镜像动态加载工程。

开始撸代码：
![头文件声明](https://img-blog.csdnimg.cn/20190912190730992.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
**doSearch**方法的复写：这里可以打断点调试，比如输出一下参数什么的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912191352831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
Hook完以后运行，点击搜索按钮，真的走到了这里。证明了我们之前的所有猜测都是正确的，可以进行下一步尝试了——分析代码，修改代码。

#### 1.2 分析并改造目标功能
上篇提到的IDA反汇编工具，在这里就派上了大用场。
查看**doSearch**伪代码：
![doSearch伪代码](https://img-blog.csdnimg.cn/20190913193939160.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
在伪代码中，我们可以看到函数内部调用了：`[r19 getSearchBarText]`。感觉这就是我们要修改的地方啊！

那么我们就分析并重写这个函数吧。该函数大致做了这么几件事：

 - 创建**SearchRequest**
 - 创建**SKBuiltinString** 用于初始化上面的**Request**，初始化参数是从`[self getSearchBarText]`得到的。
 - 创建**ProtobufCGIWrap**（网络请求包）
 - 用之前的**request**初始化网络请求包
 - 创建**Service**
 - 用**Service**创建一个Event，传参是上面创建的**CGIWrap**和一个Flag（多次断点，看到这个值永远是69，猜测是用来区分event类型的枚举）
 - 最后一步，如果上面的**Event**创建成功，就添加监听。入参是一个**unsigned int**类型的ListItem和一个**id**类型的Value。
以上这些伪代码中方法名及参数类型，是在**class-dump**中找到的。

OK分析完了，利用objc/message 和runtime的一些工具，我们开始重写doSearch。代码如下：
![重写doSearch](https://img-blog.csdnimg.cn/20190913200759713.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
Hook也Hook了，代码也重写了，那么接下来就到了见证奇迹的时刻——运行测试！

如果hook生效，那么输入框里输入"0921"，就会搜所"wx_sunhonglei"（这里大家可以换成搜自己的微信，毕竟这个微信名是我自己乱写的，该用户不存在）。

**Cmd+R**，wx起来后，进到添加好友页面，
1、先输入一个正常的微信号进行搜索测试，比如自己的微信号，**bingo**!逻辑走通了！证明我们重写的doSearch方法是没有问题的。
2、测试我们替换字符串的逻辑，输入0921，看看能不能搜索到自己的微信。

**结果**。。。点击搜索以后，界面没有任何变化！甚至连代码中的showLoading貌似也没走！因为没看到loading的菊花。

百思不得其解！怎么办！！是我们doSearch方法写的有问题吗？！还是其他**逻辑**原因？

**能不能看一下函数调用栈？**

恭喜你，问到了逆向的精髓之处。在此之前，我被卡了两星期。就是从这儿起，我开始沉下来学习了汇编原理和iOS的一些底层原理，最终越过山丘。

请继续收看下集： [**iOS逆向（三）：强大的断点调试工具**](https://github.com/OPTJoker/iOS_Reverse/blob/master/iOS%E9%80%86%E5%90%91%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E5%BC%BA%E5%A4%A7%E7%9A%84%E6%96%AD%E7%82%B9%E8%B0%83%E8%AF%95%E5%B7%A5%E5%85%B7.md)
。
