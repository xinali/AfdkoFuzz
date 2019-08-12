# CrashesAnalysis_3

这个直接使用我提交的`issue`


# heap-based out-of-bounds read when parsing undefined FontName with svg font file

Please excuse my poor English. I'm not a native speaker. I will do my best to describe this bug.

First I found this error with `honggfuzz`.

In lates commit `2e8fc3d6b218cb79a0b159ba663a2ad7622fb73c` 

> use clang compile with debug option

compile `tx` in `c/tx/build/linux/gcc/debug/`

```bash
make clean && CC=clang make
```

then use `tx `to parse a specific svg font file

```
tx -svg poc.svg
```

then `tx` segment fault

```
./tx -svg poc.svg 
./tx[1]    28077 segmentation fault (core dumped)  ./tx -svg poc.svg
```

I use `pwndbg` to debug

```
 ? 0x7ffff778f746 <strlen+38>    movdqu xmm4, xmmword ptr [rax]
   0x7ffff778f74a <strlen+42>    pcmpeqb xmm4, xmm0
   0x7ffff778f74e <strlen+46>    pmovmskb edx, xmm4
   0x7ffff778f752 <strlen+50>    test   edx, edx
   0x7ffff778f754 <strlen+52>    je     strlen+58 <0x7ffff778f75a>
    ↓
   0x7ffff778f75a <strlen+58>    and    rax, 0xfffffffffffffff0
   0x7ffff778f75e <strlen+62>    pcmpeqb xmm1, xmmword ptr [rax + 0x10]
   0x7ffff778f763 <strlen+67>    pcmpeqb xmm2, xmmword ptr [rax + 0x20]
   0x7ffff778f768 <strlen+72>    pcmpeqb xmm3, xmmword ptr [rax + 0x30]
   0x7ffff778f76d <strlen+77>    pmovmskb edx, xmm1
   0x7ffff778f771 <strlen+81>    pmovmskb r8d, xmm2
───────────────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────
00:0000│ rsp  0x7fffffffdc58 ?? 0x441a89 (writeXMLStr+25) ?? mov    qword ptr [rbp - 0x18], rax
01:0008│      0x7fffffffdc60 ?? 0x18
02:0010│      0x7fffffffdc68 ?? 0x6f37f0 ?? 0x1
03:0018│      0x7fffffffdc70 ?? 0x7fffffffdca0 ?? 0x7fffffffdef0 ?? 0x7fffffffdf10 ?? 0x7fffffffdf70 ?? ...
04:0020│      0x7fffffffdc78 ?? 0x441a04 (writeStr+52) ?? add    rsp, 0x20
05:0028│      0x7fffffffdc80 ?? 0x0
06:0030│      0x7fffffffdc88 ?? 0x6f37f0 ?? 0x1
07:0038│      0x7fffffffdc90 ?? 0x0
─────────────────────────────────────────────────[ BACKTRACE ]──────────────────────────────────────────────────
 ? f 0     7ffff778f746 strlen+38
   f 1           441a89 writeXMLStr+25
   f 2           441428 svwEndFont+984
   f 3           478914 svg_EndFont+36
   f 4           46ec3b svrReadFont+443
   f 5           4058fb doFile+939
   f 6           404f3e doSingleFileSet+46
   f 7           402d89 parseArgs+425
   f 8           401c27 main+455
   f 9     7ffff7724830 __libc_start_main+240
────────────────────────────────────────────────────────────────────────────────────────────────────────────────
Program received signal SIGSEGV (fault address 0x0)
pwndbg> bt
#0  strlen () at ../sysdeps/x86_64/strlen.S:106
#1  0x0000000000441a89 in writeXMLStr (h=0x6f37f0, s=0x0) at ../../../../../source/svgwrite/svgwrite.c:215
#2  0x0000000000441428 in svwEndFont (h=0x6f37f0, top=0x6f8e00) at ../../../../../source/svgwrite/svgwrite.c:450
#3  0x0000000000478914 in svg_EndFont ()
#4  0x000000000046ec3b in svrReadFont ()
#5  0x00000000004058fb in doFile (h=0x6ec010, srcname=0x7fffffffe67f "poc.svg") at ../../../../source/tx.c:435
#6  0x0000000000404f3e in doSingleFileSet (h=0x6ec010, srcname=0x7fffffffe67f "poc.svg") at ../../../../source/tx.c:488
#7  0x0000000000402d89 in parseArgs (h=0x6ec010, argc=2, argv=0x7fffffffe3b0) at ../../../../source/tx.c:558
#8  0x0000000000401c27 in main (argc=2, argv=0x7fffffffe3b0) at ../../../../source/tx.c:1587
#9  0x00007ffff7724830 in __libc_start_main (main=0x401a60 <main>, argc=3, argv=0x7fffffffe3a8, init=<optimized out>, fini=<optimized out>, rtld_fini=<optimized out>, stack_end=0x7fffffffe398) at ../csu/libc-start.c:291
#10 0x0000000000401989 in _start ()
```

then I do some analysis, I found when `tx` parse the specific svg font file with no `FontName`, `tx` will occur segment fault.

To show the bug details, set breakpoint at `svgwrite.c:383` in function `svwEndFont`

```c
/* Finish reading font. */
int svwEndFont(svwCtx h, abfTopDict *top) {
    size_t cntTmp = 0;
    size_t cntRead = 0;
    size_t cntWrite = 0;
    char *pBuf = NULL;

    /* Check for errors when accumulating glyphs */
    if (h->err.code != 0)
        return h->err.code;

    h->top = top; // no check!

    /* Set error handler */
    DURING_EX(h->err.env)
        ...
```

then check some variable

```
pwndbg> p top
$1 = (abfTopDict *) 0x6f8e00
pwndbg> p *top
$2 = {
  version = {
    ptr = 0x0, 
    impl = -1
  }, 
  Notice = {
    ptr = 0x0, 
    impl = -1
  }, 
  Copyright = {
    ptr = 0x0, 
    impl = -1
  }, 
  FullName = {
    ptr = 0x0, 
    impl = -1
  }, 
  FamilyName = {
    ptr = 0x0, 
    impl = -1
  }, 
  Weight = {
    ptr = 0x0, 
    impl = -1
  }, 
  isFixedPitch = 0, 
  ItalicAngle = 0, 
  UnderlinePosition = -100, 
  UnderlineThickness = 50, 
  UniqueID = -1, 
  FontBBox = {0, 0, 0, 0}, 
  StrokeWidth = 0, 
  XUID = {
    cnt = 0, 
    array = {0 <repeats 16 times>}
  }, 
  PostScript = {
    ptr = 0x0, 
    impl = -1
  }, 
  BaseFontName = {
    ptr = 0x0, 
    impl = -1
  }, 
  BaseFontBlend = {
    cnt = 0, 
    array = {0 <repeats 15 times>}
  }, 
  FSType = -1, 
  OrigFontType = -1, 
  WasEmbedded = 0, 
  SynBaseFontName = {
    ptr = 0x0, 
    impl = -1
  }, 
  cid = {
    FontMatrix = {
      cnt = 0, 
      array = {0, 0, 0, 0, 0, 0}
    }, 
    CIDFontName = {
      ptr = 0x0, 
      impl = -1
    }, 
    Registry = {
      ptr = 0x0, 
      impl = -1
    }, 
    Ordering = {
      ptr = 0x0, 
      impl = -1
    }, 
    Supplement = -1, 
    CIDFontVersion = 0, 
    CIDFontRevision = 0, 
    CIDCount = 8720, 
    UIDBase = -1
  }, 
  FDArray = {
    cnt = 1, 
    array = 0x6f90a8
  }, 
  sup = {
    flags = 0, 
    srcFontType = -1, 
    filename = 0x0, 
    UnitsPerEm = 1000, 
    nGlyphs = 0
  }, 
  maxstack = 0, 
  varStore = 0x0
}
```

you can see, svg file has no `FontName`

```
BaseFontName = {
    ptr = 0x0,  // null value
    impl = -1
 }
```

then check `top->FDArray.array`

```
pwndbg> p *(top->FDArray.array) 
$5 = {
  FontName = {
    ptr = 0x0,  // null value
    impl = -1
  }, 

```

back into souce code,  in `svgwrite.c:393`

```
h->top = top;  // no check!
```

in  `svgwrite.c:450`

```
 writeXMLStr(h, h->top->FDArray.array[0].FontName.ptr);
```

there is no code to check whether `h->top->FDArray.array[0].FontName.ptr` is available or not.

in `writeXMLStr`

```c
static void writeXMLStr(svwCtx h, const char *s) {
    /* 64-bit warning fixed by cast here */
    long len = (long)strlen(s);  // segment fault!
    int i;
    char buf[9];
    unsigned char code;
....
```

it visit `s`, then segment fault.

The bug exsits, because `tx` doesn't check `FontName` pointer.

All data is in heap. If someone use a specific svg font file,  it may cause some security issues with `OOB`

If necessary, I can send you a proof of concept for this bug.