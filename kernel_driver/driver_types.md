# Drivers types

    所有的设备文件都存放在/dev中，所有的设备都被当做文件来处理

    driver主要分为三类：
    1.字符设备（character devices）
    主要用来传输数据的设备。指只能一个字节一个字节读写的设备，不能随机读取设备内存中的某一数据，
    读取数据需要按照先后数据。字符设备是面向流的设备，常见的字符设备有鼠标、键盘、串口、控制台
    和LED设备等。

    在一个字符设备和一个普通文件之间唯一有关的不同就是, 你经常可以在普通文件中移来移去, 但是大
    部分字符设备仅仅是数据通道, 你只能顺序存取.然而, 存在看起来象数据区的字符设备, 你可以在里
    面移来移去.

    2.块设备（block devices）
    主要用来存储数据的设备。是指可以从设备的任意位置读取一定长度数据的设备。

    Linux, 相反, 允许应用程序读写一个块设备象一个字符设备一样 -- 它允许一次传送任意数目的字节.
    结果就是, 块和字符设备的区别仅仅在内核以及在内部管理数据的方式上, 并且因此在内核/驱动的软件
    接口上不同. 如同一个字符设备, 每个块设备都通过一个文件系统结点被存取的, 它们之间的区别对用
    户是透明的. 块驱动和字符驱动相比, 与内核的接口完全不同.

    3.网络设备（network devices）
    不是一个面向流的设备, 一个网络接口就不象 /dev/tty1 那么容易映射到文件系统的一个结点上.不
    向字符设备和块设备可以直接通过/dev下的节点进行操作。

    内核与网络设备驱动间的通讯与字符和块设备驱动所用的完全不同. 不用 于read 和 write, 而是由
    内核调用和报文传递相关的函数.

# 主设备号与次设备号

    查看主设备号与次设备号: ls -al /dev
    查看当前已加载的设备驱动程序的主设备号: cat /proc/devices

    每个字符设备和块设备都必须有主次设备号，主设备号相同的设备是同类设备（使用同一驱动程序）

    /*设备号，dev_t实质是一个32位整，高12位为主设备号，低20位为次设备号*/
    dev_t dev;

    提取主设备号的方法：MAJOR（dev_t dev）
    提取次设备号的方法：MINOR（dev_t dev）
    生成dev_t的方法：MKDEV（int major，int minor）

# driver开发

    内核提供了三个函数来注册一组字符设备编号，这三个函数分别是 register_chrdev_region()
    、alloc_chrdev_region() 和register_chrdev()。这三个函数都会调用一个共用的
    __register_chrdev_region() 函数来注册一组设备编号范围（即一个 char_device_struct 结构）。

    static struct char_device_struct {
    	struct char_device_struct *next;
    	unsigned int major;
    	unsigned int baseminor;
    	int minorct;
    	char name[64];
    	struct cdev *cdev;		/* will die */
    } *chrdevs[CHRDEV_MAJOR_HASH_SIZE];

    1.随机分配
    int register_chrdev_region(dev_t from, unsigned count, const char *name)
    from :要分配的设备编号范围的初始值(次设备号常设为0);
    Count:连续编号范围.
    name:编号相关联的设备名称. (/proc/devices);

    2.动态分配
    int alloc_chrdev_region(dev_t *dev,unsigned int firstminor,unsigned int count,char *name);
    firstminor是请求的最小的次编号；
    count是请求的连续设备编号的总数；
    name为设备名，返回值小于0表示分配失败。
    然后通过major=MMOR(dev)获取主设备号

    3.释放
    Void unregist_chrdev_region(dev_t first,unsigned int count);
    调用Documentation/devices.txt中能够找到已分配的设备号．

# file_operations

    Linux使用file_operations结构访问驱动程序的函数，这个结构的每一个成员的名字都对应着一个
    调用。用户进程利用在对设备文件进行诸如read/write操作的时候，系统调用通过设备文件的主设备
    号找到相应的设备驱动程序，然后读取这个数据结构相应的函数指针，接着把控制权交给该函数，这是
    Linux的设备驱动程序工作的基本原理。

    <linux/fs.h>
    struct file_operations {

        //拥有该结构的模块的指针，一般为THIS_MODULES
        struct module *owner;

        //用来修改文件当前的读写位置
        loff_t (*llseek) (struct file *, loff_t, int);

        //从设备中同步读取数据
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);

        //向设备发送数据
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);

        //初始化一个异步的读取操作
        ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);

        //初始化一个异步的写入操作
        ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);

        //仅用于读取目录，对于设备文件，该字段为NULL
        int (*readdir) (struct file *, void *, filldir_t);

        //轮询函数，判断目前是否可以进行非阻塞的读写或写入
        unsigned int (*poll) (struct file *, struct poll_table_struct *);

        //执行设备I/O控制命令
        int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);

        //不使用BLK文件系统，将使用此种函数指针代替ioctl
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);

        //在64位系统上，32位的ioctl调用将使用此函数指针代替
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);  

        //用于请求将设备内存映射到进程地址空间
        int (*mmap) (struct file *, struct vm_area_struct *);

        //打开
        int (*open) (struct inode *, struct file *);

        int (*flush) (struct file *, fl_owner_t id);

        //关闭
        int (*release) (struct inode *, struct file *);

        //刷新待处理的数据
        int (*fsync) (struct file *, struct dentry *, int datasync);

        //异步刷新待处理的数据
        int (*aio_fsync) (struct kiocb *, int datasync);  

        //通知设备FASYNC标志发生变化
        int (*fasync) (int, struct file *, int);  

        int (*lock) (struct file *, int, struct file_lock *);

        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);

        unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);

        int (*check_flags)(int);

        int (*flock) (struct file *, int, struct file_lock *);

        ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);

        ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);

        int (*setlease)(struct file *, long, struct file_lock **);

    };

    下面是各成员解析：

    1、struct module *owner
    第一个 file_operations 成员根本不是一个操作，它是一个指向拥有这个结构的模块的指针。

    这个成员用来在它的操作还在被使用时阻止模块被卸载. 几乎所有时间中, 它被简单初始化为
    THIS_MODULE, 一个在 <linux/module.h> 中定义的宏.这个宏比较复杂，在进行简单学习操作
    的时候，一般初始化为THIS_MODULE。

    2、loff_t (*llseek) (struct file * filp , loff_t p, int orig);
    (指针参数filp为进行读取信息的目标文件结构体指针；参数 p 为文件定位的目标偏移量；参数
    orig为对文件定位的起始地址，这个值可以为文件开头（SEEK_SET，0,当前位置(SEEK_CUR,1)，
    文件末尾(SEEK_END,2)）

    llseek 方法用作改变文件中的当前读/写位置, 并且新位置作为(正的)返回值.

    loff_t 参数是一个"long offset", 并且就算在 32位平台上也至少 64 位宽. 错误由一个
    负返回值指示；如果这个函数指针是 NULL, seek 调用会以潜在地无法预知的方式修改 file 结构
    中的位置计数器( 在"file 结构" 一节中描述).

    3、ssize_t (*read) (struct file * filp, char __user * buffer, size_t size , loff_t * p);
    (指针参数 filp 为进行读取信息的目标文件，指针参数buffer 为对应放置信息的缓冲区
      （即用户空间内存地址），参数size为要读取的信息长度，参数 p 为读的位置相对于文件开头的
      偏移，在读取信息后，这个指针一般都会移动，移动的值为要读取信息的长度值）

     这个函数用来从设备中获取数据。在这个位置的一个空指针导致 read 系统调用以
     -EINVAL("Invalid argument") 失败。一个非负返回值代表了成功读取的字节数( 返回值是一个
       "signed size" 类型, 常常是目标平台本地的整数类型).

    4、ssize_t (*aio_read)(struct kiocb * , char __user * buffer, size_t size , loff_t   p);
    可以看出，这个函数的第一、三个参数和本结构体中的read()函数的第一、三个参数是不同 的，异步
    读写的第三个参数直接传递值，而同步读写的第三个参数传递的是指针，因为AIO从来不需要改变文件的
    位置。异步读写的第一个参数为指向kiocb结构体的指针，而同步读写的第一参数为指向file结构体的
    指针，每一个I/O请求都对应一个kiocb结构体);初始化一个异步读 --
    可能在函数返回前不结束的读操作.如果这个方法是 NULL, 所有的操作会由 read 代替进行(同步地)

    5、ssize_t (*write) (struct file * filp, const char __user *   buffer, size_t count, loff_t * ppos);
    (参数filp为目标文件结构体指针，buffer为要写入文件的信息缓冲区，count为要写入信息的长度，
    ppos为当前的偏移位置，这个值通常是用来判断写文件是否越界）

    发送数据给设备。如果 NULL, -EINVAL 返回给调用 write 系统调用的程序. 如果非负,
    返回值代表成功写的字节数。

    (注：这个操作和上面的对文件进行读的操作均为阻塞操作）

    6、ssize_t (*aio_write)(struct kiocb *, const char __user * buffer, size_t count, loff_t * ppos);
    初始化设备上的一个异步写.参数类型同aio_read()函数;

    7、int (*readdir) (struct file * filp, void *, filldir_t);
    对于设备文件这个成员应当为 NULL; 它用来读取目录, 并且仅对文件系统有用.

    8、unsigned int (*poll) (struct file *, struct poll_table_struct *);
    (这是一个设备驱动中的轮询函数，第一个参数为file结构指针，第二个为轮询表指针）

    这个函数返回设备资源的可获取状态，即POLLIN，POLLOUT，POLLPRI，POLLERR，POLLNVAL等宏
    的位“或”结果。每个宏都表明设备的一种状态，如：POLLIN（定义为0x0001）意味着设备可以无阻塞
    的读，POLLOUT（定义为0x0004）意味着设备可以无阻塞的写。

    (poll 方法是 3 个系统调用的后端: poll, epoll, 和 select, 都用作查询对一个或多个文件
      描述符的读或写是否会阻塞.poll 方法应当返回一个位掩码指示是否非阻塞的读或写是可能的,
      并且, 可能地, 提供给内核信息用来使调用进程睡眠直到 I/O 变为可能. 如果一个驱动的 poll
      方法为 NULL, 设备假定为不阻塞地可读可写.

    (这里通常将设备看作一个文件进行相关的操作，而轮询操作的取值直接关系到设备的响应情况，
      可以是阻塞操作结果，同时也可以是非阻塞操作结果）

    9、int (*ioctl) (struct inode *inode, struct file *filp, unsigned int cmd, unsigned long arg);
    (inode 和 filp 指针是对应应用程序传递的文件描述符 fd 的值, 和传递给 open 方法的相同参数

    cmd 参数从用户那里不改变地传下来, 并且可选的参数 arg 参数以一个 unsigned long 的形式传递,
    不管它是否由用户给定为一个整数或一个指针.如果调用程序不传递第 3 个参数, 被驱动操作收到的
    arg 值是无定义的.因为类型检查在这个额外参数上被关闭, 编译器不能警告你如果一个无效的参数
    被传递给 ioctl, 并且任何关联的错误将难以查找.）

    ioctl 系统调用提供了发出设备特定命令的方法(例如格式化软盘的一个磁道, 这不是读也不是写).
    另外, 几个 ioctl 命令被内核识别而不必引用 fops 表.如果设备不提供 ioctl 方法, 对于任何
    未事先定义的请求(-ENOTTY, "设备无这样的 ioctl"), 系统调用返回一个错误.

    10、int (*mmap) (struct file *, struct vm_area_struct *);
    mmap 用来请求将设备内存映射到进程的地址空间。 如果这个方法是 NULL, mmap 系统调用返回 -ENODEV.
    (如果想对这个函数有个彻底的了解，那么请看有关“进程地址空间”介绍的书籍)

    11、int (*open) (struct inode * inode , struct file * filp ) ;
    (inode 为文件节点,这个节点只有一个，无论用户打开多少个文件，都只是对应着一个inode结构；
    但是filp就不同，只要打开一个文件，就对应着一个file结构体，file结构体通常用来追踪文件在运行时的状态信息）
    尽管这常常是对设备文件进行的第一个操作, 不要求驱动声明一个对应的方法. 如果这个项是 NULL,
    设备打开一直成功, 但是你的驱动不会得到通知.与open()函数对应的是release()函数。

    12、int (*flush) (struct file *);
    flush 操作在进程关闭它的设备文件描述符的拷贝时调用;
    它应当执行(并且等待)设备的任何未完成的操作.这个必须不要和用户查询请求的 fsync 操作混淆了.
    当前, flush 在很少驱动中使用;SCSI 磁带驱动使用它,
    例如, 为确保所有写的数据在设备关闭前写到磁带上. 如果 flush 为 NULL, 内核简单地忽略用户应用程序的请求.

    13、int (*release) (struct inode *, struct file *);
    release ()函数当最后一个打开设备的用户进程执行close()系统调用的时候，内核将调用驱动程序release()函数：
    void release(struct inode inode,struct file *file),release函数的主要任务是清理
    未结束的输入输出操作，释放资源，用户自定义排他标志的复位等。在文件结构被释放时引用这个操作.
    如同 open, release 可以为 NULL.

    14、int(*synch)(struct file *,struct dentry *,int datasync);
     刷新待处理的数据,允许进程把所有的脏缓冲区刷新到磁盘。

    15、int (*aio_fsync)(struct kiocb *, int);
    这是 fsync 方法的异步版本.所谓的fsync方法是一个系统调用函数。系统调用fsync把文件所指定
    的文件的所有脏缓冲区写到磁盘中（如果需要，还包括存有索引节点的缓冲区）。
    相应的服务例程获得文件对象的地址，并随后调用fsync方法。
    通常这个方法以调用函数__writeback_single_inode()结束，这个函数把与被选中的索引节点相关
    的脏页和索引节点本身都写回磁盘

    16、int (*fasync) (int, struct file *, int);
    这个函数是系统支持异步通知的设备驱动，下面是这个函数的模板：
    static int ***_fasync(int fd,struct file *filp,int mode)
    {
      struct ***_dev * dev=filp->private_data;

      //第四个参数为 fasync_struct结构体指针的指针。
      return fasync_helper(fd,filp,mode,&dev->async_queue);
      //这个函数是用来处理FASYNC标志的函数。
      （FASYNC：表示兼容BSD的fcntl同步操作）当这个标志改变时，驱动程序中的fasync（）函数将得到执行。  
      （注：感觉这个‘标志'词用的并不恰当）
    }

    17、int (*lock) (struct file *, int, struct file_lock *);
    lock 方法用来实现文件加锁; 加锁对常规文件是必不可少的特性, 但是设备驱动几乎从不实现它.

    18、ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
    ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
    这些方法实现发散/汇聚读和写操作. 应用程序偶尔需要做一个包含多个内存区的单个读或写操作;
    这些系统调用允许它们这样做而不必对数据进行额外拷贝.
    如果这些函数指针为 NULL, read 和 write 方法被调用( 可能多于一次 ).

    19、ssize_t (*sendfile)(struct file *, loff_t *, size_t, read_actor_t, void *);
    这个方法实现 sendfile 系统调用的读, 使用最少的拷贝从一个文件描述符搬移数据到另一个.
    例如, 它被一个需要发送文件内容到一个网络连接的 web 服务器使用. 设备驱动常常使 sendfile 为 NULL.

    20、ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    sendpage 是 sendfile 的另一半; 它由内核调用来发送数据, 一次一页, 到对应的文件. 设备驱动实际上不实现 sendpage.

    21、unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    这个方法的目的是在进程的地址空间找一个合适的位置来映射在底层设备上的内存段中。这个任务通常由内存管理代码进行; 这个方法存在为了使驱动能强制特殊设备可能有的任何的对齐请求. 大部分驱动可以置这个方法为 NULL.[10]

    22、int (*check_flags)(int)
    这个方法允许模块检查传递给 fnctl(F_SETFL...) 调用的标志.

    23、int (*dir_notify)(struct file *, unsigned long);
    这个方法在应用程序使用 fcntl 来请求目录改变通知时调用. 只对文件系统有用; 驱动不需要实现 dir_notify.
