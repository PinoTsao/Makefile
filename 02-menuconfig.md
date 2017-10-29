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

