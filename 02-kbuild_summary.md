# 内核编译系统概述

linux kernel 的编译使用了一种叫作 kbuild 的系统来管理.
Makefile 是一种语言
kernel 编译是 per directory 的. 为什么这么说? 由Makefile 可知, 所有的真正编译动作的入口长这个样子:

        $(Q)$(MAKE) $(build)=$@

也就是进入了 Makefile.build 中进行后面的编译动作. 在 Makefile.build 中又依次 include 了:

scripts/Kbuild.include
目标 directory 中的 Makefile
scripts/Makefile.lib
scripts/Makefile.host # 如果有定义 hostprogs-y

"目标 directory 的 Makefile" 中的主要内容是 obj-y, obj-m 的定义,
Makefile.lib 对这些定义进行了进一步的处理,
比如: 筛出 composite object; 挑出 subdir, Descend 进去, recursive 的进行:

        $(Q)$(MAKE) $(build)=$@

每一个目录下的 Makefile 中的 obj-y 定义的是要被编译进该目录下的built-in.o 的所有 .o 的集合

obj-y, obj-m 中的 composite object, 都可以被看作 module. 不同的是, obj-y 中的 composite object 被放入 built-in.o,
所以, module的意义实际上是 function module. 而 obj-m 中的内容本来就是要被编译成 kernel module 的, 即 .ko 文件.