# <center>UIS8910DM平台的编译系统</center>

----------
&emsp;&emsp;这里所说的编译系统是一种笼统的说法，大体上包含构建系统和编译工具集合。编译工具集合就是大家熟悉的编译器、汇编器、连接器等，该平台使用的是GCC，具体路径位于prebuilt/win32/gcc-arm-none-eabi，这里就不多说了。下面我们主要讲讲该平台的构建系统（build system）。

&emsp;&emsp;讲到构建系统，大家比较熟悉就是makefile了，它通过Makefile语言编写的脚本，组织代码、资源，调用编译工具集合及其它工具，共同完成最终目标的实现。这个最终目标可以是库文件、可执行的应用软件、不同格式的可烧写的嵌入式固件等等。实际上，每个IDE里都有一套自己的构建工具，比如VS的nmake，注重效率的Ninja，Mac系统的xcode等等。有些软件工程直接通过一个单一的构建工具完成系统构建，优点是简单，缺点是缺乏灵活、跨平台移值工作量大。Linux内核是通过Kconfig和makefile组成构建系统的，kconfig用于配置、make主导生成最终的内核镜像。对于那些希望实现跨平台的软件工程而言，直接使用makefile明显带来移值工作的大量增加，为了解决这个问题，出现如cmake、autotool这样的工具，这些工具的语言语法相对简单，通过它们自动生成makefile、nmake、Ninja、xcode所对应的脚本，完成最终目标的构建。也就说，cmake、autotool这样的工具是位于make、nmake、Ninja、xcode等工具的上层。

&emsp;&emsp;UIS8910DM平台为了实现固件功能的可灵活配置、同时支持linux和Windows的编译环境、又能提高编译效率，采用 KConfig + cmake + Ninja组成它的构建系统。


## KConfig
&emsp;&emsp;KConfig最大的作用就是实现软件项目的配置。在linux内核中，KConfig在编译前会生成2个文件：一个是autoconf.h，用于C代码的宏定义；一个是？？？，是makefile配置项。通过这两个文件，实现对软件项目的配置。有人会说。通过简单自己手写Makefile和.h文件，也可以实现这些功能，这是没错的，但这样会带来有几个问题：

1. 没有可视化的工具进行清晰可见的配置。  
2. 在配置时需要人为考虑依赖关系，比如需要支持USB storage功能，前提是USB功能，通过KConfig可以清晰得知这种依赖关系，甚至在配置时自动依据这个关系完成相关配置。  
3. 自动同步Makefile和.h文件关联性。

&emsp;&emsp;当然，KConfig不单单可以导出makefile脚本，还可以导出cmake脚本，本平台就是通过`kconfiglib.py`这个脚本内的功能导出相应的cmake脚本：`target.cmake`。

&emsp;&emsp;本平台通过运行menuconfig.bat来对项目进行可视化的配置（命令是 `menuconfig.bat <target_name>`），配置完成后会生成或者更新`target/<target_name>/`目录下`target.config`文件，通过`minconfig.py`脚本可以将`target.config`中与默认配置相同的选项清除掉，仅保留与默认配置不一致的项。
在SDK根目录下的`CMakeLists.txt`中，通过以下脚本将KConfig完成的配置转化为cmake可以识别的配置：`target.cmake`，这一步在`cmake ../.. -G Ninja` 这一操作中完成的。
	
 	# Process and include target config
	
	set(TARGET_CONFIG ${SOURCE_TOP_DIR}/target/${BUILD_TARGET}/target.config)
	set(TARGET_CMAKE ${BINARY_TOP_DIR}/target.cmake)
	execute_process(
    COMMAND python3 ${tools_dir}/cmakeconfig.py ${TARGET_CONFIG} ${TARGET_CMAKE}
    WORKING_DIRECTORY ${SOURCE_TOP_DIR}
	)
	include(${TARGET_CMAKE})

## cmake
&emsp;&emsp;cmake是一种更高层次的构建工具，