# 驱动的符号表

    驱动的符号表可以通过查看/proc/kallsyms获得，该文件中记录了所有驱动中export的符号（主要
    针对的是静态的kernel module以及已经动态链接的module）

    使用nm命令可以查看.ko中的所有符号

    System.map是被kernel使用的symbol table，使用locate System.map可以定位该文件的位置
    可以用来获得符号的地址和名称
