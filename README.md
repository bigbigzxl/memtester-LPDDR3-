# memtester-LPDDR3-
how to test LPDDR3 cell.
# memory FT测试  --目录


[**先看下DRAM大概啥样子**](https://www.techbang.com/posts/18381-from-the-channel-to-address-computer-main-memory-structures-to-understand?page=1)

![哈哈，放错了，这是内部结构图，有助于你理解它内部的寻址](http://upload-images.jianshu.io/upload_images/4749583-ad01d1c8e0782291.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 目录
**0x00：我要干嘛？**
+ memory测试


**0x01：测试项俯瞰**

从原厂的角度来设计测试项。

+ 功能：全空间扫描目的是扫描坏块，memtester为主；
+ 性能：频率，IO性能，时序，电压调节等；
+ 功耗：IDD2和IDD6，一个standby和一个读写，其余的IDD状态都是瞬态的，当前测试架构无法支持测试；
+ 温度：在高温下的性能情况主要是burn-in-test；
+ 刷新时间：刷新时间过低会不会掉数据，这个应该包括在FaultModel里面，也就是testPatern里面。


**0x02：测试框架**

上位机通过通信接口发送指令到多个测试板，测试板负责测试IC，将结果通过通信接口返回到上位机PC，上位机处理数据之后将结果显示。

+ 上位机显示界面，基于PC WINDOWS,用C#编写的界面；
+ 通信接口采用的是串口，9600波特率，用串口转网口设备，理论上1台PC可带无限块测试板；
+ 下位机采用的ARM的A53处理器作为测试控制器，单片机作为辅控制器；

**0x03：测试算法**

分为三个层次：底层物理缺陷层，中间功能层，以及性能层；
+ FaultModel：这是有一系列的论文、背景支持的，我们最终采用的是精简的失效模型来设计算法；
+ 功能层就是测一下每个cell功能是否正常，常规做法就是直接memtester，但是跟底层貌似有些重合，需确认；
+ 性能层就是我们在改变电压、时序等参数的时候，你跑pattern能跑到多高的频率。

**0x04：测试算法加速**
+ 1、从一些栗子（ARM 官网memcpy）入手，讲如何加速，引入neon及GPU；
+ 2、neon实例分析就是我们的memtester底层加速；
+ 3、neon使用手册学习；
+ 4、安卓下的JNI/NDK使用，直接把c库给应用层用，已CNN为栗子实践及对比官方库学习；
+ 5、CUDA学习；
