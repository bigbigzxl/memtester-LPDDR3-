﻿
# 0x01 前言
系统里面经常需要大量地搬运数据，一般调用的都是memcpy() C库来实现，因此本着“揪牛角尖”的精神，我们就来探究探究加速方案！毕竟很多事情被分解到底层之后就是一样的呢！

加速这个玩意，其实是跟很多因素相关的，因此我们要就环境来论加速，把当前环境考虑进去然后再设计出合理的优化策略，这才是万全之策！也即鲁棒性很强的策略。
 
这里我们从上而下地谈谈memcpy的优化问题，**一个问题可以如此优化，那么相比其他问题也都是类似罢！这怕就是深度学习之迁移学习的由来了！这个世界真奇妙！**

# 0x02 测试环境

我们在ARM Cortex-A8 环境下进行一系列的测试。 

测试时基于源/终地址以及我们要读写的byte数都是L1（64byte）的倍数；

我们需要考虑下对齐，但是这里只测了16MB，影响不大故不考虑；

测试时间是由处理器内部的性能寄存器记录的；

所有的测试中，L1 NEON 位被激活，这意味着当我们使用Neon 加载（load）指令的时候，会使得L1数据缓存进行linefill操作；

我们把分别对应指令和数据的L1、L2缓存使能，同时MMU核分支预测同样也被使能了。

有些地方甚至还使用到了PLD预取指令：这条指令会使得L2缓存在这个数据被使用前的某个时间开始加载数据，它提前发出了内存请求，所以CPU就不需要在那傻等memory把数据给吐出来了！

# 0x03 测试策略
## 策略1: 傻搬
傻搬就是使用常规汇编指令一个字一个字地拷贝。

如下汇编代码所示， 我们每次把地址值加个4，然后不断循环直至搬运结束，这是最基本最常规的一种操作了，因此我们 把这个时间作为一个baseline.

汇编代码如下：
``` cpp
WordCopy
      LDR r3, [r1], #4
      STR r3, [r0], #4
      SUBS r2, r2, #4
      BGE WordCopy
```
## 策略2: 多加载指令

我们之前是只用到了LDR指令，一次搬运32bit也就是1个word，由于只用到了r0～r3寄存器，因此每次调用的时候无需进行堆栈操作；

这里我们使用LDM核STM指令（M是Multiple的简多，意味着一次操作多个），**每个迭代过程中能够操作8word数据**，由于额外寄存器的使用，因此我们需要有现场保护操作，也就是入栈出栈操作。
```cpp
LDMCopy
      PUSH {r4-r10}
LDMloop
      LDMIA r1!, {r3 - r10}
      STMIA r0!, {r3 - r10}
      SUBS r2, r2, #32
      BGE LDMloop
      POP {r4-r10}
```
>注：
1、 r0～r3一般作为函数的局部变量，传入的函数的参数按照顺序分给他们四个，超出的就要进入堆栈区了，其中**r0一般还会作为函数返回值变量**。
2、 r1!表示从r1这个地址处连续搬数据至r3到r10，每搬一个，r1的值就会自动加4。

## 策略3: NEON搬运
 常规的NEON搬运，具体指令啊信息啊什么的，怎么操作啊，去看我上一篇博客吧！
```cpp
NEONCopy
      VLDM r1!, {d0-d7}
      VSTM r0!, {d0-d7}
      SUBS r2, r2, #0x40
      BGE NEONCopy
```
>这里一个d寄存器就是64bit，2个word，8个d寄存器搬运的是16个word了啊！报告老板，有人开挂！

## 策略4: 傻搬+预取
  如题。
+ PLD的意思是我从r1地址处开始预先取出256byte数据到cache里面；
+ r12表示的是 我要搬运16次，每次都是4个byte，共计搬运64byte每轮；
+ 然后每轮结束后，把r2中的计数器减去64byte（0x40）开始下一轮直至结束；

 ```cpp
WordCopyPLD
      PLD [r1, #0x100]
      MOV r12, #16
WordCopyPLD1
      LDR r3, [r1], #4
      STR r3, [r0], #4
      SUBS r12, r12, #1
      BNE WordCopyPLD1
      SUBS r2, r2, #0x40
      BNE WordCopyPLD
```
>同样的tips：这里每轮预取取多了！

## 策略5: 多加载+预取
这里的优势是。。。如题～

  ```cpp
LDMCopyPLD
      PUSH {r4-r10}
LDMloopPLD
      PLD [r1, #0x80]
      LDMIA r1!, {r3 - r10}
      STMIA r0!, {r3 - r10}
      LDMIA r1!, {r3 - r10}
      STMIA r0!, {r3 - r10}
      SUBS r2, r2, #0x40
      BGE LDMloopPLD
      POP {r4-r10}
```

## 策略6: NEON + PLD
+ 预取192byte；
+ d0～d7共计8x64bit=64byte

**这里计算刚刚好，都很完美自洽！**

 ```cpp
NEONCopyPLD
      PLD [r1, #0xC0]
      VLDM r1!,{d0-d7}
      VSTM r0!,{d0-d7}
      SUBS r2,r2,#0x40
      BGE NEONCopyPLD
```

## 策略7: Mixed ARM and NEON memory copy with preload
也就是说把各种指令穿插在一起组合处一个“多元体”来试验看看是不是会速度更快咯～

```cpp
ARMNEONPLD
      PUSH {r4-r11}
      MOV r3, r0
ARMNEON
      PLD [r1, #192]
      PLD [r1, #256]
      VLD1.64 {d0-d3}, [r1@128]!
      VLD1.64 {d4-d7}, [r1@128]!
      VLD1.64 {d16-d19}, [r1@128]!
      LDM r1!, {r4-r11}
      SUBS r2, r2, #128
      VST1.64 {d0-d3}, [r3@128]!
      VST1.64 {d4-d7}, [r3@128]!
      VST1.64 {d16-d19}, [r3@128]!
      STM r3!, {r4-r11}
      BGT ARMNEON
      POP {r4-r11}
```

## 测试时间结果
测试算法|时间花销（ms）|加速比
---|---|---
傻搬|104.8|100%
多加载（指令）|94.5|111%
NEON搬|104.8|100%（说明等待时间占主要比例）
傻搬+预取|137.5|76%
多加载+预取|106.6|98%
NEON+预取|70.2|149%
指令大杂烩|93.5|112%

小结：有一些奇怪的结论。

多加载指令仅仅提升了11%的性能，但是我们没有那么多指令了啊同时指令少就代表分枝预测里面的分支较少啊！原因是：
+ 指令cache100%击中，因此取指令是无须等待的；
+ 分枝预测在这里也不需要预测傻啊！
+ 单个写（一个接一个地写），memory system也把它当成突发写了，所以说效率并没有显著提升；

NEON指令居然没有提升读写速度：
+ 读写循环的执行使用的寄存器很少，因此存在寄存器数据冲突的可能性就小；
+ 因为寄存器用的少，因此特别适合搬小数据块，因为我们不需要堆栈操作来恢复现场啊！！！
+ Cortex-A8处理器可以配置NEON加载数据的时候加载到L2 cache；可以防止内存copy过程中把L1中的不用数据给替换掉；（？？？？）

尽管上面bb了一大堆好处，但是实践证明效果不咋地！
（这里存疑，我在A53平台试验过，大概会块三倍的样子啊！除非时间是异或操作的时间！？）


PLD可以使得内存控制器在数据被使用前就取到；因此加上NEON如虎添翼。
+ 其次，我们知道在burst传输的时候，第一次接入的延时是很大的，因此我们可以发起多次地请求到控制器（当然得控制器够先进够高级），这样子控制器就会把后面的请求合到一起，从而把后面每一次的请求的接入延时相当于去掉了，第一个access latency均摊给每个request之后也忽略不计了，这样子好高效啊！


## 影响内存拷贝速度的因素

### 因素1: 要拷贝的数据量
有些实现需要一定的准备时间，然后搬起来了就老快了。

因此搬运大数据块的时候，可以把建立准备的时间均摊，因此还好，但是搬运小块数据的时候就不划算了哟！

比如：在函数的开始stacking许多的寄存器，然后在主循环里面使用LDM跟STM指令来操作多个寄存器；

## 因素2: 对齐Alignment

ARM架构搬运word对齐的数据会更高效；

 courser alignment granularities这玩意能支撑性能；

多加载指令在Cortex-A8 能每个周期从L1 cache加载2个寄存器的值，但是只有地址是64位对齐的时候才可以。**（所以NEON虽然一次可以搬运128bit，但是你的CPU 位宽，DRAM位宽都是32bit的，因此数据最终还是被拆分成一个一个的32bit再存储的，而我们的程序确实加速了是因为异或操作那部分时间加速了啊！）**


cache对齐也是有影响的啊！
 `Cache behaviour (discussed later) can affect the performance of data accesses depending on its alignment relative to the size of a cache line. For the Cortex-A8, a level 1 cache line is 64 bytes, and a level 2 cache line is 64 bytes.
`
## 因素3: Memory特性
这里讨论的操作（见上述诸程序）其性能瓶颈在存储的接口部分。

因为我们的循环很小啊，所以指令cache的就很好了，而且并没有计算部分（**注意了，我们March C-代码部分优化是有数学运算的，因此这部分比较费时**），因此处理器的逻辑计算部分压力并不大，因此速度因素极大地落在**存储器的速度**上了！

特定种类的memory在某种特定的读写模式下会性能更优，比如SDRAM的burst传输需要一个很长的延时来完成初始化操作，但是一旦操作完成就能很快地完成后续的读写操作；（**我的MARCH算法加速部分把burst传输模式考虑进去**）

此外一个好的memory控制器是能够**并行**接受很多读写请求的，并把这个initial latency给均摊掉。

此外一些特定的代码读写顺序也可能改善性能；

## 因素4: cache的使用
大量数据搬运的时候，很显然会把cache里面的数据全部“换血”的；

尽管这在内存自身拷贝的时候不会有什么影响，但是它可能会减速后面的代码，最终降低整体的性能。

## 因素5: Code dependencies
在标准的 memcpy()函数运行时，尤其遇上慢速的memory时，处理器大部分时间都没有被使用。

因此我们可以考虑在memcopy期间运行一些其他的代码；

因为memcpy（）时阻塞的，因此只有函数结束才会返回，而此时cpu时被占死了；
我们可以使用管道来实现，把memcpy()放倒后台运行，然后通过poll或者中断来随时监控内存搬运的情况
+ 使用DMA操作，这样完全解放CPU了；并把数据块打碎这样就能一边搬运一遍操作了！效率提高了呢！（都是一些很常规的想法呢！）

+ cortex-A8内置的预加载引擎
数据预加载到L2 cache；

我们CPU先启动预加载指令，然后就去干别的活，直到接到电话（中断）说加载完成了，那么我就可以去对它进行操作了，操作完之后继续下一轮；

+ 使用其他处理器
  内嵌的其他核原理同DMA；


# 附录 PLD基本思想及应用考量
当我们在安卓平台需要处理图像数据的时候，其中一个基本的操作就是把大量的数据从内存搬来搬去，相对于CPU来说这个很耗时间，除了NEON加速之外，上面提到的PLD加速也是很有效的，具体多有效我们看实例。

我们比如在处理摄像头数据的时候，内存中搬数据一般是这样写的：

```cpp
while (n--) {
    *dest++ = *src++;
}
```

在我当前平台上跑了一下，1MB的空间搬完需要25ms的样子，这个就很费时了，**究其根本，是因为数据不在处理器的cache中，因此CPU需要花时间来等你DRAM传过来，也就是说，是我当前的CPU带宽大于存储器的带宽了**。

因此**解决方案**就是我们提前把数据放到cache中，由于cache带宽大于DRAM小于CPU，因此可以减少等待时间。


改进后的代码如下所示，提前预取数据，也就是在真正搬运数据之前目标数据就被放倒cache中来了，等真正搬数据时，一下子就在cache中hit了，马上走你！效率快很多！
>这里有个小细节，每次只搬运了32bit也就是4byte，但是我们预取了128byte，这样子岂不是cache很快就溢出了？关于溢出CPU会怎么处理，我就没深究了，这也不是这段代码的主旨，权当留下我自己的一个思考吧！

这个酒厉害了，时间变为8ms了，**提速了三倍之多！**

```cpp
while (n--) {
    asm ("PLD [%0, #128]"::"r" (src));
    *dest++ = *src++;
}
```

当然了，优化这种事情时做不完的，你可以随着环境的变换不断优化的：
+ 比如这里，我们预取后存在溢出问题，也就是说没有物尽其用；
+ 其次，循环里面就一句搬运指令，然后就是n--操作，判断操作等，这样子来看的话一个loop里面真正干活的指令占的比例很小，因此也是一种浪费，把资源浪费在一些不重要的事情上了！

算法里面又个概念，就是时间跟空间是可以互相转换的，在这里的表现就是我通过多写一些代码操作，从而将有效操作时间在每个loop中的比例提升上来。

如下所示，我每个loop里面增加3个搬运操作，这样就保证了cache不会溢出，同时每个loop中数据搬运部分所占的CPU时间比例提升了，也即**“有效功率”**提升了！

这样子做大概又加快了1ms的样子！

```cpp
n /= 4; //assume it's multiple of 4
while (n--) {
    asm ("PLD [%0, #128]"::"r" (src));
    *dest++ = *src++;
    *dest++ = *src++;
    *dest++ = *src++;
    *dest++ = *src++;
}
```

## 思路总结
这个世界是由基本的元素组成的，操作系统是由一些基本的门电路架起来的，雪花是分形的，因此我们总能在深入一个案例后，发现一些普世的道理！共勉！

我说过：“优化时可以随着环境的变化一直做的！”（名言啊！记住啊！划重点！要考的！）

这里不是有四个搬运操作嘛！编译器优化后也不知道会优化成啥样子，我们可以直接用汇编嘛！汇编里面的四个LDR/STR指令，然后是LDRM/STRM可以节省三次CPU指令操作时间及等待时间，然后就是NEON的并行加速了！哎呀呀～不得了！只会越来愈快！具体多快！你自己去实验咯！
