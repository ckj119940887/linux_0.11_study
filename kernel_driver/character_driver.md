# 设备类型

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

    <linux/kdev_t.h>
    提取主设备号的方法： （dev_t dev）
    提取次设备号的方法：MINOR（dev_t dev）

    <linux/types.h>
    生成dev_t的方法：MKDEV（int major，int minor）

# 分配和释放设备编号

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
    返回成功时为0，错误时为负的错误码。
    from :要分配的设备编号范围的初始值（该设备的次设备号常设为0);
    Count:连续设备编号的个数.
    name:和编号范围相关联的设备名称，将出现在/proc/devices和sysfs中;

    2.动态分配
    int alloc_chrdev_region(dev_t *dev,unsigned int firstminor,unsigned int count,char *name);
    firstminor是请求的最小的次设备号；
    count是连续设备编号的个数；
    name为设备名，返回值小于0表示分配失败。
    然后通过major=MMOR(dev)获取主设备号

    3.释放
    Void unregist_chrdev_region(dev_t first,unsigned int count);
    调用Documentation/devices.txt中能够找到已分配的设备号．

    对于新驱动, 我们强烈建议你使用动态分配来获取你的主设备编号, 而不是随机选取一个当前空闲的编
    号. 换句话说, 你的驱动应当几乎肯定地使用 alloc_chrdev_region,不是
    register_chrdev_region. 动态分配的缺点是你无法提前创建设备节点, 因为分配给你的模块的主
    编号会变化. 对于驱动的正常使用, 这不是问题, 因为一旦编号分配了, 你可从 /proc/devices
    中读取它.

    为使用动态主编号来加载一个驱动, 因此, 可使用一个简单的脚本来代替调用 insmod, 在调用
    insmod 后, 读取 /proc/devices 来创建特殊文件.

    安排主编号最好的方式, 我们认为, 是缺省使用动态分配, 而留给自己在加载时指定主编号的选项权,
    或者甚至在编译时. scull 实现以这种方式工作; 它使用一个全局变量,scull_major, 来持有选定
    的编号(还有一个 scull_minor 给次编号). 这个变量初始化为SCULL_MAJOR, 定义在 scull.h.
    发布的源码中的 SCULL_MAJOR 的缺省值是 0, 意思是"使用动态分配". 用户可以接受缺省值或者
    选择一个特殊主编号, 或者在编译前修改宏定义或者在 insmod 命令行指定一个值给 scull_major.
    最后, 通过使用 scull_load 脚本, 用户可以在 scull_load 的命令行传递参数给 insmod。

# 字符设备

![avatar](./images/character_driver_model.jpeg)

    在Linux内核中，使用cdev结构体来描述一个字符设备，cdev结构体的定义如下：

    <include/linux/cdev.h>

    struct cdev {
    	struct kobject kobj;                  //内嵌的内核对象.
    	struct module *owner;                 //该字符设备所在的内核模块的对象指针.
    	const struct file_operations *ops;    //该结构描述了字符设备所能实现的方法，是极为关键的一个结构体.
    	struct list_head list;                //用来将已经向内核注册的所有字符设备形成链表.
    	dev_t dev;                            //字符设备的设备号，由主设备号和次设备号构成.
    	unsigned int count;                   //隶属于同一主设备号的次设备号的个数.
    };

    内核给出的操作struct cdev结构的接口主要有以下几个:

    对该结构体进行初始化主要使用函数1和2,这里存在了两个方法：
    1）获取一个独立的struct cdev结构
      struct cdev *my_cdev = cdev_alloc();
      my_cdev->ops = &my_fops;
    2）将cdev嵌入到自定义的结构体中
      scull 这样做了. 在这种情况下, 你应当初始化你已经分配的结构, 使用:
      void cdev_init(struct cdev *cdev, struct file_operations *fops);

    无论上述哪种方法，有一个其他的 struct cdev 成员你需要初始化. 象file_operations 结构,
    struct cdev 中断拥有者成员, 应当设置为 THIS_MODULE.

    1. void cdev_init(struct cdev *, const struct file_operations *);
    该函数主要对struct cdev结构体做初始化，最重要的就是建立cdev 和 file_operations之间的连接：

    源码为：
    void cdev_init(struct cdev *cdev, const struct file_operations *fops)
    {
    	memset(cdev, 0, sizeof *cdev);
    	INIT_LIST_HEAD(&cdev->list);
    	kobject_init(&cdev->kobj, &ktype_cdev_default);
    	cdev->ops = fops;
    }

    (1) 将整个结构体清零；
    (2) 初始化list成员使其指向自身；
    (3) 初始化kobj成员；
    (4) 初始化ops成员；


    2. struct cdev *cdev_alloc(void);
    该函数主要分配一个struct cdev结构，动态申请一个cdev内存，并做了cdev_init中所做的前
    3步初始化工作(第四步初始化工作需要在调用cdev_alloc后，显式的做初始化即: .ops=xxx_ops).

     源码为：
     struct cdev *cdev_alloc(void)
     {
      	struct cdev *p = kzalloc(sizeof(struct cdev), GFP_KERNEL);
      	if (p) {
      		INIT_LIST_HEAD(&p->list);
      		kobject_init(&p->kobj, &ktype_cdev_dynamic);
      	}
      	return p;
      }

    3. int cdev_add(struct cdev *p, dev_t dev, unsigned count);
    该函数向内核注册一个struct cdev结构，即正式通知内核由struct cdev *p代表的字符设备已经可以使用了。

    在使用 cdev_add 是有几个重要事情要记住. 第一个是这个调用可能失败. 如果它返回一
    个负的错误码, 你的设备没有增加到系统中. 它几乎会一直成功, 但是, 并且带起了其他
    的点: cdev_add 一返回, 你的设备就是"活的"并且内核可以调用它的操作. 除非你的驱动
    完全准备好处理设备上的操作, 你不应当调用 cdev_add.

    当然这里还需提供两个参数：
    (1)第一个设备号 dev，
    (2)和该设备关联的设备编号的数量。
    这两个参数直接赋值给struct cdev 的dev成员和count成员。

    4. void cdev_del(struct cdev *p)；
    该函数向内核注销一个struct cdev结构，即正式通知内核由struct cdev *p代表的字符设备已经不可以使用了。
    从上述的接口讨论中，我们发现对于struct cdev的初始化和注册的过程中，我们需要提供几个东西

    (1) struct file_operations结构指针；
    (2) dev设备号；
    (3) count次设备号个数。

    5. Demo on SCULL devices
    struct scull_dev {
      struct scull_qset *data; /* Pointer to first quantum set */
      int quantum; /* the current quantum size */
      int qset; /* the current array size */
      unsigned long size; /* amount of data stored here */
      unsigned int access_key; /* used by sculluid and scullpriv */
      struct semaphore sem; /* mutual exclusion semaphore */
      struct cdev cdev; /* Char device structure */
    };

    static void scull_setup_cdev(struct scull_dev *dev, int index)
    {
        int err, devno = MKDEV(scull_major, scull_minor + index);
        cdev_init(&dev->cdev, &scull_fops);
        dev->cdev.owner = THIS_MODULE;
        dev->cdev.ops = &scull_fops;
        err = cdev_add (&dev->cdev, devno, 1);
        /* Fail gracefully if need be */
        if (err)
        printk(KERN_NOTICE "Error %d adding scull%d", err, index);
    }

    6. 2.6内核以前的老办法
    该方法尽量不要使用，因为内核马上会将其淘汰

    注册一个字符设备的经典方法是使用:
    int register_chrdev(unsigned int major, const char *name, struct
    file_operations *fops);
    major 是感兴趣的主编号, name 是驱动的名子(出现在 /proc/devices), fops 是
    缺省的 file_operations 结构. 一个对 register_chrdev 的调用为给定的主编号注册
    0 - 255 的次设备号, 并且为每一个建立一个缺省的 cdev 结构. 使用这个接口的驱动必须准
    备好处理对所有 256 个次编号的 open 调用

    如果你使用 register_chrdev, 从系统中去除你的设备的正确的函数是:
    int unregister_chrdev(unsigned int major, const char *name);
    major 和 name 必须和传递给 register_chrdev 的相同, 否则调用会失败

# 重要数据结构

## 文件操作 （file_operations）

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

    release等到所有的副本都关闭后才会得到调用，而flush在关闭任意一个副本时都可以用来刷新数据。

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

## file结构

    struct file是一个内核结构, 从不出现在用户程序中,定义于 <linux/fs.h>.
    文件结构代表一个打开的文件. (它不特定给设备驱动; 系统中每个打开的文件有一个关联的
    struct file 在内核空间). 它由内核在 open 时创建, 并传递给在文件上操作的任何函数, 直到
    最后的关闭. 在文件的所有实例都关闭后, 内核释放这个数据结构.

    本文中，file 指的是结构, 而 filp 是结构指针.

    mode_t f_mode;
    文件模式确定文件是可读的或者是可写的(或者都是), 通过位 FMODE_READ 和
    FMODE_WRITE. 你可能想在你的 open 或者 ioctl 函数中检查这个成员的读写许可,
    但是你不需要检查读写许可, 因为内核在调用你的方法之前检查. 当文件还没有为
    那种存取而打开时读或写的企图被拒绝, 驱动甚至不知道这个情况.

    loff_t f_pos;
    当前读写位置. loff_t 在所有平台都是 64 位( 在 gcc 术语里是 long long ).
    驱动可以读这个值, 如果它需要知道文件中的当前位置, 但是正常地不应该改变它;
    读和写应当使用它们作为最后参数而收到的指针来更新一个位置, 代替直接作用于
    filp->f_pos. 这个规则的一个例外是在 llseek 方法中, 它的目的就是改变文件位
    置.

    unsigned int f_flags;
    这些是文件标志, 例如 O_RDONLY, O_NONBLOCK, 和 O_SYNC. 驱动应当检查
    O_NONBLOCK 标志来看是否是请求非阻塞操作( 我们在第一章的"阻塞和非阻塞操作"
    一节中讨论非阻塞 I/O ); 其他标志很少使用. 特别地, 应当检查读/写许可, 使用
    f_mode 而不是 f_flags. 所有的标志在头文件 <linux/fcntl.h> 中定义.

    struct file_operations *f_op;
    和文件关联的操作. 内核安排指针作为它的 open 实现的一部分, 接着读取它当它
    需要分派任何的操作时. filp->f_op 中的值从不由内核保存为后面的引用; 这意味
    着你可改变你的文件关联的文件操作, 在你返回调用者之后新方法会起作用. 例如,
    关联到主编号 1 (/dev/null, /dev/zero, 等等)的 open 代码根据打开的次编号来
    替代 filp->f_op 中的操作. 这个做法允许实现几种行为, 在同一个主编号下而不
    必在每个系统调用中引入开销. 替换文件操作的能力是面向对象编程的"方法重载"
    的内核对等体.

    void *private_data;
    open 系统调用设置这个指针为 NULL, 在为驱动调用 open 方法之前. 你可自由使
    用这个成员或者忽略它; 你可以使用这个成员来指向分配的数据, 但是接着你必须
    记住在内核销毁文件结构之前, 在 release 方法中释放那个内存. private_data
    是一个有用的资源, 在系统调用间保留状态信息, 我们大部分例子模块都使用它.

    struct dentry *f_dentry;
    关联到文件的目录入口( dentry )结构. 设备驱动编写者正常地不需要关心 dentry
    结构, 除了作为 filp->f_dentry->d_inode 存取 inode 结构.

## inode结构

    inode 结构由内核在内部用来表示文件. 因此, 它和代表打开文件描述符的文件结构是不
    同的. 可能有代表单个文件的多个打开描述符的许多文件结构, 但是它们都指向一个单个
    inode 结构.

    dev_t i_rdev;
    对于代表设备文件的节点, 这个成员包含实际的设备编号.

    struct cdev *i_cdev;
    struct cdev 是内核的内部结构, 代表字符设备; 这个成员包含一个指针, 指向这
    个结构, 当节点指的是一个字符设备文件时.

    内核开发者已经增加了 2 个宏, 可用来从一个 inode 中获取主次编号:
    unsigned int iminor(struct inode *inode);
    unsigned int imajor(struct inode *inode);
    为了不要被下一次改动抓住, 应当使用这些宏代替直接操作 i_rdev.

# 字符设备的注册

    内核在内部使用类型 struct cdev 的结构来代表字符设备，其定义在<linux/cdev.h>。
