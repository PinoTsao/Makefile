# Note of analysis of Makefile

### Background knowledge tips on GNU make
Understanding of phony target is critical. From certain persective,
phony target determines the process of make.
Please refer "4.6 Phony Targets" of `info make` for its detailed explanation.
A summary of phony target is as following.

>A phony target is one that is not really the name of a file; rather it
>is just a name for a recipe to be executed when you make an explicit
>request.  There are two reasons to use a phony target: to avoid a
>conflict with a file of the same name, and to improve performance.

A typical phony target(rule) defination looks like this:

	.PHONY: clean
	clean:
		rm *.o temp

If you want the recipe to be executed, you must explicitely make request via `make clean`.

Phony target could be a prerequisite of other targets, either phony or real file target.
When it serves as prerequisite of another target, it acutally serves as a subroutine of the other.


### Process of targets
Generally, a target will have many prerequisites. Here, assuming we have a clean kernel
source code, and we are on a x86_64 platform, to analyse the process of different make targets.

When compiling from a clean source, first we need to `make *config`(we take `make menuconfig`
for example), then `make`, and then `make modules_install`,`make install`, so, we will analyse
the process of these targets. We will list all the rules needed by a target.

#### make menuconfig
When you do `make menuconfig`, the rule's chain looks as following:

	# [pino] In top makefile. [oniq]
	%config: scripts_basic outputmakefile FORCE
		$(Q)$(MAKE) $(build)=scripts/kconfig $@

*menuconfig* depends on *scripts_basic*, *outputmakefile*, *FORCE*

	scripts_basic:
		$(Q)$(MAKE) $(build)=scripts/basic
		$(Q)rm -f .tmp_quiet_recordmcount		

*build* is a variable defined in scripts/Kbuild.include

>build := -f $(srctree)/scripts/Makefile.build obj

So, the recipe becomes

	make -f $(srctree)/scripts/Makefile.build obj=scripts/basic

*outputmakefile* rule doesnt't work when we build from kernel source directory,
so, skip the analysis of it.

Then, the most relevant step seems is the recipe of the rule:

	$(Q)$(MAKE) $(build)=scripts/kconfig $@

which actually is:

	make -f $(srctree)/scripts/Makefile.build obj=scripts/kconfig menuconfig
	
Makefile.build looks like a general, universal makefile, the trick is: it receives the variable obj= when invokde by other makefile, then it will determine the where is kbuild target locates, refer these codes:

	# The filename Kbuild has precedence over Makefile
	kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
	kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
	include $(kbuild-file)

It means the target locates at the $kbuild-file in $kbuild-dir, take menu-config for example, it locates scripts/kconfig/Makefile.

Makefile.build will include several other makefiles, it inlcude the following by sequence:

>include scripts/Kbuild.include
>include $(kbuild-file)
>include scripts/Makefile.lib
>include scripts/Makefile.host

Target "menuconfig" is a host program(Refer "4 Host Program support" of Documentation/kbuild/makefiles.txt).

After the 4 makefiles above are included, the make process finally work.
To analyse the process of "make menuconfig", we will extract the key lines in each makefile as following.
I bet you won't have patience to read all the following makefile, so lets just give the result(steps) of "make menuconfig" at the end, and the contents here could be a quick reference.

scripts/kconfig/Makefile

	# [pino] src=obj=scripts/kconf [oniq]
	menuconfig: $(obj)/mconf
		$< $(silent) $(Kconfig)
	
	check-lxdialog  := $(srctree)/$(src)/lxdialog/check-lxdialog.sh
	lxdialog := lxdialog/checklist.o lxdialog/util.o lxdialog/inputbox.o
	lxdialog += lxdialog/textbox.o lxdialog/yesno.o lxdialog/menubox.o
	mconf-objs     := mconf.o zconf.tab.o $(lxdialog)
	
	hostprogs-y := conf nconf mconf kxgettext qconf gconf
	
	# Check that we have the required ncurses stuff installed for lxdialog (menuconfig)
	PHONY += $(obj)/dochecklxdialog
	$(addprefix $(obj)/, mconf.o $(lxdialog)): $(obj)/dochecklxdialog
	$(obj)/dochecklxdialog:
        	$(Q)$(CONFIG_SHELL) $(check-lxdialog) -check $(HOSTCC) $(HOST_EXTRACFLAGS) $(HOSTLOADLIBES_mconf)

scripts/Makefile.host

	__hostprogs := $(sort $(hostprogs-y) $(hostprogs-m))
	
	# C executables linked based on several .o files
	host-cmulti := $(foreach m,$(__hostprogs),\
				$(if $($(m)-cxxobjs),,$(if $($(m)-objs),$(m))))

	host-cmulti     := $(addprefix $(obj)/,$(host-cmulti))
	
	# Object (.o) files compiled from .c files
 	host-cobjs      := $(sort $(foreach m,$(__hostprogs),$($(m)-objs)))
 	# Link an executable based on list of .o files, all plain c
	# host-cmulti -> executable
	quiet_cmd_host-cmulti   = HOSTLD  $@
	cmd_host-cmulti   = $(HOSTCC) $(HOSTLDFLAGS) -o $@ \
					$(addprefix $(obj)/,$($(@F)-objs)) \
					$(HOST_LOADLIBES) $(HOSTLOADLIBES_$(@F))
	$(host-cmulti): FORCE
		$(call if_changed,host-cmulti)
		
	# Create .o file from a single .c file
	# host-cobjs -> .o
	quiet_cmd_host-cobjs    = HOSTCC  $@
		cmd_host-cobjs    = $(HOSTCC) $(hostc_flags) -c -o $@ $<
	$(host-cobjs): $(obj)/%.o: $(src)/%.c FORCE
		$(call if_changed_dep,host-cobjs)

scripts/Kbuild.include

	# Escape single quote for use in echo statements
	escsq = $(subst $(squote),'\$(squote)',$1)

	# echo command.
	# Short version is used, if $(quiet) equals `quiet_', otherwise full one.
	echo-cmd = $(if $($(quiet)cmd_$(1)),\
        	echo '  $(call escsq,$($(quiet)cmd_$(1)))$(echo-why)';)

	# if_changed      - execute command if any prerequisite is newer than
	#                           target, or command line has changed
	
	ifneq ($(KBUILD_NOCMDDEP),1)
	# Check if both arguments are the same including their order. Result is empty
	# string if equal. User may override this check using make KBUILD_NOCMDDEP=1
	arg-check = $(filter-out $(subst $(space),$(space_escape),$(strip $(cmd_$@))), \
        	                 $(subst $(space),$(space_escape),$(strip $(cmd_$1))))
	else
	arg-check = $(if $(strip $(cmd_$@)),,1)
	endif
	
	make-cmd = $(call escsq,$(subst \#,\\\#,$(subst $$,$$$$,$(cmd_$(1)))))
	
	# Find any prerequisites that is newer than target or that does not exist.
	# PHONY targets skipped in both cases.
	any-prereq = $(filter-out $(PHONY),$?) $(filter-out $(PHONY) $(wildcard $^),$^)
	
	# Execute command if command has changed or prerequisite(s) are updated.
	if_changed = $(if $(strip $(any-prereq) $(arg-check)),                       \
        	@set -e;                                                             \
        	$(echo-cmd) $(cmd_$(1));                                             \
        	printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd, @:)

The process/steps of "make menuconfig":

	# 1st rule
	menuconfig: $(obj)/mconf
		$< $(silent) $(Kconfig)
	
	# which actually is:
	menuconfig: scripts/kconfig/mconf
		scripts/kconfig/mconf Kconfig
		
The following is all the translation steps:

	hostprogs-y := conf nconf mconf kxgettext qconf gconf
	__hostprogs := $(sort $(hostprogs-y) $(hostprogs-m))

	# $host-cmulti will contains "mconf"
	host-cmulti := $(foreach m,$(__hostprogs),\
		$(if $($(m)-cxxobjs),,$(if $($(m)-objs),$(m))))
	
	# its value will become to scripts/kconfig/mconf
	host-cmulti     := $(addprefix $(obj)/,$(host-cmulti))

.

	# 2nd rule's recipe. Implicte rule will have the .c -> .o dependency.
	cmd_host-cmulti   = $(HOSTCC) $(HOSTLDFLAGS) -o $@ \
					$(addprefix $(obj)/,$($(@F)-objs)) \
					$(HOST_LOADLIBES) $(HOSTLOADLIBES_$(@F))

	# 2nd rule
	$(host-cmulti): FORCE
	        $(call if_changed,host-cmulti)
	
	# 3rd rule. Here overwrite the .c -> .o rule
	#cmd_host-cobjs    = $(HOSTCC) $(hostc_flags) -c -o $@ $<
	$(host-cobjs): $(obj)/%.o: $(src)/%.c FORCE
    		$(call if_changed_dep,host-cobjs)
	
	# extra rule to do the check
	$(addprefix $(obj)/, mconf.o $(lxdialog)): $(obj)/dochecklxdialog
	$(obj)/dochecklxdialog:
	$(Q)$(CONFIG_SHELL) $(check-lxdialog) -check $(HOSTCC) $(HOST_EXTRACFLAGS) $(HOSTLOADLIBES_mconf)
        
	# Now look how the recipe will be executed.
	# $(echo-cmd) will print the exectued cmd, $(cmd_$(1)) will execute the cmd_host-cmulti,
	# printf bluh... will record the executed cmd in the file $(dot-target).cmd.
	# So you can see .mconf.cmd .mconf.o.cmd in the scripts/kconfig/
	if_changed = $(if $(strip $(any-prereq) $(arg-check)),                       \
        	@set -e;                                                             \
        	$(echo-cmd) $(cmd_$(1));                                             \
        	printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd, @:)
	
Now scripts/kconfig/mconf has been compiled out, its recipe will be executed:

	scripts/kconfig/mconf Kconfig

Then we know the result, the **.config** file is produced.

#### make or "make all"

Assuming you have "make meuconfig", now all you have to do to build the kernel via "make" or "make all".

In top makefile, the 1st/default goal is *_all*

	ifeq ($(KBUILD_EXTMOD),)
	_all: all
	else
 	_all: modules
	endif

*_all* has prerequisite *all*, so *all* is the subroutine.

	# The all: target is the default when no target is given on the command line.
	# This allow a user to issue only 'make' to build a kernel including modules
	# Defaults to vmlinux, but the arch makefile usually adds further targets
	all: vmlinux
	.....
	ifdef CONFIG_MODULES
	# By default, build modules as well
	all: modules
	...
	
In arch/x86/Makefile, we have:

	# Default kernel to build
	all: bzImage

So, target *all* has prerequisite *vmlinux*, *bzImage*. For *vmlinux*, keep looking into top makefile:

	vmlinux: scripts/link-vmlinux.sh vmlinux_prereq $(vmlinux-deps) FORCE
		     +$(call if_changed,link-vmlinux)

vmlinux_prereq seems doesn't have neccessary relation with target *vmlinux*:

	vmlinux_prereq: $(vmlinux-deps) FORCE
	ifdef CONFIG_HEADERS_CHECK
        	$(Q)$(MAKE) -f $(srctree)/Makefile headers_check
	endif
	ifdef CONFIG_GDB_SCRIPTS
        	$(Q)ln -fsn `cd $(srctree) && /bin/pwd`/scripts/gdb/vmlinux-gdb.py
	endif
	ifdef CONFIG_TRIM_UNUSED_KSYMS
        	$(Q)$(CONFIG_SHELL) $(srctree)/scripts/adjust_autoksyms.sh \
        	  "$(MAKE) -f $(srctree)/Makefile vmlinux"
	endif

Skip it. Keep looking at $(vmlinux-deps)

	# Externally visible symbols (used by link-vmlinux.sh)
	export KBUILD_VMLINUX_INIT := $(head-y) $(init-y)
	export KBUILD_VMLINUX_MAIN := $(core-y) $(libs-y) $(drivers-y) $(net-y) $(virt-y)
	export KBUILD_LDS          := arch/$(SRCARCH)/kernel/vmlinux.lds
	vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN)
	
head-y, init-y is defined as following

	# in arch/x86/Makefile:
	head-y := arch/x86/kernel/head_$(BITS).o
	head-y += arch/x86/kernel/head$(BITS).o
	head-y += arch/x86/kernel/ebda.o
	head-y += arch/x86/kernel/platform-quirks.o

	init-y          := init/
	init-y          := $(patsubst %/, %/built-in.o, $(init-y))
	# [pino] So init-y actually evaluates to init/built-in.o
	
core-y, libs-y, drivers-y, net-y, virt-y is defined in top makefile as following:

	core-y          := usr/
	# in arch/x86/makefile
	core-y += arch/x86/
	core-y          += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/
	core-y          := $(patsubst %/, %/built-in.o, $(core-y))

	libs-y          := lib/
	# in arch/x86/makefile
	libs-y  += arch/x86/lib/
	libs-y1         := $(patsubst %/, %/lib.a, $(libs-y))
	libs-y2         := $(patsubst %/, %/built-in.o, $(libs-y))
	libs-y          := $(libs-y1) $(libs-y2)

	drivers-y       := drivers/ sound/ firmware/
	drivers-y       := $(patsubst %/, %/built-in.o, $(drivers-y))
	# there are addtional definition in arch/x86/makefile

	net-y           := net/
	net-y           := $(patsubst %/, %/built-in.o, $(net-y))

	virt-y          := virt/
	virt-y          := $(patsubst %/, %/built-in.o, $(virt-y))	

In a word, $(vmlinux-deps) are the real file.
Then we have

	vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
	                      $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
	                      $(net-y) $(net-m) $(libs-y) $(libs-m) $(virt-y)))
	# for example, foo/bar/ will be foo/bar
		    
	# The actual objects are generated when descending,
	# make sure no implicit rule kicks in
	$(sort $(vmlinux-deps)): $(vmlinux-dirs) ;
 
	PHONY += $(vmlinux-dirs)
	$(vmlinux-dirs): prepare scripts
	         $(Q)$(MAKE) $(build)=$@

	# All the preparing..
	prepare: prepare0 prepare-objtool
	
	prepare0: archprepare gcc-plugins
		         $(Q)$(MAKE) $(build)=.
	prepare-objtool: $(objtool_target)
	archprepare: archheaders archscripts prepare1 scripts_basic


### Suspicious cleanup

>hostlibs, hostlibs-y, hostlibs-m

