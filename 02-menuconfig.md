# Flowchart of make menuconfig

内核编译的第一步永远是配置: make *config。有各种样子的 config 的 target:

	config          - Update current config utilising a line-oriented program
	nconfig         - Update current config utilising a ncurses menu based program
	menuconfig      - Update current config utilising a menu based program
	xconfig	        - Update current config utilising a Qt based front-end
	gconfig	        - Update current config utilising a GTK+ based front-end
	oldconfig	    - Update current config utilising a provided .config as base
	localmodconfig  - Update current config disabling modules not loaded
	localyesconfig  - Update current config converting local mods to core
	silentoldconfig - Same as oldconfig, but quietly, additionally update deps
	defconfig	    - New config with default from ARCH supplied defconfig
	savedefconfig   - Save current config as ./defconfig (minimal config)
	allnoconfig     - New config where all options are answered with no
	allyesconfig    - New config where all options are accepted with yes
	allmodconfig    - New config selecting modules when possible
	alldefconfig    - New config with all symbols set to default
	randconfig      - New config with random answer to all options
	listnewconfig   - List new options
	olddefconfig    - Same as silentoldconfig but sets new symbols to their default value
	kvmconfig       - Enable additional options for kvm guest kernel support
	xenconfig       - Enable additional options for xen dom0 and guest kernel support
	tinyconfig      - Configure the tiniest possible kernel

他们 match 了 top makefile 中的这条 rule：

	config: scripts_basic outputmakefile FORCE
		$(Q)$(MAKE) $(build)=scripts/kconfig $@
 
	%config: scripts_basic outputmakefile FORCE
		$(Q)$(MAKE) $(build)=scripts/kconfig $@
		
变量 build 定义在 scripts/Kbuild.include 中：

	build := -f $(srctree)/scripts/Makefile.build obj

所以当执行 make *config 时，其实是执行下面这条命令

	make -f $(srctree)/scripts/Makefile.build obj=scripts/kconfig *config
	
由于 target “*config” 还有很多 prerequisites, 所以我们以 menuconfig 为例，画出他们的流程图。

在 Top makefile 中：

![config](/home/pino/Pictures/menuconfig.png  "target config")

Target "scripts_basic" 被很多 target 依赖，它的作用是生成整个内核编译过程中所需要的 basic program: fixdep & bin2c。其实 target “*config” 最终也是执行一个 host program，用来生成配置文件 .config 等，后面将以它为例介绍编译 host program 的详细过程。

所以，make menuconfig 的过程就剩下执行 config 的 recipe，也就是进入 scripts/kconfig/Makefile 中寻找真正的 target： menuconfig，在此 makefile 中有：

	menuconfig: $(obj)/mconf
		$< $(silent) $(Kconfig)

	hostprogs-y := conf nconf mconf kxgettext qconf gconf
	mconf-objs     := mconf.o zconf.tab.o $(lxdialog)

	lxdialog := lxdialog/checklist.o lxdialog/util.o lxdialog/inputbox.o
	lxdialog += lxdialog/textbox.o lxdialog/yesno.o lxdialog/menubox.o

看起来 menuconfig 的 target-prerequisite 关系流程很简单，仅仅是使用一个叫做 mconf 的 host program。那么剩下的问题就分为了 2 个：

1. host program "mconf" 是如何生成的？
2. mconf 如何执行生成 .config 等配置文件的？