---

title: 使用 pmu-tools 进行 TopDown 分析
date: 2021-01-24 18:40
author: gatieme
tags: hexo
categories:
        - hexo
thumbnail: 
blogexcerpt: 博文摘要

---

| CSDN | GitHub | Hexo |
|:----:|:------:|:----:|
| [Aderstep--紫夜阑珊-青伶巷草](http://blog.csdn.net/gatieme) | [`AderXCoding/system/tools`](https://github.com/gatieme/AderXCoding/tree/master/system/tools) | [gatieme.github.io](https://gatieme.github.io) |

<br>

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可, 转载请注明出处, 谢谢合作

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 鄙人在此谢谢啦

<br>

# 1 Topdown & PMU-TOOLS 简介
-------

## 1.1 TopDown 简介
-------

## 1.2 TopDown 资料汇总
-------


Top-down Microarchitecture Analysis Method(TMAM)资料
之前介绍过TMAM的具体内容, 在这里对网络上相关的信息和资料做一个汇总：

*       国外资料

[Tuning Applications Using a Top-down Microarchitecture Analysis Method](https://software.intel.com/content/www/us/en/develop/documentation/vtune-cookbook/top/methodologies/top-down-microarchitecture-analysis-method.html)

[Top-down Microarchitecture Analysis through Linux perf and toplev tools](http://www.cs.technion.ac.il/~erangi/TMA_using_Linux_perf__Ahmad_Yasin.pdf)

[A Top-Down method for performance analysis and counters architecture](https://ieeexplore.ieee.org/document/6844459/metrics#metrics)

[Performance_Analysis_in_a_Nutshell](https://doc.itc.rwth-aachen.de/download/attachments/28344675/08.Performance_Analysis_in_a_Nutshell.pdf?version=1&modificationDate=1480665136000&api=v2)

[Top Down Analysis Never lost with Xeon® perf. counters](https://indico.cern.ch/event/280897/contributions/1628888/attachments/515367/711139/Top_Down_for_CERN_2nd_workshop_-_Ahmad_Yasin.pdf)

[How TMA* Addresses Challenges in Modern Servers and Enhancements Coming in IceLake](https://dyninst.github.io/scalable_tools_workshop/petascale2018/assets/slides/TMA%20addressing%20challenges%20in%20Icelake%20-%20Ahmad%20Yasin.pdf)

*       国内资料


[几句话说清楚15：Top-Down性能分析方法资料及Toplev使用](https://decodezp.github.io/2019/02/14/quickwords15-toplev)

当然还有一些其他的相关信息, 不过上面几个都可以覆盖


## 1.3 PMU-TOOLS 简介
-------

`toplev` 是一个基于 `perf` 和 `TMAM` 方法的应用性能分析工具. 从之前的介绍文章中可以了解到 `TMAM` 本质上是对 `CPU Performance Counter` 的整理和加工. 取得 `Performance Counter` 的读数需要 `perf` 来协助, 对读数的计算进而明确是 `Frondend bound` 还是 `Backend bound` 等等. 

在最终计算之前, 你大概需要做三件事: 

明确 CPU 型号, 因为不同的 CPU, 对应的 PMU 也不一样
读取 TMAM 需要的 perf event 读数
按 TMAM 规定的算法计算, 具体算法在这个 Excel 表格里
这三步可以自动化地由程序来做. 本质上 `toplev` 就是在做这件事. 

toplev的Github地址: https://github.com/andikleen/pmu-tools

另外补充一下, TMAM作为一种Top-down方法, 它一定是分级的. 通过上一级的结果下钻, 最终定位性能瓶颈. 那么toplev在执行的时候, 也一定是包含这个“等级”概念的. 

下面是 `toplev` 使用方法的资料：

[toplev manual](https://github.com/andikleen/pmu-tools/wiki/toplev-manual)

[pmu-tools, part II: toplev](http://halobates.de/blog/p/262)

基本上都是由toplev的开发者自己写的, 可以作为一个Quick Start Guide. 


# 2 PMU-TOOLS 安装
-------

## 2.1 下载 github 仓库
-------

```cpp
git clone https://github.com/andikleen/pmu-tools
```

## 2.2 下载 PMU 表
-------


# 3 PMU-TOOLS 使用
-------




<br>

*	本作品/博文 ( [AderStep-紫夜阑珊-青伶巷草 Copyright ©2013-2017](http://blog.csdn.net/gatieme) ), 由 [成坚(gatieme)](http://blog.csdn.net/gatieme) 创作.

*	采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可. 欢迎转载、使用、重新发布, 但务必保留文章署名[成坚gatieme](http://blog.csdn.net/gatieme) ( 包含链接: http://blog.csdn.net/gatieme ), 不得用于商业目的. 

*	基于本文修改后的作品务必以相同的许可发布. 如有任何疑问, 请与我联系.
