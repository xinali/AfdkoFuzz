# CrashesAnalysis_1

这个是一周前`clone`的最新库`fuzz`结果

## crash结果


首先来看一下批处理`crash`发现的`gdb`结果

```
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x00000000004965b9 in parseSVG (h=0xa228030) at ../../../../../source/svread/svread.c:915
915	            if (tk->length == 1) {
#0  0x00000000004965b9 in parseSVG (h=0xa228030) at ../../../../../source/svread/svread.c:915
#1  0x0000000000495d79 in svrBegFont (h=0xa228030, flags=0, top=0xa20e078) at ../../../../../source/svread/svread.c:1298
#2  0x00000000004e1126 in svrReadFont ()
#3  0x000000000042b49b in doFile (h=0xa20e040, srcname=0x7fffffffe48c "./crashes/SIGSEGV.PC.7f8e371974d9.STACK.badbad0d4cce199d.CODE.128.ADDR.(nil).INSTR.[NOT_MMAPED].2019-07-24.17:14:19.36397.fuzz") at ../../../../source/tx.c:435
#4  0x000000000042acd5 in doSingleFileSet (h=0xa20e040, srcname=0x7fffffffe48c "./crashes/SIGSEGV.PC.7f8e371974d9.STACK.badbad0d4cce199d.CODE.128.ADDR.(nil).INSTR.[NOT_MMAPED].2019-07-24.17:14:19.36397.fuzz") at ../../../../source/tx.c:488
#5  0x0000000000429f15 in main (argc=1, argv=0x7fffffffe190) at ../../../../source/tx.c:1576
Dump of assembler code from 0x4965b9 to 0x4965c9:
=> 0x00000000004965b9 <parseSVG+1881>:	mov    0x408(%rax),%rax
   0x00000000004965c0 <parseSVG+1888>:	mov    %rax,%rsi
   0x00000000004965c3 <parseSVG+1891>:	mov    %rax,-0x188(%rbp)
End of assembler dump.
rax            0x0	0
rbx            0x0	0
rcx            0x1	1
rdx            0x1355	4949
rsi            0x7a9098	8032408
rdi            0x1	1
rbp            0x7fffffffdca0	0x7fffffffdca0
rsp            0x7fffffffda30	0x7fffffffda30
r8             0x0	0
r9             0x1207c60	18906208
r10            0x0	0
r11            0x9002000	151003136
r12            0x404c40	4213824
r13            0x7fffffffe180	140737488347520
r14            0x0	0
r15            0x0	0
rip            0x4965b9	0x4965b9 <parseSVG+1881>
eflags         0x10202	[ IF RF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
A debugging session is active.

	Inferior 1 [process 115528] will be killed.

Quit anyway? (y or n) [answered Y; input not from terminal]
[+] crash file ===> ./crashes/SIGSEGV.PC.7f8e371974d9.STACK.badbad0d4cce199d.CODE.128.ADDR.(nil).INSTR.[NOT_MMAPED].2019-07-24.17:14:19.36397.fuzz
```


## 分析

编译后直接利用`poc.otf`运行

![1564391281743](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_1/1564391281743.png)

发现确实是出问题了，现在来看看为什么会出问题

```
pwndbg> p/x tk->length
Cannot access memory at address 0x408
pwndbg> p/x tk
$3 = 0x0
```

可以发现`tk==NULL`，但是却要访问其数据，所以出错了。

那来看看`tk`是哪里来的

根据出错附近的代码

```
....
char *endPtr;
tk = getAttribute(h);
if (tk->length == 1) {
...
```

可以发现`tk`是`getAttribute`的返回值

继续回溯

```c
static token *getAttribute(svrCtx h) {
    char ch;
    h->mark = NULL;

    while (bufferReady(h)) {
        ch = *h->src.next;
        if (ch == 0) {
            break;
        }
        if (ch == '"')
            h->src.next++;
        else
            break;
    }

    while (bufferReady(h)) {
        if (ch == 0) {
            break;
        } else if ((ch == '%') || (ch == '#')) {
            ch = *h->src.next++;
            if (!bufferReady(h))
                break;
            while (!(ch == '\n' || ch == '\r' || ch == '\f')) {
                ch = *h->src.next++;
                if ((ch == 0) || (!bufferReady(h)))
                    break;
            }
        } else if (h->mark == NULL) {
            h->mark = h->src.next++;
            if ((ch == 0) || (!bufferReady(h)))
                break;
            ch = *h->src.next;
            while (!(ch == '"')) {
                h->src.next++;
                if ((ch == 0) || (!bufferReady(h)))
                    break;
                ch = *h->src.next;
            }
            break;
        } else {
            break;
        }
    }
    return setToken(h);
}
```

`getAttribute`最后返回了`setToken`

继续

```c
static token *setToken(svrCtx h) {
    size_t len;
    if (h->src.buf == NULL || h->mark == NULL)
        return NULL;

    len = h->src.next - h->mark;
    if ((len + 1) > kMaxToken)
        return NULL;

    memcpy(h->src.tk.val, h->mark, len);
    h->src.tk.val[len] = 0;
    h->src.tk.length = len;
    h->src.tk.offset = h->src.offset + (h->mark - h->src.buf);
    h->src.tk.type = svrUnknown;

    return &h->src.tk;
}
```

可以发现`setToken`会返回两种结果

```
NULL
&h->src.tk;
```

所以当`tk==NULL`，而`parseSVG`对`tk`过滤不严时，就会出问题

回到`parseSVG`的代码可以发现，在`tk`被赋值后，并没有对`tk`做任何的检查

```
tk = getAttribute(h);
if (tk->length == 1)
```

所以出问题了

具体利用`gdb`跟一下，看看逻辑是否有问题

设下断点

![1564393108526](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_1/1564393108526.png)

之后设置一个条件断点

```
b svread.c:915 if tk==0
```

继续，经过多次循环，会发现返回值为`NULL`

![1564400574950](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_1/1564400574950.png)

可以发现成功断了下了，确实是这样。

## 修复

`24`号出的结果，我`29`来分析，谁知道四天前这个漏洞被修复了

![1564401969904](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_1/1564401969904.png)

很蛋疼

来看一下修复

![1564402021089](https://raw.githubusercontent.com/xinali/img/master/blog/fuzzing/CrashesAnalysis_1/1564402021089.png)

在开始取值前做了判断