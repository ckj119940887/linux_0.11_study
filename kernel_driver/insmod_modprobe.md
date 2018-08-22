# 许可证
    MODULE_LECENSE("GPL");

# 符号导出
    将一个模块的符号导出到另一个模块

    EXPORT_SYMBOL(name);
    EXPORT_SYMBOL_GPL(name);    要导出的模块只能被GPL许可证下的模块使用

# 头文件
    #include <linux/module.h>       包含了大量加载模块需要的函数和符号的定义
    #include <linux/init.h>         指定你的初始化和清理函数
    #include <linux/moudleparam.h>  在模块加载时传递参数给模块

# 初始化和关闭

    static int __init initialization_function(void)
    {
    /* Initialization code here */
    }
    module_init(initialization_function);    

    __init 给内核的暗示, 给定的函数只是在初始化使用. 模块加载者在模块加载后会丢掉这个初始化函数,
    使它的内存可做其他用途. 类似的标签(__initdata) 给只在初始化时用的数据.

    static void __exit cleanup_function(void)
    {
    /* Cleanup code here */
    }
    module_exit(cleanup_function);

    __exit 修饰符标识这个代码是只用于模块卸载

    其中初始化函数的类型位static int，而销毁函数则为static void

# 模块的加载
linux设备驱动有两种加载方式insmod和modprobe，下面谈谈它们用法上的区别

    1.insmod一次只能加载特定的一个设备驱动，且需要驱动的具体地址。写法为：
        insmod drv.ko

    2.modprobe则可以一次将有依赖关系的驱动全部加载到内核。不加驱动的具体地址，但需要在安装文件系统时是按
    照make modues_install的方式安装驱动模块的。写法为：
           modprob drv

    在写好hello.c和Makefile后，生成hello.ko，然后拷贝到/lib/modules/$(uname -r)/目录下
    执行depmod，会在/lib/modules/#uname -r#/目录下生成modules.dep和modules.dep.bb文件，表明模块的依赖关系
    modprobe hello（注意这里无需输入.ko后缀）

    modprobe 和insmod一样都是用来加载内核module的,不过modprobe比较智能，它可以根据module的依赖性来自动为你加载；而insmod就做不到这点。参数的值可由 insmod 或者 modprobe 在加载时指定; 后者也可以从它的配置文件(/etc/modprobe.conf)读取参数的值.

# 错误处理
    错误恢复有时用 goto 语句处理是最好的.

    int __init my_init_function(void)
    {
        int err;
        err = register_this(ptr1, "skull"); /* registration takes a pointer and a name */
        if (err)
          goto fail_this;

        err = register_that(ptr2, "skull");
        if (err)
        goto fail_that;

        err = register_those(ptr3, "skull");
        if (err)         
        goto fail_those;

        return 0;          /* success */

        fail_those:
          unregister_that(ptr2, "skull");
        fail_that:
          unregister_this(ptr1, "skull");
        fail_this:
          return err;         /* propagate the error */
    }

    my_init_function 的返回值, err, 是一个错误码. 在 Linux 内核里, 错误码是负数, 属
    于定义于 <linux/errno.h> 的集合.

    为了有效的降低清除代码的重复性，采用如下方法：
    struct something *item1;
    struct somethingelse *item2;
    int stuff_ok;
    void my_cleanup(void)
    {
        if (item1)
        release_thing(item1);
        if (item2)
        release_thing2(item2);
        if (stuff_ok)
        unregister_stuff();
        return;
    }

    int __init my_init(void)
    {
        int err = -ENOMEM;

        item1 = allocate_thing(arguments);
        item2 = allocate_thing2(arguments2);

        if (!item2 || !item2)
        goto fail;

        err = register_stuff(item1, item2);
        if (!err)
          stuff_ok = 1;
        else
          goto fail;

        return 0; /* success */

        fail:
        my_cleanup();

        return err;
    }
    清理函数当由非退出代码调用时不能标志为 __exit

# 模块参数

    模块参数类似于linux中的命令行参数，在load-time时被赋予

    主要有两种传递参数的方法：
    1.使用insmod和modprobe时
      insmod module.ko [optional parameter, para1=value]
      modprobe也类似

      该方法接受很多类型的参数
      为了使module parameter是可用的，我们会用到module_param宏（定义在moduleparam.h）

      module_param会用到三个参数：
      1）变量名
      2）类型
      3）Permission mast （定义linux/stat.h）
      #define S_IRUSR    00400 文件所有者可读
      #define S_IWUSR    00200 文件所有者可写
      #define S_IXUSR    00100 文件所有者可执行
      #define S_IRGRP    00040 与文件所有者同组的用户可读
      #define S_IWGRP    00020
      #define S_IXGRP    00010
      #define S_IROTH    00004 与文件所有者不同组的用户可读
      #define S_IWOTH    00002
      #define S_IXOTH    00001

      使用 S_IRUGO 作为参数可以被所有人读取, 但是不能改变; S_IRUGO|S_IWUSR 允许 root 来
      改变参数. 注意,如果一个参数被 sysfs 修改, 你的模块看到的参数值也改变了, 但是你的模块没
      有任何其他的通知. 你应当不要使模块参数可写, 除非你准备好检测这个改变并且因而作出反应.

      Module_param_array(name,type,num,perm)
      name既是外部模块的参数名又是程序内部的变量名，type是数据类型，perm是sysfs的访问权限。
      num:参数个数; 这个变量其实无决定性作用;只要name数组大小够大,在插入模块的时候,输入的
      参数个数会改变num的值,最终传递数组元素个数存在num中.

      详见./demo/Module_parameter_demo
      命令行输入：
      insmod hello.ko howmany=5 whom="ckj" myIntArr=10,12,13
      结果输出：
      [168098.684448] hello ckj for 0 time
      [168098.684452] hello ckj for 1 time
      [168098.684454] hello ckj for 2 time
      [168098.684456] hello ckj for 3 time
      [168098.684457] hello ckj for 4 time
      [168098.684459] array[0] = 10
      [168098.684461] array[1] = 12
      [168098.684462] array[2] = 13
      [168098.684464] array[3] = 0
      [168098.684465] array[4] = 0
      [168098.684466] the numbers of array is 3

    2.从配置文件中读取（/etc/modprobe.conf）
      在接受大量参数时，使用该方法更加有效

    模块参数支持许多类型:
    bool
    invbool   一个布尔型( true 或者 false)值(相关的变量应当是 int 类型). invbool 类型颠
    倒了值, 所以真值变成 false, 反之亦然.
    charp     一个字符指针值. 内存为用户提供的字串分配, 指针因此设置.
    int
    long
    short
    uint
    ulong
    ushort
