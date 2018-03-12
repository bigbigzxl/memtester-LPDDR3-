## 前言
为一究memtester原理，现对其每个函数均按照如下格式进行描述：
+ 方法
+ 原理
+ 时间花销


以下是对每个测试项的简要描述：

memtester-4.3.0 | memtester-ARM 
---|---|
int test_stuck_address(bufa, count);|（√ ） 先全部把地址值交替取反放入对应存储位置，然后再读出比较，重复2次（官网的重复了16次）：测试address bus |
int test_random_value(bufa, bufb, count); |（√ ）等效test_random_comparison(bufa, bufb, count)：数据敏感型测试用例 |
 int test_xor_comparison(bufa, bufb, count);|（-） 与test_random_value比多了个异或操作，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF； | 
int test_sub_comparison(bufa, bufb, count); |（-）与test_random_value比多了个减法操作，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF；  | 
int test_mul_comparison(bufa, bufb, count); | （-）与test_random_value比多了个乘法操作，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF；| 
int test_div_comparison(bufa, bufb, count); | （-）与test_random_value比多了个除法操作，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF；| 
int test_or_comparison(bufa, bufb, count); | （√ ）在test_random_comparison()里面合并了，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF； | 
int test_and_comparison(bufa, bufb, count); | （√ ）在test_random_comparison()里面合并了，用户场景之一，用例覆盖。数据敏感/指令功能验证，同时可验证SAF；|
int test_seqinc_comparison(bufa, bufb, count); | （√ ）这是 test_blockseq_comparison的一个子集；模拟客户压力测试场景。 | 
int test_solidbits_comparison(bufa, bufb, count); |（√ ）固定全1后写入两个buffer，然后读出比较，然后全0写入读出比较；这就是Zero-One算法，Breuer & Friedman 1976 ，检测SAF的，算法是{w0,r0,w1,r1}时间复杂度是4N，又叫做MSCAN，验证每个cell能读写，间接测试了stuck at fault| 
int test_checkerboard_comparison(bufa, bufb, count); | （√ ）把设定好的几组Data BackGround，依次写入，然后读出比较 （注：论文里说设计良好的Data background可以检测出state coupling faults时间复杂度是4N，这是验证相邻位置是否互相影响从而设计的用例。| 
int test_blockseq_comparison(bufa, bufb, count); | （√ ）一次写一个count大小的块，写的值是拿byte级的数填充32bit，然后取出对比，接着重复256次；也是压力用例，只是次数变多了； | 
int test_walkbits0_comparison(bufa, bufb, count); | （√ ）就是bit=1的位置在32bit里面移动，每移动一次就全部填满buffer，先是从低位往高位移，再是从高位往低位移动，(这么做的目的是啥？其中的一个目的是检测NPSF其次是CFs，其次是数据敏感型异常检测，注这里是32bit的，还有8bit的粒度更细了) | 
int test_walkbits1_comparison(bufa, bufb, count); |（√ ）与上同理，另注：早memtester86中这个算法叫做moving inversions algorithm | 
int test_bitspread_comparison(bufa, bufb, count); |（√ ）还是在32bit里面移动，只是这次移动的不是单单的一个0或者1，而是两个1，这两个1之间隔着两个空位，（是临近耦合异常的一种data pattern变体：两个1之间间隔1个位置，然后同步移动）  | 
int test_bitflip_comparison(bufa, bufb, count); | （√ ）也是32bit里面的一个bit=1不断移动生成data pattern然后，每个pattern均执行：｛取反交替写入a、b缓冲区，写完之后检查一遍，然后不断重复以下步骤八次｛用八个DMA从a缓冲区搬数据到b缓冲区，并行搬，模拟短时间内反复读写同一位置看是否有数据丢失异常｝｝核心思想：短时间内反复读写同一位置。 | 
int test_8bit_wide_random(bufa, bufb, count）; | （√ ）以char指针存值，也就是每次存8bit，粒度更细； | 
int test_16bit_wide_random(bufa, bufb, count);|（√ ）以unsigned short指针存值，也就是每次存16bit，不同粒度检测；  |
×|int test_crosstalk_comparison(bufa, bufb, count)：[32个0,接着32bit里面1个0移动]以这样的模型叠加写入内存；（只有上行，没像有moving inversions algorithm一样进行反转）|


## 详解函数
### memtester-4.3.0 版本
#### 方法test_stuck_address
函数名：`int test_stuck_address(ulv *bufa, size_t count) `
基本pattern按照下图所示，j=0时，先把P1的**地址值**写入**对应的内存位置**处，然后P2取反放入对应位置处，如此反复；

然后下一轮开始，即j=1，把上述步骤反过来再进行一遍即可；
![](http://upload-images.jianshu.io/upload_images/4749583-5b15eb50df973bf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

直到16轮结束，假若发生异常就把异常的地址直接返回即可！

#### 目的（原理）
为了验证是否有地址无法访问，验证的是地址线。

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.
```cpp
int test_stuck_address(ulv *bufa, size_t count) {
    ulv *p1 = bufa;
    unsigned int j;
    size_t i;
    off_t physaddr;
    printf("           ");
    fflush(stdout);
    for (j = 0; j < 16; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        p1 = (ulv *) bufa;
        printf("setting %3u", j);
        fflush(stdout);
        for (i = 0; i < count; i++) {
            *p1 = ((j + i) % 2) == 0 ? (ul) p1 : ~((ul) p1);
            *p1++;
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        p1 = (ulv *) bufa;
        for (i = 0; i < count; i++, p1++) {
            if (*p1 != (((j + i) % 2) == 0 ? (ul) p1 : ~((ul) p1))) {
                if (use_phys) {
                    physaddr = physaddrbase + (i * sizeof(ul));
                    fprintf(stderr, 
                            "FAILURE: possible bad address line at physical "
                            "address 0x%08lx.\n", 
                            physaddr);
                } else {
                    fprintf(stderr, 
                            "FAILURE: possible bad address line at offset "
                            "0x%08lx.\n", 
                            (ul) (i * sizeof(ul)));
                }
                printf("Skipping to next test...\n");
                fflush(stdout);
                return -1;
            }
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b  \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```  


### ARM A53移植版本

```cpp
int test_stuck_address(unsigned int *bufa, unsigned int count)
{	
	unsigned int *p1 = bufa;
	unsigned int j;
	int i;
	for(j = 0; j < 2; j++){
		p1 = (unsigned int *)bufa;
		for(i = 0; i < count; i++){
			*p1 = ((j + i) % 2) == 0 ? (unsigned int)p1 : (~(unsigned int)p1);
			p1++;
		}
		p1 = (unsigned int *)bufa;
		for(i = 0; i < count; i++, p1++){
			if (*p1 != (((j + i) % 2) == 0 ? (unsigned int) p1 : ~((unsigned int) p1))){
				#ifdef PRINTK
				printk("[DRAM]test_stuck_address: %x is %x error\n", p1,(unsigned int)*p1);
				#endif
        if(((j + i) % 2) == 0){
        	return (((unsigned int) p1)^(*p1));
        }
        else{
        	return ((~((unsigned int) p1))^(*p1));
        }				
			}
		}
	}
	return 0;
}
```
---

#### 方法test_random_value
函数名：`int test_random_value(ulv *bufa, ulv *bufb, size_t count) `

开了两个Buffer区域，然后同时写入随机值，写完count个之后，用compare_regions(bufa, bufb, count)函数来对比验证。

```cpp
int compare_regions(ulv *bufa, ulv *bufb, size_t count) {
    int r = 0;
    size_t i;
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    off_t physaddr;
    for (i = 0; i < count; i++, p1++, p2++) {
        if (*p1 != *p2) {
            if (use_phys) {
                physaddr = physaddrbase + (i * sizeof(ul));
                fprintf(stderr, 
                        "FAILURE: 0x%08lx != 0x%08lx at physical address "
                        "0x%08lx.\n", 
                        (ul) *p1, (ul) *p2, physaddr);
            } else {
                fprintf(stderr, 
                        "FAILURE: 0x%08lx != 0x%08lx at offset 0x%08lx.\n", 
                        (ul) *p1, (ul) *p2, (ul) (i * sizeof(ul)));
            }
            /* printf("Skipping to next test..."); */
            r = -1;
        }
    }
    return r;
}
```
#### 目的（原理）
目的是测试data bus，以及某种数据pattern是否会导致cell无法读写，类似于软件测试里面的Monkey test；

其中有几个注意点：

+ 这里开源版本是没有底层加速优化的，一个个地往内存地址写数据，每写一个就要fflush操作一些，免得数据在stdout缓冲区内堆积而不会立即写入DRAM，因此我们底层优化的时候要考虑cache的影响；

+ 上述的对比函数就是一个一个值地对比，要是底层不做优化地话时间花销基本上是neon加速后的四倍；


#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_random_value(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    ul j = 0;
    size_t i;
    putchar(' ');
    fflush(stdout);
    for (i = 0; i < count; i++) {
        *p1++ = *p2++ = rand_ul();
        if (!(i % PROGRESSOFTEN)) {
            putchar('\b');
            putchar(progress[++j % PROGRESSLEN]);
            fflush(stdout);
        }
    }
    printf("\b \b");
    fflush(stdout);
    return compare_regions(bufa, bufb, count);
}
```


### ARM A53移植版本

在公版的基础上融合进来与或的操作，但是还是一个一个写，验证了一下全空间写0的时间是neon写的4倍。

arm平台（平台参数见上述描述）跑了24s左右。

```cpp
int test_random_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count)
{
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	int i;
	int q;
	for(i = 0; i < count; i++, p1++, p2++){
		*p1 = *p2 =  rand_ul();
	}
	p1 = bufa;
	p2 = bufb;
	for(i = 0; i < count; i++, p1++, p2++){
		if(i%2==0){
			q=rand_ul();
			*p1|=q;
			*p2|=q;
		}
		else{
			q=rand_ul();
			*p1&=q;
			*p2&=q;
		}
	}
	return compare_regions(bufa, bufb, count);
}

```
读出对比的时间，全1GB空间实测大概花了6s的样子（A53平台单核1.46GHz 32bit），时间大大减少。
```cpp
int compare_regions(unsigned int *bufa, unsigned int *bufb, unsigned int count)
{	
	unsigned int ret = 0,i,ERRO_ADDR[2];

	ret = mctl_neon_cmp(bufa,bufb,count<<2,&ERRO_ADDR[0]);
  
	if(ret)
	{		
    for(i=0;i<1000;i++)
    {
    	if(__ADDR(ERRO_ADDR[0])==__ADDR(ERRO_ADDR[1]))
    	{
    		printk("[DRAM]addr0:%x!=addr1:%x___compare erro bit is :%x---read erro\n",ERRO_ADDR[0],ERRO_ADDR[1],ret);
    		break;
    	}
    }
    if(i==1000)
    {
    	printk("[DRAM]addr0:%x!=addr1:%x___compare erro bit is :%x---write erro\n",ERRO_ADDR[0],ERRO_ADDR[1],ret);
    }mctl_neon_write
	}
	return ret;
}
```
而减少的时间主要是利用了neon底层加速：注释见代码

由于穿进来的参数是(bufa, bufb, count)，因此：
+ **r0 = bufa**
+ **r1 = bufb**
+ **r2 = count**

![](http://upload-images.jianshu.io/upload_images/4749583-97f7eebbaf662343.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

基本思想就是对比，比完之后异或出现非零值就是出异常了，报警！返回异常值。

``` cpp
mctl_neon_cmp
      	PUSH	{r3-r12, lr}
      	ADD  r2,r2,r0  ；把count的值转成最终的地址值了

neon_cmp
	VLDM r0!,{q0-q3} ;一次从bufa加载4*128bit数据到4个neon的Q寄存器
	VLDM r1!,{q4-q7} ;一次从bufb加载4*128bit数据到4个neon的Q寄存器


      	VEOR q8,q0,q4
      	VEOR q9,q1,q5
      	VEOR q10,q2,q6
      	VEOR q11,q3,q7

        VORR q12,q8,q9
		VORR q13,q11,q10
		VORR q14,q12,q13
		VORR d30,d28,d29
		VMOV r4, r5, d30
		ORR  r6,r4,r5
      
		CMP  r6,#0x0   
        ;上面这一段就是上图的一个实现，为什么一次只操作2X4Q个数据，
        ;是因为总共就16个Q，不够用啊！
        ;要是支持VEOR q0, q0,q4的话这里还可以加速的哟！

	    BNE  rw_detect
        ;不为0跳转指令
        ;检测是否出现异常了，出现异常则结果不为0
        

        ;检测是否全部对比完了
      	CMP r0,r2
		BNE neon_cmp

       ;这里应为r0是返回值，因此测试正常则赋0
		MOV r0,#0x0
		POP	{r3-r12, pc}

rw_detect
;这里是假如中间出错了要进行的操作
      	SUB r0,r0,#0x40
      	SUB r1,r1,#0x40
        ;bufa和bufb地址回退至4X16个32bit，及16个Q之前。
        ;（这里是要对16个Q值均进行清算）
      	
        VMOV r4,r5,d16   ;d16解开
      	CMP r4,#0
      	BNE rw_detect_done
     	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d17;d17
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d18;d18
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d19;d19
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d20;d20
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d21;d21
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d22;d22
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      
      	VMOV r4,r5,d23;d23
      	CMP r4,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4
      	CMP r5,#0
      	BNE rw_detect_done
      	ADD r0,r0,#0x4
      	ADD r1,r1,#0x4

rw_detect_done
      	MOV r2,r3
      	STR r0,[r2];save erro addr
      
      	ADD r3,r3,#0x4
      	MOV r2,r3
      	STR r1,[r2];save erro addr
      
      	MOV r0,r6
      	POP	{r3-r12, pc}
```

---

#### 方法test_walkbits1_comparison/test_walkbits0_comparison
函数名：`nt test_walkbits1_comparison(bufa, bufb, count); `

首先设定一个初始值（以walk1为例）为0x00000001，然后左移一位之后写入下一个地址，依此类推。

#### 目的（原理）
This test is intended to uncover **data or address bus problems** both **internal** to the memory device as well as **external**. 

**同时也覆盖测试临近耦合缺陷。**

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_walkbits1_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j;
    size_t i;

    printf("           ");
    fflush(stdout);
    for (j = 0; j < UL_LEN * 2; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        p1 = (ulv *) bufa;
        p2 = (ulv *) bufb;
        printf("setting %3u", j);
        fflush(stdout);
        for (i = 0; i < count; i++) {
            if (j < UL_LEN) { /* Walk it up. */
                *p1++ = *p2++ = UL_ONEBITS ^ (ONE << j);
            } else { /* Walk it back down. */
                *p1++ = *p2++ = UL_ONEBITS ^ (ONE << (UL_LEN * 2 - j - 1));
            }
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        if (compare_regions(bufa, bufb, count)) {
            return -1;
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b  \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本

```cpp
int test_walkbits1_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count)
{
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	int ret = 0;

	for(j = 0; j <UL_LEN * 2; j++){
		p1 = bufa;
		p2 = bufb;
		if(j < UL_LEN) {
			neon_write(p1,p1+(count<<0),(ONE << j));
			neon_write(p2,p2+(count<<0),(ONE << j));
		}
		else{
			neon_write(p1,p1+(count<<0),(ONE << (UL_LEN * 2 - j - 1)));
			neon_write(p2,p2+(count<<0),(ONE << (UL_LEN * 2 - j - 1)));			
		}
		ret = compare_regions(bufa, bufb, count);
		if(ret){
			return ret;
		}
	}
	return 0;
```

---

#### 方法test_seqinc_comparison/test_blockseq_comparison
函数名：`int test_seqinc_comparison(ulv *bufa, ulv *bufb, size_t count) ` / ` `


#### 目的（原理）

验证连续按顺序写是否功能正常；

block模式则加上了压力部分；

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_seqinc_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    size_t i;
    ul q = rand_ul();
    for (i = 0; i < count; i++) {
        *p1++ = *p2++ = (i + q);
    }
    return compare_regions(bufa, bufb, count);
}

int test_blockseq_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j;
    size_t i;
    printf("           ");
    fflush(stdout);
    for (j = 0; j < 256; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        p1 = (ulv *) bufa;
        p2 = (ulv *) bufb;
        printf("setting %3u", j);
        fflush(stdout);
        for (i = 0; i < count; i++) {
            *p1++ = *p2++ = (ul) UL_BYTE(j);
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        if (compare_regions(bufa, bufb, count)) {
            return -1;
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本

```cpp
int test_blockseq_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count){
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	int i;
	int ret = 0;
	for(j = 0; j < 256; j++){
		p1 = (unsigned int*)bufa;
		p2 = (unsigned int*)bufb;
		for(i = 0; i < count; i++, p1++, p2++){
			*p1 = *p2 = (unsigned int)UL_BYTE(j);
		}
		ret = compare_regions(bufa, bufb, count);
		if(ret){
			return ret;
		}
	}
	return 0;
}
```

---

#### 方法 test_solidbits_comparison
函数名：`int test_solidbits_comparison(ulv *bufa, ulv *bufb, size_t count) `

取一个pattern（全1）然后取反交替写入buffer区间，然后再检测时候有问题，反复64次；

移植版本里面假如了**hammer操作**，也即短时间内不断快速读写同一个位置，看功能是否正常。
#### 目的（原理）

hammer异常检测；
全空间scan测试；

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_solidbits_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j;
    ul q;
    size_t i;
    printf("           ");
    fflush(stdout);
    for (j = 0; j < 64; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        q = (j % 2) == 0 ? UL_ONEBITS : 0;
        printf("setting %3u", j);
        fflush(stdout);
        p1 = (ulv *) bufa;
        p2 = (ulv *) bufb;
        for (i = 0; i < count; i++) {
            *p1++ = *p2++ = (i % 2) == 0 ? q : ~q;
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        if (compare_regions(bufa, bufb, count)) {
            return -1;
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本
增加了并行读写操作（DMA部分）

**DMA_TRAN(1,(s32)bufa,(s32)bufb,0,count << 2)**参数解释：
+ “1”：是从sdram到sdram
+ src_addr = bufa
+ dst_addr = bufb
+ 0~7八个DMA来并行搬运数据，搬运四分之一，那这里模拟的就是对同一个位置反复不断地读写8次，最后再检查一下有没有出错；
```cpp
int test_solidbits_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count)
{
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	unsigned int q;
	int ret = 0,done;
	for(j = 0; j <64; j++){
		q = (j% 2) == 0 ? UL_ONEBITS:~UL_ONEBITS;
		p1 = (unsigned int *)bufa;
		p2 = (unsigned int *)bufb;
		mctl_neon_write(p1,p1+(count<<0),q);
		mctl_neon_write(p2,p2+(count<<0),q);
		ret = compare_regions(bufa, bufb, count);
		if(ret==0){
			DMA_TRAN(1,(s32)bufa,(s32)bufb,0,count << 2);
            DMA_TRAN(1,(s32)bufa,(s32)bufb,1,count << 2);
            DMA_TRAN(1,(s32)bufa,(s32)bufb,2,count << 2);
			DMA_TRAN(1,(s32)bufa,(s32)bufb,3,count << 2);
            DMA_TRAN(1,(s32)bufa,(s32)bufb,4,count << 2);
            DMA_TRAN(1,(s32)bufa,(s32)bufb,5,count << 2);
			DMA_TRAN(1,(s32)bufa,(s32)bufb,6,count << 2);
            DMA_TRAN(1,(s32)bufa,(s32)bufb,7,count << 2);
			ret = (ret|compare_regions0(bufa, bufb, count));
			done = (0xffff);
			do{
			   done = (get_wvalue(0x03002000+0x30) & 0xfff);
			}while(done != 0x0);		//wait for dma transfor finish
		}
		if(ret)
			return ret;		
	}
	return ret;
}
```

---

#### 方法
函数名：` int test_bitflip_comparison(ulv *bufa, ulv *bufb, size_t count)`

也是32bit里面的一个bit=1不断移动生成data pattern然后，每个pattern均执行：｛
  取反交替写入a、b缓冲区，写完之后检查一遍，然后不断重复以下步骤八次｛
用八个DMA从a缓冲区搬数据到b缓冲区，并行搬，模拟短时间内反复读写同一位置看是否有数据丢失异常｝｝

核心思想：短时间内反复读写同一位置。
#### 目的（原理）
对比上面的 test_solidbits_comparison可以发现不同之处就在于data pattern的设计，上面是固定的两个值，这里是walking bit 1，因此这个用例是 test_solidbits_comparison和test_walkbits1_comparison的组合技能。

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_bitflip_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j, k;
    ul q;
    size_t i;

    printf("           ");
    fflush(stdout);
    for (k = 0; k < UL_LEN; k++) {
        q = ONE << k;
        for (j = 0; j < 8; j++) {
            printf("\b\b\b\b\b\b\b\b\b\b\b");
            q = ~q;
            printf("setting %3u", k * 8 + j);
            fflush(stdout);
            p1 = (ulv *) bufa;
            p2 = (ulv *) bufb;
            for (i = 0; i < count; i++) {
                *p1++ = *p2++ = (i % 2) == 0 ? q : ~q;
            }
            printf("\b\b\b\b\b\b\b\b\b\b\b");
            printf("testing %3u", k * 8 + j);
            fflush(stdout);
            if (compare_regions(bufa, bufb, count)) {
                return -1;
            }
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b           \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本

```cpp
int test_bitflip_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count){
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	int k,q;
	int ret = 0,done;
	for (k = 0; k <UL_LEN; k++){
       q = ONE << k;
       for (j = 0; j < 8; j++){
		  q = ~q;
          p1 = bufa;
          p2 = bufb;
          mctl_neon_write(p1,p1+(count<<0),q);
          mctl_neon_write(p2,p2+(count<<0),q);
          ret = compare_regions(bufa, bufb, count);
  		  if(ret==0) {
  		  	//1：是从sdram到sdram
  		  	//src_addr = bufa
  		  	//dst_addr = bufb
  		  	//0~7八个DMA来并行搬运数据，搬运四分之一，那这里模拟的就是对同一个位置反复不断地读写8次，最后再检查一下有没有出错；
  			  DMA_TRAN(1,(s32)bufa,(s32)bufb,0,count << 2);
              DMA_TRAN(1,(s32)bufa,(s32)bufb,1,count << 2);
              DMA_TRAN(1,(s32)bufa,(s32)bufb,2,count << 2);
  			  DMA_TRAN(1,(s32)bufa,(s32)bufb,3,count << 2);
              DMA_TRAN(1,(s32)bufa,(s32)bufb,4,count << 2);
              DMA_TRAN(1,(s32)bufa,(s32)bufb,5,count << 2);
  			  DMA_TRAN(1,(s32)bufa,(s32)bufb,6,count << 2);
              DMA_TRAN(1,(s32)bufa,(s32)bufb,7,count << 2);
  			  ret = (ret|compare_regions0(bufa, bufb, count));
  			  done = (0xff);
  			  do{
  			     done = (get_wvalue(0x03002000+0x30) & 0xff);
  			  }while(done != 0x0);		//wait for dma transfor finish
  		  }
          if(ret) {
			 return ret;
		  }
     }
  }
	return 0;
}
```

---

#### 方法 test_checkerboard_comparison
函数名：`int test_checkerboard_comparison(ulv *bufa, ulv *bufb, size_t count) `
测64轮，每轮都是从两个表格中取出一个data pattern来写入内存，最后读出对比即可；

#### 目的（原理）
数据敏感型功能缺陷检测，也就是说有可能memory就是无法存取某个值的情况，这也是从用户视角出发的。

注：这里并没有给出checkboard的值。

>**注2： 虽然这里有点类似于底层的NPSF（neighborhood pattern sensitive fault），但是这里的锚点却不是这个，而是:比如说我的客户在把内存插入电脑后，使用过程中有这种pattern的数据写入内存，会不会存在数据互相影响从而丢失的问题呢？这搞不好就是蓝屏了啊！也就是说我们关注的是表层的状态而不是底层的缺陷。**

#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_checkerboard_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j;
    ul q;
    size_t i;

    printf("           ");
    fflush(stdout);
    for (j = 0; j < 64; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        q = (j % 2) == 0 ? CHECKERBOARD1 : CHECKERBOARD2;
        printf("setting %3u", j);
        fflush(stdout);
        p1 = (ulv *) bufa;
        p2 = (ulv *) bufb;
        for (i = 0; i < count; i++) {
            *p1++ = *p2++ = (i % 2) == 0 ? q : ~q;
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        if (compare_regions(bufa, bufb, count)) {
            return -1;
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b           \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本
这里的checkboard选择好随意啊！！！
checkboard一般是选择0x55跟0xaa交替写， 检测stuck bit cases and many adjacent cell dependency cases.

标准算法描述如下：

![](http://upload-images.jianshu.io/upload_images/4749583-1dff2a79a5bfe966.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后这个地方的移植······

```cpp
int test_checkrboard_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count){
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	unsigned int q;
	int ret = 0;
	unsigned int CHECKERBOARD1[16]={0x00000000,0x11111111,0x22222222,0x33333333,
                                    0x44444444,0x55555555,0x66666666,0x77777777,
	                                0x88888888,0x99999999,0xaaaaaaaa,0xbbbbbbbb,
                                    0xcccccccc,0xdddddddd,0xeeeeeeee,0xffffffff};
	unsigned int CHECKERBOARD2[16]={0xffffffff,0xeeeeeeee,0xdddddddd,0xcccccccc,
                                    0xbbbbbbbb,0xaaaaaaaa,0x99999999,0x88888888,
	                                0x77777777,0x66666666,0x55555555,0x44444444,
                                    0x33333333,0x22222222,0x11111111,0x00000000};
	for(j = 0; j < 64; j++){
		q = (j % 2) == 0 ? CHECKERBOARD1[(j/2)%16] : CHECKERBOARD2[(j/2)%16];
		p1 = (unsigned int *)bufa;
		p2 = (unsigned int *)bufb;
		mctl_neon_write(p1,p1+(count<<0),q);
		mctl_neon_write(p2,p2+(count<<0),q);
		ret = compare_regions(bufa, bufb, count);
		if(ret){
			return ret;
		}
	}
	return 0;
}
```


---

#### 方法
函数名：`int test_bitspread_comparison(ulv *bufa, ulv *bufb, size_t count) `

跟walkbits1比起就是data pattern变了一点点而已，其余不变，因此可以看作是walkbit1的一个扩展测试；
walkbit1： 00000001 -> 00000010
bitspread:  00000101 -> 00001010
#### 目的（原理）

也是主要为了检测临近耦合缺陷；
![这是内部结构图，有助于理解它内部的寻址](http://upload-images.jianshu.io/upload_images/4749583-ad01d1c8e0782291.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 时间花销
**条件：**
全空间1G Byte ，DDR带宽1600M\*32bit，CPU: ARM A53 （1460~1800）M\* 32bit单核跑。

**时间成本：**
___Sec.

```cpp
int test_bitspread_comparison(ulv *bufa, ulv *bufb, size_t count) {
    ulv *p1 = bufa;
    ulv *p2 = bufb;
    unsigned int j;
    size_t i;

    printf("           ");
    fflush(stdout);
    for (j = 0; j < UL_LEN * 2; j++) {
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        p1 = (ulv *) bufa;
        p2 = (ulv *) bufb;
        printf("setting %3u", j);
        fflush(stdout);
        for (i = 0; i < count; i++) {
            if (j < UL_LEN) { /* Walk it up. */
                *p1++ = *p2++ = (i % 2 == 0)
                    ? (ONE << j) | (ONE << (j + 2))
                    : UL_ONEBITS ^ ((ONE << j) | (ONE << (j + 2)));
            } else { /* Walk it back down. */
                *p1++ = *p2++ = (i % 2 == 0)
                    ? (ONE << (UL_LEN * 2 - 1 - j)) | (ONE << (UL_LEN * 2 + 1 - j))
                    : UL_ONEBITS ^ (ONE << (UL_LEN * 2 - 1 - j)
                                    | (ONE << (UL_LEN * 2 + 1 - j)));
            }
        }
        printf("\b\b\b\b\b\b\b\b\b\b\b");
        printf("testing %3u", j);
        fflush(stdout);
        if (compare_regions(bufa, bufb, count)) {
            return -1;
        }
    }
    printf("\b\b\b\b\b\b\b\b\b\b\b           \b\b\b\b\b\b\b\b\b\b\b");
    fflush(stdout);
    return 0;
}
```


### ARM A53移植版本

```cpp
int test_bitspread_comparison(unsigned int *bufa, unsigned int *bufb, unsigned int count){
#define Q  2
	unsigned int *p1 = bufa;
	unsigned int *p2 = bufb;
	unsigned int j;
	int ret = 0;
for(j = 0; j <UL_LEN * 2; j++){
		p1 = bufa;
		p2 = bufb;
		if(j < UL_LEN) {
			mctl_neon_write(p1,p1+(count<<0),((ONE << j) | (ONE << (j + Q))));
			mctl_neon_write(p2,p2+(count<<0),((ONE << j) | (ONE << (j + Q))));
		}
		else{
			mctl_neon_write(p1,p1+(count<<0),((ONE << (UL_LEN * 2 - 1 - j)) | (ONE << (UL_LEN * 2 +Q- 1 - j))));
			mctl_neon_write(p2,p2+(count<<0),((ONE << (UL_LEN * 2 - 1 - j)) | (ONE << (UL_LEN * 2 +Q- 1 - j))));			
		}
	 	ret = compare_regions(bufa, bufb, count);
		if(ret){
			return ret;
		}
}
	return 0;
}
```

## 总结

整个memtester测试的视角就是从用户的角度来看的，从用户角度设立不同的测试场景即测试用例，然后针对性地进行**功能**测试，注意是从系统级来测试，也就是说关注的不单单是内存颗粒了，还有系统板级的连线、IO性能、PCB等等相关的因素，在这些因素的影响下，你的memory是否还能**正常工作**；

>注2： checkboard这里虽然有点类似于底层的NPSF（neighborhood pattern sensitive fault），但是这里的锚点却不是这个，而是:比如说我的客户在把内存插入电脑后，使用过程中有这种pattern的数据写入内存，会不会存在数据互相影响从而丢失的问题呢？这搞不好就是蓝屏了啊！也就是说我们关注的是表层的状态而不是底层的缺陷。
