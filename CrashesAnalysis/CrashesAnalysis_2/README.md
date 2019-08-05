# CrashesAnalysis_2

这个同样是一周前`clone`的最新库`fuzz`结果

问题出现在`svread.c`中的`parseSVG`执行`strncpy`复制前长度判断不严。

## crash结果

首先来看一下批处理`crash`发现的`gdb`结果

```
Program received signal SIGSEGV, Segmentation fault.
0x0000000000496010 in parseSVG (h=0x998c5a43686190bf) at ../../../../../source/svread/svread.c:864
864	    while (!(h->flags & SEEN_END)) {
#0  0x0000000000496010 in parseSVG (h=0x998c5a43686190bf) at ../../../../../source/svread/svread.c:864
#1  0x0d895f8642801ab0 in ?? ()
#2  0x330ad0e8e8e8e8a7 in ?? ()
#3  0x000000000a24ddef in ?? ()
#4  0x0000000000581468 in ?? ()
#5  0x0000000000000008 in ?? ()
#6  0x000000000a20e078 in ?? ()
#7  0x0000000000000000 in ?? ()
Dump of assembler code from 0x496010 to 0x496020:
=> 0x0000000000496010 <parseSVG+432>:	mov    0xcd28(%rax),%ecx
   0x0000000000496016 <parseSVG+438>:	and    $0x1,%ecx
   0x0000000000496019 <parseSVG+441>:	mov    %ecx,%esi
   0x000000000049601b <parseSVG+443>:	mov    %ecx,-0x148(%rbp)
End of assembler dump.
rax            0x998c5a43686190bf	-7382426443606552385
rbx            0x0	0
rcx            0xffffffdd	4294967261
rdx            0x0	0
rsi            0x0	0
rdi            0x0	0
rbp            0x7fffffffdca0	0x7fffffffdca0
rsp            0x7fffffffda30	0x7fffffffda30
r8             0x0	0
r9             0xfffffffffffffff	1152921504606846975
r10            0x0	0
r11            0x7ffff703c5e0	140737337607648
r12            0x404c40	4213824
r13            0x7fffffffe180	140737488347520
r14            0x0	0
r15            0x0	0
rip            0x496010	0x496010 <parseSVG+432>
eflags         0x10246	[ PF ZF IF RF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
A debugging session is active.

	Inferior 1 [process 120570] will be killed.

Quit anyway? (y or n) [answered Y; input not from terminal]
[+] crash file ===> ./crashes/SIGSEGV.PC.7f8e371974d9.STACK.badbad0d4cce199d.CODE.128.ADDR.(nil).INSTR.[NOT_MMAPED].2019-07-24.18:05:33.75589.fuzz
===========================================================
```

## 原理分析

调试，运行，崩溃

![1564474504910](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564474504910.png)

查看一下相关变量

```c
pwndbg> p/x h->flags
Cannot access memory at address 0x998c5a4368625de7
pwndbg> p/x h
$1 = 0x998c5a43686190bf
```

查看一下内存布局

![1564474610621](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564474610621.png)

所以`h`作为一个指针，这个地址，肯定是错的

下断点，重新调试

看一下没出问题前`h`的值

![1564474710083](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564474710083.png)

指向这个堆

![1564474736491](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564474736491.png)

目前没有任何问题，但从刚刚崩溃中可以看到`h`的值会被修改为一个无效地址

所以监控一下`h`的值，如果发生变化，就断下

```
pwndbg> watch h!=0xa228030
```

继续执行

![1564475170107](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564475170107.png)

在执行`strncpy`时断下来了，看一下调用栈，并定位一下出错代码

![1564475284775](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564475284775.png)

好了，出错位置找到了

在这个位置下一个断点，重新执行

![1564475411090](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564475411090.png)

看看执行这句后会有什么效果

![1564475442784](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564475442784.png)

`h`已经变为了一个无效指针。

来看看为什么，重点在`strncpy`函数

```c
strncpy(tempVal, tk->val + 1, tk->length - 2);
```

其中`tempVal`定义

```
char tempVal[kMaxName];
```

`kMaxName`定义

```c
#define kMaxName 64
```

`tempVal`是一个`64`字节长度的字符数组

再来看看`tk->length - 2`数据

```
pwndbg> p/x tk->length-2
$6 = 0xab
pwndbg> p tk->length-2
$7 = 171
```

所以`tempVal`数组执行完`strncpy`是会溢出的

那溢出一定会覆盖`h`的值吗？答案是不一定，看你使用什么编译器了。

经过多次测试发现，如果使用`gcc`编译，那么就不会被覆盖，如果使用`clang`编译就会被覆盖至于为什么，其实直接看反汇编代码就懂了

`gcc`编译

![1564476053040](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564476053040.png)

`clang`编译

![1564476112415](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564476112415.png)

可以发现`gcc`把`h`这种重要的数据结构放在了距离`rbp`更远的地方，减少了被覆盖的风险。

利用这个漏洞，攻击者通过构建特定结构的`svg`的字体文件，很有可能会造成命令执行

## 总结

很不幸，分析完之后，发现利用`afdko`的最新代码并不能复现！

经过多次测试，发现`afdko`通过`commit 973a8c929b4fb204dd8fcc84cd12ff02c1ea8aab`

![1564476633927](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_2/1564476633927.png)

把这个洞给堵上了，堵上的原理就是，利用的`poc.svg`在执行到`strncpy`前就被断了。

但这个漏洞理论上还是存在的，因为`strncpy`执行前，现有代码还是没有对长度进行判断，但要是想绕过的话，需要进一步`fuzz`出能够触发这个洞的`poc`数据

连续两个洞，都是因为这一个`commit`，导致无法利用，真是心塞啊


为了赚奶粉钱，本文首发于[安全客](https://www.anquanke.com/post/id/183077)