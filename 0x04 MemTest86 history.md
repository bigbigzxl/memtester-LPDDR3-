
### MemTest86 History - from 1994
MemTest86 was **originally** developed by Chris Brady (BradyTech Inc) with a first release in 1994.
 
However, some of the **testing algorithms** used have been under development since 1981 and have been previously implemented on Dec PDP-11, VAX-11/780 and Cray XMP architectures. 

Since then there has been more than a dozen new versions being released. Support for 64bit, new CPU types, **symmetrical multiprocessors** and many other features have been added during this period. 

MemTest86 was released as **free** open source (GPL) software.

### MemTest86 and MemTest86+
被不同人发展出来了一系列的版本，所谓是百花齐放百家争鸣且螺旋式优化改进啊~


### MemTest86 the new era
新时代（64bit、DDR4）的到来，面临新的挑战。
In Feb 2013, PassMark Software took over the maintenance of the MemTest86 project from Chris. 

This was around the time that a lot of technological changes were occurring. The 64bit era was here, DDR4 was coming, UEFI had already arrived and Microsoft's Secure boot technology threatened to prevent MemTest86 from booting on future PC hardware.

Starting from MemTest86 v5, the code was re-written to support self booting from the newer UEFI platform. UEFI is able to provide additional services that is unavailable in BIOS, such as graphical, mouse and file system support. Support for DDR4 & 64bit were also added and Microsoft agreed to code sign MemTest86 for secure boot. 

The software (Free Edition) still remains free to use without restrictions. The MemTest86 v4 project is still maintained and remains open source, for use on old machines with BIOS. From V5 however the software is being released under a proprietary license. For advanced/enthusiast users or commercial applications, a professional version is available for users that require additional customizability and advanced features that may be more suitable for their testing needs. A comparison of the different versions can be found here. We have also created a support forum where users can discuss issues.


[这里是它的发布历史轨迹](https://www.memtest86.com/support/ver_history.htm)
##  MemTest86定义
MemTest86 is the original, free, stand alone memory testing software for x86 computers. MemTest86 boots from a USB flash drive or CD and tests the RAM in your computer for **faults** using a series of **comprehensive algorithms and test patterns**.
>指出：首先是免费，其次是使用了一些测试**pattern**跟**综合算法**来测试内存的**缺陷**。

特点：（就关注这两点）
+ 13 different RAM testing algorithms
+DDR4 RAM (and DDR2 & DDR3) support
+ 测试算法发展了20年了，初代目早在1994年就出现了

### 这是memtest86的测试项：

By running MemTest, you can ensure that your computers RAM is **correctly functioning**. 
>只能测功能；

Unlike other memory checking software, MemTest is designed to **find all types of memory errors** including intermittent problems.
>找到所有的内存缺陷； 

Therefore, it needs to be run for **several hours** to truly evaluate your RAM. MemTest works with any type of memory.
>代价就是要测好几个小时；

**Test 0 [Address test, walking ones, no cache]**
Tests all address bits in all memory banks by using a walking ones address pattern.
>walking ones address pattern测试地址位；

**Test 1 [Address test, own address]**
Each address is written with its own address and then is checked for consistency. In theory previous tests should have caught any memory addressing problems. This test should catch any addressing errors that somehow were not previously detected.
>补充第一个测试缺陷的；

**Test 2 [Moving inversions, ones&zeros]**
This test uses the moving inversions algorithm with patterns of all ones and zeros. Cache is enabled even though it interferes to some degree with the test algorithm. With cache enabled this test does not take long and should quickly find all "hard" errors and some more subtle errors. This test is only a quick check.
>快速检查出异常使用moving inversions algorithm with patterns of all ones and zeros.

**Test 3 [Moving inversions, 8 bit pat]**
This is the same as test one but uses a 8 bit wide pattern of "walking" ones and zeros. This test will better detect subtle errors in "wide" memory chips. A total of 20 **data patterns** are used.
>算法同第一个测试项，但是使用的是8bit位宽，能检测更微小的内存错误；

**Test 4 [Moving inversions, random pattern]**
Test 4 uses the same algorithm as test 1 but the data pattern is a random number and it's complement. This test is particularly effective in finding difficult to detect **data sensitive errors**. A total of 60 patterns are used. The random number sequence is different with each pass so multiple passes increase effectiveness.
>找出数据敏感型异常（基于随机数）；

**Test 5 [Block move, 64 moves]**
This test stresses memory by using block move (movsl) instructions and is based on Robert Redelmeier's burnBX test. Memory is initialized with shifting patterns that are inverted every 8 bytes. Then 4mb blocks of memory are moved around using the movsl instruction. After the moves are completed the data patterns are checked. Because the data is checked only after the memory moves are completed it is not possible to know where the error occurred. The addresses reported are only for where the bad pattern was found. Since the moves are constrained to a 8mb segment of memory the failing address will always be less than 8mb away from the reported address. Errors from this test are not used to calculate BadRAM patterns.
>内存块移动方式来进行压力测试；

**Test 6 [Moving inversions, 32 bit pat]**
This is a variation of the moving inversions algorithm that shifts the data pattern left one bit for each successive address. The starting bit position is shifted left for each pass. To use all possible data patterns 32 passes are required. This test is quite effective at detecting data sensitive errors but the execution time is long.

**Test 7 [Random number sequence]**
This test writes a series of random numbers into memory. By resetting the seed for the random number the same sequence of number can be created for a reference. The initial pattern is checked and then complemented and checked again on the next pass. However, unlike the moving inversions test writing and checking can only be done in the forward direction.
>写一个序列的随机值进内存，只能一个方向；

**Test 8 [Modulo 20, ones&zeros]**
Using the Modulo-X algorithm should uncover errors that are not detected by moving inversions due to cache and buffering interference with the the algorithm. As with test one only ones and zeros are used for data patterns.
>移动逆转由于cache的作用可能会覆盖不全某些异常，因此Modulo-X算法补充之；

**Test 9 [Bit fade test, 90 min, 2 patterns]**
The bit fade test initializes all of memory with a pattern and then sleeps for 90 minutes. Then memory is examined to see if any memory bits have changed. All ones and all zero patterns are used. This test takes 3 hours to complete. The Bit Fade test is not included in the normal test sequence and must be run manually via the runtime configuration menu.
>睡90分钟后看是不是数据掉了。

---

## 异常显示
Error Display
Memtest has two options for reporting errors. The default is to report individual errors. Memtest is also able to create patterns used by the Linux BadRAM feature. This slick feature allows Linux to avoid bad memory pages. Details about the BadRAM feature can be found at: http://home.zonnet.nl/vanrein/badram

For individual errors the following information is displayed when a memory error is detected. An error message is only displayed for errors with a different address or failing bit pattern. All displayed values are in hexadecimal.

Tst: Test Number

Failing Address: Failing memory address

Good: Expected data pattern

Bad: Failing data pattern

Err-Bits: Exclusive or of good and bad data (this shows the position of the failing bit(s))

Count: Number of consecutive errors with the same address and failing bits Error Display

##  问题定位

Troubleshooting Memory Errors
Please be aware that not all errors reported by Memtest86 are due to bad memory. 
The test implicitly tests the **CPU, L1 and L2 caches as well as the motherboard. **

>讲到这算发测出异常了大概率是memory出问题了，小概率是是板子啊CPU啊cache啊出问题了。

It is impossible for the test to determine what causes the failure to occur. However, most failures will be due to a problem with memory. When it is not, the only option is to replace parts until the failure is corrected.

出问题时按下面步骤来定温问题：
there are steps that may be taken to determine the failing module. Here are four techniques that you may wish to use:

1) Removing modules

2) Rotating modules

3) Replacing modules

4) Avoiding allocation

兼容问题导致异常。

Memtest86 can not diagnose many types of PC failures. For example a faulty CPU that causes Windows to crash will most likely just cause Memtest86 to crash in the same way. 
>就是用户用来检测内存是否可用的，系统是否有异常的。

## 执行时间

The time required for a complete pass of Memtest86 will vary greatly depending on CPU speed, memory speed and memory size. Here are the execution times from a Pentium-II-366 with **64mb** of RAM: 
Test 0 0:05
Test 1 0:18
Test 2 1:02
Test 3 1:38
Test 4 8:05
Test 5 1:40
Test 6 4:24
Test 7 6:04
**Total (default tests) 23:16**

Test 8 12:30
Test 9 49:30
Test 10 30:34
Test 11 3:29:40
**Total (all tests) 5:25:30**

Memtest86 continues executes indefinitely. The pass counter increments each time that all of the selected tests have been run. Generally a single pass is sufficient to catch all but the most obscure errors. However, for complete confidence when intermittent errors are suspected testing for a longer period is advised.
>为了发现间歇性问题，多测几轮还是很有必要的。

## 内存测试理念
Memory Testing Philosophy
There are many good approaches for testing memory. However, many tests simply throw some patterns at memory without much thought or knowledge of the memory architecture or how errors can best be detected. This works fine for hard memory failures but does little to find intermittent errors.
>大多测试算法没有针对特定的存储架构做优化，因此测出来的都是致命性问题，间歇性的问题一般很难找出来。
 The BIOS based memory tests are useless for finding intermittent memory errors.
>基于BIOS的测试对间歇性异常无能为力。

Memory chips consist of a large array of tightly packed memory cells, one for each bit of data. The vast majority of the intermittent failures are a result of interaction between these memory cells. Often writing a memory cell can cause one of the adjacent cells to be written with the same data. An effective memory test should attempt to test for this condition. Therefore, an ideal strategy for testing memory would be the following:

1) write a cell with a zero
2) write all of the adjacent cells with a one, one or more times
3) check that the first cell still has a zero
>这就是测1被0包围或者0被1包围的临近测试场景、

It should be obvious that this strategy requires an exact knowledge of how the memory cells are laid out on the chip.
>**很明显地我们需要知道内存在实际地三维空间中是如何排列的**。

 In addition there is a never ending number of possible chip layouts for different chip types and manufacturers making this strategy impractical. However, there are testing algorithms that can approximate this ideal.
>布局不同测试算法就不同，因此我们很难搞适配啊！还好，有算法能搞定这个问题。

## 临近耦合缺陷测试算法

Memtest86 uses two algorithms that provide a reasonable approximation of the ideal test strategy **above.** 

### moving inversions算法

解决相邻cell互相影响的问题。

The first of these strategies is called moving inversions. The moving inversion test works as follows:

1) Fill memory with a pattern
2) Starting at the lowest address
2a check that the pattern has not changed
2b write the patterns complement
2c increment the address
repeat 2a - 2c
3) Starting at the highest address
3a check that the pattern has not changed
3b write the patterns complement
3c decrement the address
repeat 3a - 3c

>由于上了平台而不是在测试机，因此我们不能针对每个bit，一位一位连续着写（因为我们平台最多支持Byte操作），因此只能折中，把一些pattern组合着写入进去，**少了一些动态过程**。

This algorithm is a good approximation of an ideal memory test but there are some **limitations**. Most high density chips today store data 4 to 16 bits wide. With chips that are more than one bit wide it is impossible to selectively read or write just one bit. This means that we cannot guarantee that all adjacent cells have been tested for interaction. In this case the best we can do is to use some patterns to insure that all adjacent cells have at least been written with all possible one and zero combinations.

>同时caching、buffering都是不会使你的读写操作变连续的，因为会缓存嘛，这就使得本来要连续操作的变成断断续续了，要不得，因此我们整了个Modulo-X 算法来解决这个问题，其实就是一个很常规的操作：**我们把数据填充满cache或者buffer（高级特性无法关闭）这样就会自动触发flush操作，从而不会滞留数据了**。

It can also be seen that caching, buffering and out of order execution will interfere with the moving inversions algorithm and make less effective. It is possible to turn off cache but the memory buffering in new high performance chips can not be disabled. To address this limitation a new algorithm I call Modulo-X was created. This algorithm is not affected by cache or buffering. The algorithm works as follows:

1) For starting offsets of 0 - 20 do
1a write every 20th location with a pattern
1b write all other locations with the patterns complement
repeat 1b one or more times
1c check every 20th location for the pattern

This algorithm accomplishes nearly the same level of adjacency testing as moving inversions but is not affected by caching or buffering.
>于是就实现了adjacency testing as moving inversions，且不受caching or buffering的影响。

 Since separate write passes (1a, 1b) and the read pass (1c) are done for all of memory we can be assured that all of the buffers and cache have been flushed between passes. The selection of 20 as the stride size was somewhat arbitrary. 

>大的步进会使得更高效，但是会使得测试时长加大，因此折中速度和吞吐率.(**我们是全空间读写，因此这里可以设置大一点哟！**)

Larger strides may be more effective but would take longer to execute. The choice of 20 seemed to be a reasonable compromise between speed and thoroughness.


## 单项测试描述（静态异常？）

Memtest86 executes a series of numbered test sections to check for errors. 

These test sections consist of a combination of **test algorithm, data pattern and cache setting**. 

The **execution order** for these tests were arranged so that errors will be detected as rapidly as possible. 

Tests 8, 9, 10 and 11 are very long running extended tests and are only executed when **extended testing** is selected. 

The extended tests have a low probability of finding errors that were missed by the default tests. 

A description of each of the test sections follows:

#### Test 0 [Address test, walking ones, no cache]

Tests all address bits in all memory banks by using a walking ones address pattern.

#### Test 1 [Moving Inv, ones&zeros, cached]

This test uses the moving inversions algorithm with patterns of only ones and zeros.

 Cache is enabled even though it interferes to some degree with the test algorithm. With cache enabled this test does not take long and should quickly find all "hard" errors and some more subtle errors. This test is only a quick check.

#### Test 2 [Address test, own address, no cache]

Each address is written with its own address and then is checked for consistency. 

In theory previous tests should have caught any memory addressing problems. This test should catch any addressing errors that **somehow** were not previously detected.

#### Test 3 [Moving inv, 8 bit pat, cached]

This is the same as test one but uses a 8 bit wide pattern of "walking" ones and zeros. This test will **better detect subtle errors in "wide" memory chips.** 

**A total of 20 data patterns are used.**

#### Test 4 [Moving inv, 32 bit pat, cached]

This is a variation of the moving inversions algorithm that shifts the data pattern left one bit for each successive address. 

The starting bit position is shifted left for each pass. To use all possible data patterns **32 passes** are required. 

This test is effective in detecting** data sensitive** errors in "wide" memory chips.

#### Test 5 [Block move, 64 moves, cached]

This test stresses memory by using block move (movsl) instructions and is based on Robert Redelmeier's burnBX test. 

Memory is initialized with shifting patterns that are inverted every 8 bytes. Then 4mb blocks of memory are moved around using the movsl instruction. 

After the moves are completed the data patterns are checked. Because the data is checked only after the memory moves are completed it is not possible to know where the error occurred. 

The addresses reported are only for where the bad pattern was found. Since the moves are constrained to a 8mb segment of memory the failing address will always be less than 8mb away from the reported address. Errors from this test are not used to calculate BadRAM patterns.

#### Test 6 [Modulo 20, ones&zeros, cached]

Using the Modulo-X algorithm should uncover errors that are not detected by moving inversions due to cache and buffering interference with the the algorithm. As with test one only ones and zeros are used for data patterns.
#### Test 7 [Moving inv, ones&zeros, no cache]

This is the same as test one but without cache. With cache off there will be much less interference with the test algorithm. However, the execution time is much, much longer. This test may find very subtle errors missed by previous tests.

#### Test 8 [Block move, 512 moves, cached]

This is the first extended test. This is the same as test #5 except that we do more memory moves before checking memory. Errors from this test are not used to calculate BadRAM patterns.

#### Test 9 [Moving inv, 8 bit pat, no cache]

By using an 8 bit pattern with cache off this test should be effective in detecting all types of errors. However, it takes a very long time to execute and there is a low probability that it will detect errors not found by the previous tests.

#### Test 10 [Modulo 20, 8 bit, cached]

This is the first test to use the Modulo-X algorithm with a **data pattern** other than ones and zeros. This combination of algorithm and data pattern should be quite effective. However, it's very long execution time relegates it to the extended test section.

#### Test 11 [Moving inv, 32 bit pat, no cache]

This test should be the most effective in finding errors that are data pattern sensitive. However, without cache it's execution time is excessively long.

测试类型|测试内容描述
---|---
Test 0 [Address test, walking ones, no cache]| walking ones address pattern测试所有的地址位
Test 1 [Moving Inv, ones&zeros, cached] | moving inversions algorithm with patterns of only ones and zeros.
Test 2 [Address test, own address, no cache] | 地址写入，检查其连续性，是test0的补充
Test 3 [Moving inv, 8 bit pat, cached]|重点在8bit，检查粒度更细了
Test 4 [Moving inv, 32 bit pat, cached]| 检测数据敏感性异常；
Test 5 [Block move, 64 moves, cached]|检测压力下内存是否会出现问题；
Test 6 [Modulo 20, ones&zeros, cached]| 去除cache的影响，还是moving inversions
Test 7 [Moving inv, ones&zeros, no cache]| 补充测试未chche情况下是否ok，时间花销显著变大哟！（解决用例覆盖度问题）
Test 8 [Block move, 512 moves, cached]| 既然加压了，那就试试不同压力下的情况吧！
Test 9 [Moving inv, 8 bit pat, no cache]|理论上检测所有缺陷，因为粒度都到8bit了，但是时间花销上也是呵呵哒！
Test 10 [Modulo 20, 8 bit, cached]|将data pattern（之前的就是0啊1啊的pattern而已）跟去除cache影响的Modulo-X算法组合技能确实效率更高，但是执行时间也是长长的~
Test 11 [Moving inv, 32 bit pat, no cache]|我们用的是32bit的数据pattern，因此在找出数据敏感型的异常情况中最有效，但是由于没有cache这个速度也是很感人。

>**总结**：就是从用户的角度来看，设立不同的测试场景即测试用例，然后针对性地进行**功能**测试，注意是从系统级来测试，也就是说关注的不仅仅是内存颗粒了，而是在系统板级的连线、IO性能、PCB等等相关的因素一同考虑进去后，你的memory是否还能**功能**正常；
