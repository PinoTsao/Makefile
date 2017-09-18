# Neccesary make knowledge

本系列文章有以下约定：
1. 下文所有提到 “GNU make“ 文档的时候，都表示 `info make` 的内容，二者表示同一个意思
2. 所有术语均使用英文，因为很多中文翻译并不到位（比如将 prerequisite 翻译为“依赖”，并不十分准确）

Makefile 是一种语言, 有关于 GNU make 的知识都来自于它的文档。本文的读者不必知道所有的细节，只需知道基础概念，即可阅读 Linux kernel 的 Makefile。其他的语法细节可以在需要的时候查阅文档来获得。

关于 Makefile 的中文文档，《跟我一起写makefile》应该是最有名且是最为详细 & 系统的一个，可以通过阅读它来入门。但它的内容，几乎完全来自 GNU make 文档(因为翻译较早，此文档内容可能较为老旧)。 GNU make 的英文文档，个人感受是比较容易阅读的，所以中文文档可以为那些英语能力不够的同学服务，有英语阅读能力的人，强烈建议阅读 `info make`, 80%以上的内容还是很容易读懂的。

对于完全不了解 makefile 的读者，可以先读《跟我一起写makefile》获得直观感受后，再阅读GNU make 的英文文档，来获得最准确权威的 makefile 知识

**在已经了解 makefile 基本概念后，能力允许的情况下，尽可能多的阅读官方文档是学习它的最佳方式。**

关于 `info make` 的内容, 前三章应该通读，即：

>* Overview::
>* Introduction::
>* Makefiles::

接下来的四章

>* Rules::
>* Recipes::
>* Using Variables::
>* Conditionals::

应该通读其前若干小节。其他内容大部分可以按需查找。

本系列文章的目标是分析编译 kernel 时常见操作的详细流程，为了这一目的，最需要掌握的 makefile 知识是 Phony target（伪目标），参考 `info make` 的 "4.6 Phony Targets"

>A phony target is one that is not really the name of a file; rather it
>is just a name for a recipe to be executed when you make an explicit
>request.  There are two reasons to use a phony target: to avoid a
>conflict with a file of the same name, and to improve performance.

典型的例子是：

        .PHONY: clean
        clean:
                rm *.o temp

想执行 rm 命令，则需显式的通过 `make clean` 来完成。

**Phony target 可以作为其他 target 的 prerequisites, 不管是 phony 的还是真实文件的 target。当它作为其他 target 的 prerequisites 的时候，它的实际作用是这个 target 的一个 subroutine**