# What has make [all] done

配置结束后，作为想编译内核的小白，自然是直接敲入 make，或者 make all, 二者是一样的效果。
Makefile 中的第一个 target 会作为 make 的 default goal，在 top Makefile 中，第一个 target 是：

	# That's our default target when none is given on the command line
	PHONY := _all
	_all:

_all 更的作用更像是个 placeholder，因为真正作用的 target 都是它的 prerequisites。当不加任何参数执行 make 时，这可能是整个编译过程最复杂的一步(这也意味着，本文可能无比的长)，用流程图来观察会更清晰；又因为流程无比的长，所以一张图是不够的。
