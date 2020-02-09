# UIS8910DM平台的编译系统

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
&emsp;&emsp;在SDK根目录下的`CMakeLists.txt`中，通过以下脚本将KConfig完成的配置转化为cmake可以识别的配置：`target.cmake`，这一步在`cmake ../.. -G Ninja` 这一操作中完成的。

    # Process and include target config
    
    set(TARGET_CONFIG ${SOURCE_TOP_DIR}/target/${BUILD_TARGET}/target.config)
    set(TARGET_CMAKE ${BINARY_TOP_DIR}/target.cmake)
    execute_process(
        COMMAND python3 ${tools_dir}/cmakeconfig.py ${TARGET_CONFIG} ${TARGET_CMAKE}
        WORKING_DIRECTORY ${SOURCE_TOP_DIR}
    )
    include(${TARGET_CMAKE})

## cmake

&emsp;&emsp;cmake是一种更高层次的构建工具，也是更为通用的构建工具，基于它可以构建跨平台的编译环境。  
&emsp;&emsp;本平台通过`cmake/extension.cmake`这个文件扩展很多cmake的功能函数。比如relative_glob、beautify_c_code等等。基于cmake的这些功能，将每个软件功能模块各自生成为一个静态库，最后链接这些静态库生成最终的目标固件。基本上components下的每个子目录都会生成一个或者多个静态库，生成的静态库位于`out/<project_target>/lib`目录下。  
&emsp;&emsp;最终固件的生成脚本位于SDK主目录的`CMakeLists.txt`中，从这段脚本上看，我们可以为一个项目配置多个不同的NV，以适配不同的RF硬件，该编译系统能把适用于不同RF硬件的固件都生成出来。

    # Create pac for all variants
    foreach(nvmvariant ${CONFIG_NVM_VARIANTS}) build_modem_image(${nvmvariant})
        set(nvname ${NVM_VARIANT_${nvmvariant}_NVMITEM})
        set(pac_config ${out_hex_dir}/${nvmvariant}.json)
        set(pac_file ${out_hex_dir}/${BUILD_TARGET}-${nvmvariant}-${nvname}_${BUILD_RELEASE_TYPE}.pac) pac_init_fdl(init_fdl ${pac_config})
        pac_nvitem_8910(nvitem_8910 ${pac_config})
        if(DEFINED package_file_depends) 
            set(pac_package_file cfg-pack-cpio -i PACKAGE_FILE -p ${out_hex_dir}/${package_file_cpio} ${pac_config}) 
        endif()
        execute_process(
            COMMAND python3 ${pacgen_py} ${init_fdl} ${nvitem_8910} 
            cfg-image -i BOOTLOADER -a ${CONFIG_BOOT_FLASH_ADDRESS} -s ${CONFIG_BOOT_FLASH_SIZE}
                    -p ${out_hex_dir}/boot.sign.img ${pac_config} 
            cfg-image -i AP -a ${CONFIG_APP_FLASH_ADDRESS} -s ${CONFIG_APP_FLASH_SIZE}
                    -p ${out_hex_dir}/${BUILD_TARGET}.sign.img ${pac_config} 
            cfg-image -i PS -a ${CONFIG_FS_MODEM_FLASH_ADDRESS} -s ${CONFIG_FS_MODEM_FLASH_SIZE}
                    -p ${out_hex_dir}/${nvmvariant}.img ${pac_config} 
            ${pac_package_file}
            cfg-clear-nv ${pac_config}
            cfg-phase-check ${pac_config}
            cfg-nv -s ${CONFIG_NVBIN_FIXED_SIZE} -p ${out_hex_dir}/${nvmvariant}_nvitem.bin ${pac_config} 
            dep-gen --base ${SOURCE_TOP_DIR} ${pac_config} 
            OUTPUT_VARIABLE pac_dep
            OUTPUT_STRIP_TRAILING_WHITESPACE
            WORKING_DIRECTORY ${SOURCE_TOP_DIR} 
        )
        add_custom_command(OUTPUT ${pac_file} 
            COMMAND python3 ${pacgen_py} pac-gen ${pac_config} ${pac_file}
            DEPENDS ${pacgen_py} ${pac_config} ${pac_dep} 
            WORKING_DIRECTORY ${SOURCE_TOP_DIR} 
        )
        add_custom_target(${nvmvariant}_pacgen ALL DEPENDS ${pac_file}) 
    endforeach() 

&emsp;&emsp;为了在C/C++代码中引入开关配置，本平台采用了cmake的`configure_file`机制。通过这个机制，可以将KConfig中定义的配置信息（通过`target.cmake`）引入到C/C++代码中。  
比如hal_config.h.in中的 `#cmakedefine CONFIG_APPIMG_LOAD_FLASH`，如果KConfig中`CONFIG_APPIMG_LOAD_FLASH`开启，则C/C++代码中`CONFIG_APPIMG_LOAD_FLASH`宏就会被定义。  
又比如atr_config.h.in中的`#cmakedefine CONFIG_ATR_URC_BUFF_SIZE @CONFIG_ATR_URC_BUFF_SIZE@`，在C/C++代码中的CONFIG_ATR_URC_BUFF_SIZE宏就引用了KConfig中定义的值。  
&emsp;&emsp;**这种做法有个不好地方就是：C/C++代码中引用这些宏的地方需要手动 include 相应的.h文件（通过.h.in生成的，位于`out/<project_target>/include`目录）。**

## Ninja

&emsp;&emsp;Ninja是一种类似于 make 的构建工具，它的主要特点是通过编译任务并行组织，大大提高了构建速度。关于 Ninja 的详细知识请查看相关网页，这里就不做描述。因为在本平台中，所有需要手动编写的编译脚本都是 cmake 脚本，通过 cmake 的 -G 命令将相关的cmake脚本转化为 Ninja 脚本，然后通过 Ninja 构建最终的目标固件。

&emsp;&emsp;以上是本人关于UIS8910DM平台编译系统粗浅描述，不足和有误的地方，望不吝指教。
