# AfdkoFuzz

最近`PJ0`的`mjurczyk`公开了`afdko`的`20`个漏洞，依次学习分析并`fuzz`一下

整个过程，涉及了每个漏洞的具体分析和一些分析方法，最后尽力给出每个漏洞的`fuzz`代码(这个小目标实现当然是最好了)

## CVE-2019-1117


具体参见目录`CVE-2019-1117`


## CVE-2019-1118


具体参见目录`CVE-2019-1118`


## Continuous fuzzing


One-off fuzzing might find you some bugs,
but unless you make the fuzzing process **continuous**
it will be a wasted effort.

A simple continuous fuzzing system could be written in < 100 lines of bash code.
In an infinite loop do the following:

* Pull the current revision of your code.
* Build the fuzz target
* Copy the current corpus from cloud to local disk
* Fuzz for some time.
  * With libFuzzer, use the flag `-max_total_time=N` to set the time in seconds).
* Synchronize the updated corpus back to the cloud
* Provide the logs, coverage information, crash reports, and crash reproducers
  via e-mail, web interface, or cloud storage.


## 参考


[PJ0](https://bugs.chromium.org/p/project-zero/issues/list?can=1&q=finder%3Amjurczyk+reported%3A2019-apr-26)

[最后引用来源](https://github.com/google/fuzzer-test-suite/edit/master/tutorial/libFuzzerTutorial.md)