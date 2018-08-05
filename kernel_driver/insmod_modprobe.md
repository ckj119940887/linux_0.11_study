linux设备驱动有两种加载方式insmod和modprobe，下面谈谈它们用法上的区别

    1.insmod一次只能加载特定的一个设备驱动，且需要驱动的具体地址。写法为：
        insmod drv.ko

    2.modprobe则可以一次将有依赖关系的驱动全部加载到内核。不加驱动的具体地址，但需要在安装文件系统时是按
    照make modues_install的方式安装驱动模块的。写法为：
           modprob drv

    在写好hello.c和Makefile后，生成hello.ko，然后拷贝到/lib/modules/$(uname -r)/目录下
    执行depmod，会在/lib/modules/#uname -r#/目录下生成modules.dep和modules.dep.bb文件，表明模块的依赖关系
    modprobe hello（注意这里无需输入.ko后缀）

modprobe 和insmod一样都是用来加载内核module的,不过modprobe比较智能，它可以根据module的依赖性来自动为你加
载；而insmod就做不到这点。
