# What has `make [all]` done

配置结束后，作为想编译内核的小白，自然是直接敲入 make，或者 make all, 二者是一样的效果。
Makefile 中的第一个 target 是 make 的 default goal。在 top Makefile 中，第一个 target 是：

	# That's our default target when none is given on the command line
	PHONY := _all
	_all:

_all 更的作用更像是个 placeholder，因为真正作用的 target 都是它的 prerequisites。当不加任何参数执行 make 时，这可能是整个编译过程最复杂的一步(这也意味着，本文可能无比的长)，用流程图来观察会更清晰；又因为流程无比的长，所以一张图是不够的，将分成几个 part。 

Part1：
![vmlinux-1](res/vmlinux-1.png  "vmlinux_process_1")

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