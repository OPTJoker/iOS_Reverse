## 1、强大的lldb
上文我们说到了调试。在iOS逆向中，很多人推荐**debugserver** + **lldb** 其实调试只需要**lldb**就够了。
**debugserver**配置的文章有很多，从14年到18年不等，但大部分都过时了。所以我也没用。那 只用**lldb**该怎么断点调试三方app呢？我们先来简单看下lldb的断点命令。

#### 1.1  LLDB断点命令：
**1.1.1 设置断点** 下断点命令我们只需要先会两个就够了：

**a：** 给指定内存地址下断点： ```br set -a 0x00000000```全拼我记不住
**b：** 给某方法下断点：`b -n "[ClassName methodName]"` 全拼同样记不住
关于**a**，怎么定位方法内存，一会再讲。
关于**b**，很多app打App Store包的时候，是去掉了符号表的编译配置项的。所以，我们暂时打不了三方app的方法断点。不过不用急，后面我们会教大家如何还原三方app的符号表。

**1.1.2** 查看断点

```br list```这个全拼我能记住：breakpoint list

**1.1.3** 删除断点
`br del n` 这里的**n**是上一步`br list`列出的断点的序号。根据对应的序号删除想删的断点。当然你也可以直接`br del`，然后**lldb**会问你是否要删除全部断点。`[Y/n] ?` 输入Y即可。

#### 1.2 如何定位函数地址
好 我们先来看看如何找到想断点的函数的地址。为了降低被黑客攻击的风险，操作系统大都采用**ASLR**(地址空间布局随机化) 技术： [详细解释请看这](https://www.jianshu.com/p/33c9883b647a) 这个人的内存管理讲的不错，一共7篇，建议大家有时间看看。

**ASLR**是一种避免类似攻击的有效保护。进程每一次启动时，地址空间都会被简单地随机化：进行整体的地址偏移，而不是搅乱。通过内核将整个进程的内存空间“平移”某个随机数，进程的基本内存布局如程序文本、数据、库等相对位置仍然是一样的，但其具体的地址都不同了。
简单来说，**ASLR**会在进程启动时候随机一个**基地址**。
在**lldb**中的查看命令是`image list` 或`image list -o -f`。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915033936469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
如过命令后面不加参数，它打印的就是每个image（镜像）的虚拟内存地址。
如果加了-o参数，打印的就是image的偏移量（相对于谁，暂时还没研究）。

具体区别如下：
`image list` 打印的基地址：	0x**1**02a94000
`image list -o -f`的地址：		0x**0**02a94000

看出区别了么？第**9**位差了一个**1**。
这个**1**正好跟**IDA**工具中的0x **1** 0000 0000对上了。
原因我还没去探究，如果有知道的小伙伴，望告知。

这里我们提到的地址是虚拟地址，为什么不是物理地址？
原因是：操作系统将我们（程序猿）跟物理内存划分了一个安全界限，我们程序猿只需要用虚拟内存跟操作系统打交道即可，操作系统调用底层硬件api，跟物理内存打交道。如果程序猿直接跟物理内存交流，恐怕不是那么安全的。程序一旦出了bug，可能会导致系统瘫痪。

**基地址**有了，在前面**ASLR**技术中我们提到内核将整个进程的内存“**平移**”，所以数据段、代码段等等内存的偏移量其实是固定的，他们相对于mach-o header部分的偏移量是个常亮，那既然如此，我们的**IDA**工具又派上用场了：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915024401205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
这里列出的偏移地址，就是函数相对于mach-o的起始地址的偏移量，所以：
**虚拟地址 = 基地址 + 偏移地址**。
那么，我们就可以在**lldb**用`br set -a 0x102a94000`下断点了。
在这里在交给大家两个好用的命令：
读内存命令：`x 0x102a94000` 全拼是：`memory read 0x102a94000`
反汇编命令：`dis -s 0x102a94000`  dis是dissamble的简写。

我们调试别人的app，是看不到源码的，只能看汇编。所以这里建议大家学一下汇编。我知道很多人听到汇编就头大，不过不要紧，找对了教材，汇编真的可以通俗易懂。比如这本：**《汇编语言第3版》王爽著**。学习时长**两三天**就够。不信你试试。
我是从网上下的影印版PDF，阅读效果不太好，大家可以买正版书或者正版PDF来学习。

学完了汇编，CPU工作原理你就基本了解了，再跟内存交流起来就方便多了。不过你可能还是看不懂**xcode**中出现的上古语言。因为两者的汇编指令集不一样，书里的CPU是古老的**8086**，是地址总线**20**位，数据总线**16**位的16位机器。而iPhone 5s之后都是**arm64** CPU，命令不太一样，总线位数也不一样，不过如果你理解了CPU的工作原理，**arm64**其实只是换了一种语法而已。而且CPU寻址操作也变得更简单。毕竟**arm64**数据总线跟地址总线位数相同（皆为64位），CPU不再需要地址加法器计算地址了。

这些都掌握了之后，我们可以看一下**runtime**源码，重点看一下**objc/message**部分。
目前苹果官方最新的是**objc4-762**版本，为了方便调试，我从**github**上下载了**objc4-750**版本，就差一个版本，不影响我们理解原理。
[runtime 非官方Git地址](https://github.com/acBool/RuntimeSourceCode.git)
可以结合[这篇文章](https://www.jianshu.com/p/89713ed70653)进行理解。作者梳理的很好，从objc_msgSend的汇编代码入口开始，梳理到最终的**runtime**消息转发机制。

有了这些基础，下面我们再进行断点调试的时候，就会方便很多。

比如说，我们给没有隐藏符号表的app下断点：
![断点viewDidAppear:](https://img-blog.csdnimg.cn/2019091503312354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)

未隐藏符号表的程序，断点的时候，函数名也会显示出来。

接下来我们将用到一个进阶命令：`register read` 查看寄存器信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915035205586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
前面如果你研究了runtime的objc_msgSend函数，你就会知道，该函数有两个默认参数`(receiver, cmd)`，第一个参数是消息接收者，第二个是函数地址。在**iOS arm64 CPU**中，通用寄存器中的`x0~x7`寄存器 用于参数传递。所已`x0`寄存器的值就是我们该函数的第一个参数：消息接收者，也就是DJHomeViewController类型的一个对象。结合po命令，我们就可以查看很多东西了。
第二个参数是函数地址，对应`x1`寄存器。
objc_msgSend中第三个参数（`x2`寄存器），就是viewDidAppear:后面的传参了，我们看到这个值是0. 回头看看`class-dump`，可以看到这个参数要求传一个布尔值，那么 我们就可以猜出该函数被调用时候，入参是false。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915035829663.png)

有了对象和参数地址，你就拥有了一切。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915040155970.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70 )

比如，我们通过class-dump 看到该类有这么一个属性，那我们就可以直接访问！
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915040558481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIyNDE1NTI=,size_16,color_FFFFFF,t_70)
我们可以用`p`命令 输出一下对象，该功能有点类似于`expression`命令。
此时会生成一个`$`符号开头的变量，这个变量可以在后续**lldb**中使用。
那么接下来我们就可以利用**kvc**进行访问任何内容了。也可以直接像写**OC**代码一样，在**lldb**中写方法调用。例如：`po [$10 class]`;
**lldb**常用命令可以[看这里](https://www.jianshu.com/p/7fb43e0b956a)、[或这里](https://juejin.im/post/5b1cd870e51d4506dc0ac76c)。网上有很多这类文章，大家可以自行查找。

到这里，我们可以看到，假如调试的时候能看到函数名，那我们逆向就没有任何阻碍。所以，符号表是我们的必争之地！
如何还原符号表，请看下集：[iOS逆向（四）：还原符号表，再无障碍](https://github.com/OPTJoker/iOS_Reverse/blob/master/iOS%E9%80%86%E5%90%91%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%9A%E8%BF%98%E5%8E%9F%E7%AC%A6%E5%8F%B7%E8%A1%A8%EF%BC%8C%E5%86%8D%E6%97%A0%E9%9A%9C%E7%A2%8D.md)。
