# AfdkoFuzz

最近`PJ0`的`mjurczyk`公开了`afdko`的`20`个漏洞，依次学习分析并`fuzz`一下

整个过程，涉及了每个漏洞的具体分析和一些分析方法，最后尽力给出每个漏洞的`fuzz`代码(这个小目标实现当然是最好了)

## CVE-2019-1117


具体参见目录`CVE-2019-1117`


## CVE-2019-1118


具体参见目录`CVE-2019-1118`


## CVE-2019-1127


具体参见目录`CVE-2019-1127`


以上三个漏洞由一种文件格式形成，即都是在`tx`在处理`otf`文件出错，所以`fuzz`代码一种就可以全部搞定
并且是直接处理`otf`文件，没有其他需要注意的选项


## MSRC-51734

`testing`



## CrashesAnalysis

分析自己`fuzz`的最新结果

## 参考


[PJ0](https://bugs.chromium.org/p/project-zero/issues/list?can=1&q=finder%3Amjurczyk+reported%3A2019-apr-26)