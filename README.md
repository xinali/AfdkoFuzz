# AfdkoFuzz

最近`PJ0`的`mjurczyk`公开了`afdko`的`20`个漏洞，依次学习分析并`fuzz`一下

## CVE-2019-1117


`tx_fuzz.c`



## 正常fuzz构建


```
1. build.sh
2. target.patch || target.cc
3. input_corpus/output_corpus
```


## 参考
[PJ0](https://bugs.chromium.org/p/project-zero/issues/list?can=1&q=finder%3Amjurczyk+reported%3A2019-apr-26)