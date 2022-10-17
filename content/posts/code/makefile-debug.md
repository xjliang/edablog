---
title: "如何调试 Makefile"
tags: ["debug"]
date: 2022-10-17T21:54:57+08:00
draft: false
---

## 简介

对于Makefile中的各种变量，可能是我们比较头痛的事了。我们要查看他们并不是很方便，需要修改makefile加入echo命令。这有时候很不方便。其实我们可以制作下面一个专门用来输出变量的makefile（假设名字叫：vars.mk）

```makefile
%:
	@echo '$*=$($*)'

d-%:
	@echo '$*=$($*)'
	@echo '  origin = $(origin $*)'
	@echo '   value = $(value  $*)'
	@echo '  flavor = $(flavor $*)'
```

这样一来，我们可以使用make命令的-f参数来查看makefile中的相关变量（包括make的内建变量，比如：COMPILE.c或MAKE_VERSION之类的）。**注意：第二个以“d-”为前缀的目标可以用来打印关于这个变量更为详细的东西**

```makefile
make -f test.mk -f vars.mk OBJS
```

我们可以看到：

make 的第一个 -f 后是要测试的makefile，第二个是我们的 debug makefile。

后面直接跟变量名，如果在变量名前加”d-”，则输出更为详细的东西。
说一说”d-” 前缀（其意为details），其中调用了下面三个参数。

(origin)：告诉你这个变量是来自哪儿，file表示文件，environment表示环境变量，还有environment override，command line，override，automatic等。
​(value)：打出这个变量没有被展开的样子。比如上述示例中的 foo 变量。
$(flavor)：有两个值，simple表示是一般展开的变量，recursive 表示递归展开的变量