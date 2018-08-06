# module parameter

    模块参数类似于linux中的命令行参数，在load-time时被赋予

    主要有两种传递参数的方法：
    1.使用insmod和modprobe时
      insmod module.ko [optional parameter, para1=value]
      modprobe也类似

      该方法接受很多类型的参数
      为了使module parameter是可用的，我们会用到module_param宏（定义在moduleparam.h）

      module_para会用到三个参数：
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
