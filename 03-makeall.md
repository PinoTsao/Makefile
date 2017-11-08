# What has `make [all]` done

配置结束后，作为想编译内核的小白，自然是直接敲入 make，或者 make all, 二者是一样的效果。
本文写作时的内核版本是 4.14 rc6，因为 kbuild 还在不断的变化，很有可能在细节上有变化。

Makefile 中的第一个 target 是 make 的 default goal。在 top Makefile 中，第一个 target 是：

	# That's our default target when none is given on the command line
	PHONY := _all
	_all:

_all 更的作用更像是个 placeholder，因为真正作用的 target 都是它的 prerequisites。当不加任何参数执行 make 时，这可能是整个编译过程最复杂的一步(这也意味着，本文可能无比的长)，用流程图来观察会更清晰；又因为流程无比的长，所以一张图是不够的，将分成几个 part。 

Part1：
![vmlinux-1](res/vmlinux-1.png  "vmlinux_process_1")

直接 `make` 时的 target 有 3 个: vmlinux, bzImage, modules，一个个来介绍。

## vmlinux

可以看出 $(vmlinux-deps) 是重点，那它包含了什么？它的完整定义分散在 top Makefile 和 arch/x86/Makefile 中：

	vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN) $(KBUILD_VMLINUX_LIBS)
	# Externally visible symbols (used by link-vmlinux.sh)
	export KBUILD_VMLINUX_INIT := $(head-y) $(init-y)
	export KBUILD_VMLINUX_MAIN := $(core-y) $(libs-y2) $(drivers-y) $(net-y) $(virt-y)
	export KBUILD_VMLINUX_LIBS := $(libs-y1)
	export KBUILD_LDS          := arch/$(SRCARCH)/kernel/vmlinux.lds

	# head-y 定义来自于 arch/x86/Makefile
	head-y := arch/x86/kernel/head_$(BITS).o
	head-y += arch/x86/kernel/head$(BITS).o
	head-y += arch/x86/kernel/ebda.o
	head-y += arch/x86/kernel/platform-quirks.o


	# Objects we will link into vmlinux / subdirs we need to visit
	init-y          := init/
	drivers-y       := drivers/ sound/ firmware/
	net-y           := net/
	libs-y          := lib/
	core-y          := usr/
	virt-y          := virt/

	core-y          += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/

	# 下面 3 行定义也来自 arch/x86/Makefile
	libs-y  += arch/x86/lib/
	core-y += arch/x86/
	drivers-<CONFIG> # 一堆定义

	# 这一步处理把所有变量变成对应目录下的 built-in.o，除了 lib 目录下会有一个 lib.a
	init-y          := $(patsubst %/, %/built-in.o, $(init-y))
	core-y          := $(patsubst %/, %/built-in.o, $(core-y))
	drivers-y       := $(patsubst %/, %/built-in.o, $(drivers-y))
	net-y           := $(patsubst %/, %/built-in.o, $(net-y))
	libs-y1         := $(patsubst %/, %/lib.a, $(libs-y))
	libs-y2         := $(filter-out %.a, $(patsubst %/, %/built-in.o, $(libs-y)))
	virt-y          := $(patsubst %/, %/built-in.o, $(virt-y))

上面的代码指明了 $(vmlinux-deps) 的定义，即各个子目录下的 built-in.o 和若干 lib.a。
下面看看如何处理它(rule):

	# The actual objects are generated when descending,
	# make sure no implicit rule kicks in
	# 参考 GNU make 文档的 “4.10 Multiple Targets in a Rule”。所以，最终落到了 vmlinux-dirs 的处理上！
	$(sort $(vmlinux-deps)): $(vmlinux-dirs) ;

	# 将上面定义的各目录路径名后的 “/” 删除，得到 vmlinux-dirs
	vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
                     $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
                     $(net-y) $(net-m) $(libs-y) $(libs-m) $(virt-y)))

	# Handle descending into subdirectories listed in $(vmlinux-dirs)
	# Preset locale variables to speed up the build process. Limit locale
	# tweaks to this spot to avoid wrong language settings when running
	# make menuconfig etc.
	# Error messages still appears in the original language
	$(vmlinux-dirs): prepare scripts
        	$(Q)$(MAKE) $(build)=$@

代码一堆，不如一张图来清晰的说明其依赖关系:
part2:
![vmlinux-2](res/vmlinux-2.png  "vmlinux_process_2")

可以看出，在执行 vmlinux-dirs 的 recipe 之前有一堆的 prepare 工作要做，最重要的就是 archprepare，即 part3 的部分：

![vmlinux-3](res/vmlinux-3.png  "vmlinux_process_3")

archprepare 下面一大块工作又是由 prepare1 来完成，即 part4:

![vmlinux-4](res/vmlinux-4.png  "vmlinux_process_4")

巨长的流程，也可以铁杵磨成针。按照每一项任务的先后顺序(深度优先)来介绍。

### archheaders & archscripts

Target "archheaders" & "archscripts" 都定义在 arch/x86/Makefile 中，他们的定义都很清晰。archheaders 用于生成 arch/x86/include/generated/ 下的部分头文件；archscripts 用于生成 arch/x86/tools 下的 hostprogram: relocs。他们的 recipe 很简洁明了：

	$(Q)$(MAKE) $(build)=arch/x86/entry/syscalls all
	$(Q)$(MAKE) $(build)=arch/x86/tools relocs

archheaders 完成如下事情
创建2个文件夹：

>arch/x86/include/generated/asm
arch/x86/include/generated/uapi/asm

然后在 arch/x86/entry/syscalls 中，以 syscallhdr.sh & syscalltbl.sh 作为工具，分别处理 syscall_64.tbl & syscall_32.tbl， 生成：

>arch/x86/include/generated/uapi/asm/unistd_32.h
arch/x86/include/generated/uapi/asm/unistd_64.h
arch/x86/include/generated/uapi/asm/unistd_x32.h
&
arch/x86/include/generated/asm/syscalls_32.h
arch/x86/include/generated/asm/unistd_32_ia32.h (if defined CONFIG_X86_64)
arch/x86/include/generated/asm/unistd_64_x32.h (if defined CONFIG_X86_64)
arch/x86/include/generated/asm/syscalls_64.h (if defined CONFIG_X86_64)
arch/x86/include/generated/asm/xen-hypercalls.h (if defined CONFIG_XEN)

archscripts 完成的事情只是生成一个 host program: relocs

### include/config/auto.conf

flowchart 图中没有列出它后面的处理，实际后面有一些过程。它在 top Makefile 中 match 了下面这条 rule:

	# To avoid any implicit rule to kick in, define an empty command
	$(KCONFIG_CONFIG) include/config/auto.conf.cmd: ;

	# If .config is newer than include/config/auto.conf, someone tinkered
	# with it and forgot to run make oldconfig.
	# if auto.conf.cmd is missing then we are probably in a cleaned tree so
	# we execute the config step to be sure to catch updated Kconfig files
	include/config/%.conf: $(KCONFIG_CONFIG) include/config/auto.conf.cmd
		$(Q)$(MAKE) -f $(srctree)/Makefile silentoldconfig

在 clean src 配置完成后，auto.conf.cmd 是不存在的，是由上面的 recipe 生成。又一次 recursive make，调用自己完成 target silentoldconfig。由上一片文章可知，silentoldconfig 将 match 这条 rule:

	%config: scripts_basic outputmakefile FORCE
		$(Q)$(MAKE) $(build)=scripts/kconfig $@

所以，直接看 scripts/kconfig/Makefile 中的处理：

	silentoldconfig: $(obj)/conf
		$(Q)mkdir -p include/config include/generated
		$(Q)test -e include/generated/autoksyms.h || \
			touch   include/generated/autoksyms.h
		$< $(silent) --$@ $(Kconfig)

	hostprogs-y := conf nconf mconf kxgettext qconf gconf
	conf-objs       := conf.o  zconf.tab.o

部分细节过程可以直接参考上一篇文章。silentconfig 最终要执行的命令行是：

	scripts/kconfig/conf $(silent) --silentoldconfig Kconfig

同时创建了几个文件夹和文件：

>include/config
include/generated
include/generated/autoksyms.h

由可执行程序 conf 的 main 函数(conf.c)可以看到， 只有在 silentoldconfig 时， 才会调用函数 conf_write_autoconf，在此函数中会生成:

>include/config/auto.conf.cmd
include/generated/autoconf.h
include/config/tristate.conf
include/config/auto.conf

这里埋下一个小问题：从内容上看，auto.conf 和 tristate.conf 的内容有重复，有不同，他们是什么关系？

### include/config/kernel.release
Top Makefile 中：

	# Store (new) KERNELRELEASE string in include/config/kernel.release
	include/config/kernel.release: include/config/auto.conf FORCE
		$(call filechk,kernel.release)

	define filechk_kernel.release
		echo "$(KERNELVERSION)$$($(CONFIG_SHELL) $(srctree)/scripts/setlocalversion $(srctree))"
	endef

filechk 函数定义在 scripts/Kbuild.include 中：

	# filechk is used to check if the content of a generated file is updated.
	# Sample usage:
	# define filechk_sample
	#       echo $KERNELRELEASE
	# endef
	# version.h : Makefile
	#       $(call filechk,sample)
	# The rule defined shall write to stdout the content of the new file.
	# The existing file will be compared with the new one.
	# - If no file exist it is created
	# - If the content differ the new file is used
	# - If they are equal no change, and no timestamp update
	# - stdin is piped in from the first prerequisite ($<) so one has
	#   to specify a valid file as first prerequisite (often the kbuild file)
	define filechk
        	$(Q)set -e;                             \
        	$(kecho) '  CHK     $@';                \
       		mkdir -p $(dir $@);                     \
        	$(filechk_$(1)) < $< > $@.tmp;          \
        	if [ -r $@ ] && cmp -s $@ $@.tmp; then  \
                	rm -f $@.tmp;                   \
        	else                                    \
                	$(kecho) '  UPD     $@';        \
                	mv -f $@.tmp $@;                \
        	fi
	endef

生成 include/config/kernel.release。以我本地为例，其内容是: 4.14.0-rc6+，“+” 是脚本 scripts/setlocalversion 的结果。

### asm-generic 做什么

Target "asm-generic" 的处理近几个月发生了一些小小变化，有兴趣的同学请自行 git blame 细节。
以前，[long_recipe_0] 和 [long_recipe_1] 都是 asm-generic 的 recipe。先看两条 recipe 的定义：

[long_recipe_0]: $(Q)$(MAKE) -f $(srctree)/scripts/Makefile.asm-generic src=asm obj=arch/$(SRCARCH)/include/generated/asm
[long_recipe_1]: $(Q)$(MAKE) -f $(srctree)/scripts/Makefile.asm-generic src=uapi/asm obj=arch/$(SRCARCH)/include/generated/uapi/asm

每个 arch 有自己的专用的头文件, 位于 arch/$(SRCARCH)/include 目录, 此目录下还有 asm, uapi, generated 3个目录.
其中 generated 目录下的内容是编译过程中创建的，部分内容在 target "archheaders" 的处理流程中完成；“asm-generic” 负责生成其他的内容。

在 include/asm-generic 目录下存放的是一些 common 的头文件, 如注释所说:

>include/asm-generic contains a lot of files that are used verbatim by several architectures.

asm-generic 的做的事情是在 arch/$(SRCARCH)/include/generated 目录下创建一些 wrapper 文件, wrap include/asm-generic 下的一些文件，具体文件由 arch/x86/include/asm/Kbuild & arch/x86/include/uapi/asm/Kbuild 中定义的 generic-y 指定。
上述两个 Kbuild 文件中的 generated-y 指的是 archheaders 过程中生成的那些头文件。

对于这些变量的详细介绍，参考 “7 Kbuild syntax for exported headers” of Documentation/kbuild/makefiles.txt

### What is $(version_h)

变量 *version_h* 的定义和处理在 top Makefile 中：

	version_h := include/generated/uapi/linux/version.h
	old_version_h := include/linux/version.h

	define filechk_version.h
	        (echo \#define LINUX_VERSION_CODE $(shell                         \
	        expr $(VERSION) \* 65536 + 0$(PATCHLEVEL) \* 256 + 0$(SUBLEVEL)); \
	        echo '#define KERNEL_VERSION(a,b,c) (((a) << 16) + ((b) << 8) + (c))';)
	endef

只是简单的生成 include/generated/uapi/linux/version.h

### include/generated/utsrelease.h

生成此头文件，具体处理在 top makefile 中：

	include/generated/utsrelease.h: include/config/kernel.release FORCE
        	$(call filechk,utsrelease.h)

	# KERNELRELEASE can change from a few different places, meaning version.h
	# needs to be updated, so this check is forced on all builds
	uts_len := 64
	define filechk_utsrelease.h
        	if [ `echo -n "$(KERNELRELEASE)" | wc -c ` -gt $(uts_len) ]; then \
        	  echo '"$(KERNELRELEASE)" exceeds $(uts_len) characters' >&2;    \
        	  exit 1;                                                         \
        	fi;                                                               \
        	(echo \#define UTS_RELEASE \"$(KERNELRELEASE)\";)
	endef

这里有个 trick：rule 写的清楚，utsrelease.h 依赖于 kernel.release，但是，why？ 从 filechk_utsrelease.h 可以看出，正常情况下就是执行一句：

	echo \#define UTS_RELEASE \"$(KERNELRELEASE)\"

并且，KERNELRELEASE 在 top makefile 中的定义(很上面)如下：

	# Read KERNELRELEASE from include/config/kernel.release (if it exists)
	KERNELRELEASE = $(shell cat include/config/kernel.release 2> /dev/null)

但是第一次编译 kernel 前，kernel.release 文件明显是不存在的，所以 KERNELRELEASE 的值为空？可以知道它的值并不为空。这里的赋值操作符是 “=”，不是“：=”，这就是 trick 所在。参考 "6.2 The Two Flavors of Variables" of GNU 文档。

### gcc-plugins

Top Makefile 中 include 了 scripts/Makefile.gcc-plugins，target "gcc-plugins" 其中 。
关于 kbuild 对 gcc-plugins 的支持，参考： Documentation/gcc-plugins.txt。

gcc-plugin 以 .so 的形式存在，作为 gcc 的参数来使用：

	-fplugin=/path/to/name.so -fplugin-arg-name-key1[=value1] 

关于 gcc-plugin 本身的一切信息，在：[GCC 官方文档](https://gcc.gnu.org/onlinedocs/gccint/Plugins.html) 。

既然 gcc-plugin 以 .so 的形式存在 scripts/gcc-plugins 目录，那么在 kbuild 系统中它就是一个 host lib，所有编译相关的细节处理在 scripts/Makefile.host 中。

不同版本的 gcc 可能是被不同的编译器编译(gcc or g++)出来的，那么对应的plugin也要使用那个编译器。比如 gcc 4.8 以上都要使用 g++。

Target "gcc-plugins" 的作用是编译出 scripts/gcc-plugins 目录下的所有 plugin。

### $(vmlinux-dirs)

至此，$(vmlinux-dirs) 下的主要过程已介绍完，剩下的工作就是要编译 vmlinux 了，即执行：

	$(Q)$(MAKE) $(build)=$@

根据上文的分析，已知变量 vmlinux-dirs 的值是各个文件夹(with trailing "/" stripped)，具体的处理框架同 “make menuconfig” 文中的描述一致，编译进入 scripts/Makefile.build，其中依次 include:

>-include include/config/auto.conf # 配置已完成，auto.conf是存在的
include scripts/Kbuild.include
include $(kbuild-file) # kbuild makefile, 本例是 init/Makefile
include scripts/Makefile.lib # 对 kbuild makefile 中定义的通用变量(如 obj-y, obj-m 等) 进行处理

下面的代码分析来自上述这些 makeifle。

因为没有指定 target，所以 scripts/Makefile.build 中的第一条 rule(__build:)，即 default goal:

	__build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
	        $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
	        $(subdir-ym) $(always)
	        @:

	ifneq ($(strip $(obj-y) $(obj-m) $(obj-) $(subdir-m) $(lib-target)),)
	builtin-target := $(obj)/built-in.o
	endif

	$(builtin-target): $(obj-y) FORCE
		$(call if_changed,link_o_target)

	ifneq ($(strip $(lib-y) $(lib-m) $(lib-)),)
	lib-target := $(obj)/lib.a
	obj-y += $(obj)/lib-ksyms.o
	endif

	modorder-target := $(obj)/modules.order

"__build" 作为 default goal， 在我们的假设场景下，它的 prerequisites 就是所有列出的，但我们下面只详细介绍关键部分。

obj-y，obj-m，extra-y，always 的定义基本都在 kbuild makefile 中(在 “3 The kbuild files” of Documentation/kbuild/makefiles.txt 有介绍 obj-y，obj-m，**必须**先阅读掌握)，subdir-ym 的定义在 scripts/Makefile.lib 中。

kbuild 中定义的 obj-y，obj-m 变量中有三种可能：

1. single object(由一个 .c 编译而来的 .o)
2. composite object(由多个 .c 编译而来的 .o)
3. sub-directory

kbuild 系统对这些变量的处理主要在 scripts/Makefile.lib 中：

	# Handle objects in subdirs
	# ---------------------------------------------------------------------------
	# o if we encounter foo/ in $(obj-y), replace it by foo/built-in.o
	#   and add the directory to the list of dirs to descend into: $(subdir-y)
	# o if we encounter foo/ in $(obj-m), remove it from $(obj-m)
	#   and add the directory to the list of dirs to descend into: $(subdir-m)
	# 经过下面代码的处理，定义在 obj-y 和 obj-m 中的文件夹被撸出到 subdir-ym；
	# obj-y 中的文件夹被替换为 foo/built-in.o，也就是说，当前目录的 built-in.o 中包含了所有子目录中的 built-in.o；
	# obj-m 中的文件夹被剔除。上面的注释也描述的很清楚。
	__subdir-y      := $(patsubst %/,%,$(filter %/, $(obj-y)))
	subdir-y        += $(__subdir-y)
	__subdir-m      := $(patsubst %/,%,$(filter %/, $(obj-m)))
	subdir-m        += $(__subdir-m)
	obj-y           := $(patsubst %/, %/built-in.o, $(obj-y))
	obj-m           := $(filter-out %/, $(obj-m))
	# 经过上面的处理，obj-y 目前内容是 single object，composite object，sub-dir/built-in.o;
	# obj-m 目前的内容是 single object， composite object；
	# subdir-y 的内容是 obj-y 中的目录； subdir-m 的内容是 obj-m 中的文件夹

	# Subdirectories we need to descend into
	subdir-ym       := $(sort $(subdir-y) $(subdir-m))


	# if $(foo-objs), $(foo-y), or $(foo-m) exists, foo.o is a composite object
	# 在 kbuild 中有时你会发现，赋值给 obj-y 的 foo.o，并不存在它对应的 foo.c，而可能存在 foo-objs，foo-y，foo-m 的定义，
	# 这样的 foo.o 被称作 composite object，即一个 foo.o 由多个 .o 合并而来。
	# 这样做的意义可能是为了逻辑清晰，因为有时某个功能可能由一个 .c 文件实现，有时则由多个 .c。
	# 下面的前三行代码将 obj-y 和 obj-m 中 composite object 的 .o 撸出到 multi-used。
	# 因为 obj-m 中现在只可能有  single object 和 composite object，第四行将 single object 挑出来。
	multi-used-y := $(sort $(foreach m,$(obj-y), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y))), $(m))))
	multi-used-m := $(sort $(foreach m,$(obj-m), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y)) $($(m:.o=-m))), $(m))))
	multi-used   := $(multi-used-y) $(multi-used-m)
	single-used-m := $(sort $(filter-out $(multi-used-m),$(obj-m)))

	# Build list of the parts of our composite objects, our composite
	# objects depend on those (obviously)
	# 从 composite object 分别撸出所有 .o 的列表。
	multi-objs-y := $(foreach m, $(multi-used-y), $($(m:.o=-objs)) $($(m:.o=-y)))
	multi-objs-m := $(foreach m, $(multi-used-m), $($(m:.o=-objs)) $($(m:.o=-y)))
	multi-objs   := $(multi-objs-y) $(multi-objs-m)

	# $(subdir-obj-y) is the list of objects in $(obj-y) which uses dir/ to tell kbuild to descend
	# subdir-obj-y 的值是所有的 foo/built-in.o
	subdir-obj-y := $(filter %/built-in.o, $(obj-y))

	# Replace multi-part objects by their individual parts, look at local dir only
	# 下面两个变量表示当前目录中所有需要编译得到的.o(composite object 和 single object)
	real-objs-y := $(foreach m, $(filter-out $(subdir-obj-y), $(obj-y)), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y))),$($(m:.o=-objs)) $($(m:.o=-y)),$(m))) $(extra-y)
	real-objs-m := $(foreach m, $(obj-m), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y)) $($(m:.o=-m))),$($(m:.o=-objs)) $($(m:.o=-y)) $($(m:.o=-m)),$(m)))

	# 下面还有一段代码给上述所有变量加上 $(obj) 前缀，此处不赘述。

现在回头看"__build"的 prerequisites的处理，即：$(builtin-target)， $(lib-target)， $(extra-y)，$(obj-m)， $(modorder-target)，$(subdir-ym) $(always)。

#### $(builtin-target)：

	# 经过 Makefile.lib 的处理，obj-y 中只可能有 single object, composite object，foo/built-in.o
	builtin-target := $(obj)/built-in.o
	$(builtin-target): $(obj-y) FORCE
		$(call if_changed,link_o_target)

	# If the list of objects to link is empty, just create an empty built-in.o
	cmd_link_o_target = $(if $(strip $(obj-y)),\
	                      $(cmd_make_builtin) $@ $(filter $(obj-y), $^) \
	                      $(cmd_secanalysis),\
	                      $(cmd_make_empty_builtin) $@)

	# 1. single object 的处理最简单，直接 match 下面这条 rule。所有 .c --> .o 的处理都 match 这条(比如 composite object 包含的所有 .o)
	# Built-in and composite module parts
	$(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_dep) FORCE
        $(call cmd,force_checksrc)
        $(call if_changed_rule,cc_o_c)

	define rule_cc_o_c
	        $(call echo-cmd,checksrc) $(cmd_checksrc)                         \
	        $(call cmd_and_fixdep,cc_o_c)                                     \
	        $(cmd_modversions_c)                                              \
	        $(call echo-cmd,objtool) $(cmd_objtool)                           \
	        $(call echo-cmd,record_mcount) $(cmd_record_mcount)
	endef

	cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<

	# 下面几行代码定义在 scripts/Kbuild.include
	# Usage: $(call if_changed_rule,foo)
	# Will check if $(cmd_foo) or any of the prerequisites changed,
	# and if so will execute $(rule_foo).
	if_changed_rule = $(if $(strip $(any-prereq) $(arg-check) ),                 \
	        @set -e;                                                             \
	        $(rule_$(1)), @:)

	#上面的代码可以看出，$(call cmd_and_fixdep,cc_o_c 是核心，完成编译.c 的动作。

	# 2. obj-y 和 obj-m 中的 composite object 则 match 下面2条 rule。函数 multi_depend 的用法在"make menuconfig"的文章已介绍过。
	$(multi-used-y): FORCE
        	$(call if_changed,link_multi-y)
	$(call multi_depend, $(multi-used-y), .o, -objs -y)

	$(multi-used-m): FORCE
        	$(call if_changed,link_multi-m)
        	@{ echo $(@:.o=.ko); echo $(link_multi_deps); \
        	   $(cmd_undef_syms); } > $(MODVERDIR)/$(@F:.o=.mod)
	$(call multi_depend, $(multi-used-m), .o, -objs -y -m)
	
	# 一般情况下，上面2条 rule 的 recipe 的核心(link_multi-y, link_multi-m)最后都落在这一条(此处省略了一点分析)
	cmd_link_multi-link = $(LD) $(ld_flags) -r -o $@ $(link_multi_deps) $(cmd_secanalysis)
	
	# 3. foo/built-in.o match 下面这条
	# To build objects in subdirs, we need to descend into the directories
	$(sort $(subdir-obj-y)): $(subdir-ym) ;

	$(subdir-ym):
	        $(Q)$(MAKE) $(build)=$@
	# 怎么样，现在看起来是不是有点感觉了，又一次 recursive 的 make 了。
	# 就这样， $(obj)/built-in.o 依赖的所有 prerequisites 都准备好了，剩下的就是执行它的 recipe 了

总结：每一层文件夹下的所有 obj-y 生成 built-in.o，下一层的 built-in.o 会被 linked into 上一层的 built-in.o，最后 link 为 kernel src 目录下的 vmlinux 文件。

#### obj-m

经过 Makefile.lib 的处理，obj-m 中只剩下 single object 和 composite object，single object 的处理因有少许不同，所以并没有走普通 .o 的 rule，他们有自己的 rule 来处理：

	$(multi-used-m): FORCE
        	$(call if_changed,link_multi-m)
        	@{ echo $(@:.o=.ko); echo $(link_multi_deps); \
        	   $(cmd_undef_syms); } > $(MODVERDIR)/$(@F:.o=.mod)
	$(call multi_depend, $(multi-used-m), .o, -objs -y -m)

	$(single-used-m): $(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_dep) FORCE
	        $(call cmd,force_checksrc)
	        $(call if_changed_rule,cc_o_c)
	        @{ echo $(@:.o=.ko); echo $@; \
	           $(cmd_undef_syms); } > $(MODVERDIR)/$(@F:.o=.mod)
	#上面的 rule 的核心内容上和他们的同类是一样的。

	# MODVERDIR 定义在 top Makefile 中
	export MODVERDIR := $(if $(KBUILD_EXTMOD),$(firstword $(KBUILD_EXTMOD))/).tmp_versions

不同的是，会 echo 一些信息到 tmp_versions/ 目录下的同名文件中。
obj-m 中的每一个 single object 和 composite object 都是一个 module。 

#### $(modorder-target)

	modorder-target := $(obj)/modules.order

	$(modorder-target): $(subdir-ym) FORCE
		$(Q)(cat /dev/null; $(modorder-cmds)) > $@

	# Rule to create modules.order file
	#
	# Create commands to either record .ko file or cat modules.order from
	# a subdirectory
	modorder-cmds =                                         \
	        $(foreach m, $(modorder),                       \
	                $(if $(filter %/modules.order, $m),     \
	                        cat $m;, echo kernel/$m;))

	# 下面的代码定义在 scripts/Makefile.lib
	# Determine modorder.
	# Unfortunately, we don't have information about ordering between -y
	# and -m subdirs.  Just put -y's first.
	modorder := $(patsubst %/,%/modules.order, $(filter %/, $(obj-y)) $(obj-m:.o=.ko))
	# 这里有一个疑问：为什么子目录的处理不包含 obj-m？难道 obj-y 和 obj-m 的子目录一定相同？
 
从目录树的最底层开始，生成该目录下的 modules.order 文件。文件的内容包含下一层目录中同名文件的内容，和本层目录中的 $(obj-m:.o=.ko)。最底层的目录肯定是没有子目录的，所以只是 echo kernel/$m 到 modules.order 文件。

## bzImage


## modules