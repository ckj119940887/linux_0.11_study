# Scull设备中使用到的函数

## open 方法

    open 方法提供给驱动来做任何的初始化来准备后续的操作. 在大部分驱动中, open 应当进行下面
    的工作:

    检查设备特定的错误(例如设备没准备好, 或者类似的硬件错误
    如果它第一次打开, 初始化设备
    如果需要, 更新 f_op 指针.
    分配并填充要放进 filp->private_data 的任何数据结构

    int (*open)(struct inode *inode, struct file *filp);

    inode 参数有我们需要的信息,以它的 i_cdev 成员的形式, 里面包含我们之前建立的cdev 结构.
    唯一的问题是通常我们不想要 cdev 结构本身, 我们需要的是包含 cdev 结构的 scull_dev 结构.

    以 container_of 宏的形式实现了上述功能, 在 <linux/kernel.h> 中定义:
    container_of(pointer, container_type, container_field);
    该宏的本质就是通过container_field的地址（即pointer的值）来计算出其所在的结构体
    container_type的首地址

    struct scull_dev *dev; /* device information */
    dev = container_of(inode->i_cdev, struct scull_dev, cdev);
    filp->private_data = dev; /* for other methods */

    int scull_open(struct inode *inode, struct file *filp)
    {
        struct scull_dev *dev; /* device information */
        dev = container_of(inode->i_cdev, struct scull_dev, cdev);
        filp->private_data = dev; /* for other methods */
        /* now trim to 0 the length of the device if open was write-only */
        if ( (filp->f_flags & O_ACCMODE) == O_WRONLY)
        {
        scull_trim(dev); /* ignore errors */
        }
        return 0; /* success */
    }

## release 方法

    release 方法的角色是 open 的反面. 有时你会发现方法的实现称为 device_close, 而不
    是 device_release.任一方式, 设备方法应当进行下面的任务:

    释放 open 分配在 filp->private_data 中的任何东西
    在最后的 close 关闭设备

    scull 的基本形式没有硬件去关闭, 因此需要的代码是最少的:
    int scull_release(struct inode *inode, struct file *filp)
    {
        return 0;
    }

    内核维持一个文件结构被使用多少次的计数.fork 和 dup 都不创建新文件(只有 open 这样);
    它们只递增正存在的结构中的计数.close 系统调用仅在文件结构计数掉到 0 时执行 release 方法,
    这在结构被销毁时发生

    flush 方法在每次应用程序调用 close 时都被调用. 但是, 很少驱动实现 flush,因为常常在
    close 时没有什么要做, 除非调用 release.

## 内存使用

    scull 使用的内存区, 也称为一个设备, 长度可变. 你写的越多, 它增长越多; 通过使用
    一个短文件覆盖设备来进行修整.

    scull 驱动引入 2 个核心函数来管理 Linux 内核中的内存. 这些函数, 定义在
    <linux/slab.h>, 是:
    void *kmalloc(size_t size, int flags);
    void kfree(void *ptr);

    kmalloc 的调用试图分配 size 字节的内存; 返回值是指向那个内存的指针或者如果分配失败为
    NULL. flags 参数用来描述内存应当如何分配;分配的内存应当用 kfree 来释放. 你应当从
    不传递任何不是从 kmalloc 获得的东西给 kfree. 但是, 传递一个 NULL 指针给 kfree
    是合法的.

    sucll_trim 函数负责释放整个数据区：

    int scull_trim(struct scull_dev *dev)
    {
        struct scull_qset *next, *dptr;
        int qset = dev->qset; /* "dev" is not-null */
        int i;

        for (dptr = dev->data; dptr; dptr = next)
        { /* all the list items */
            if (dptr->data) {
                for (i = 0; i < qset; i++)
                kfree(dptr->data[i]);
                kfree(dptr->data);
                dptr->data = NULL;
            }
            next = dptr->next;
            kfree(dptr);
        }

        dev->size = 0;
        dev->quantum = scull_quantum;
        dev->qset = scull_qset;
        dev->data = NULL;

        return 0;
    }

## 读和写

    与应用空间进行数据交互
    ssize_t read(struct file *filp, char __user *buff, size_t count, loff_t *offp);
    ssize_t write(struct file *filp, const char __user *buff, size_t count, loff_t *offp);
    对于 2 个方法, filp 是文件指针, count 是请求的传输数据大小. buff 参数指向持有被
    写入数据的缓存, 或者放入新数据的空缓存. 最后, offp 是一个指针指向一个"long
    offset type"对象, 它指出用户正在存取的文件位置. 返回值是一个"signed size type";

    read 和 write 方法的 buff 参数是用户空间指针. 因此, 它不能被内核代码直接解引用.
    scull 中的读写代码需要拷贝一整段数据到或者从用户地址空间. 这个能力由下列内核函
    数提供, 它们拷贝一个任意的字节数组, 并且位于大部分读写实现的核心中.
    unsigned long copy_to_user(void __user *to,const void *from,unsigned long count);
    unsigned long copy_from_user(void *to,const void __user *from,unsigned long count);

    必须加一点小心在从内核代码中存取用户空间.寻址的用户也当前可能不在内存, 虚拟内存子系统会使
    进程睡眠在这个页被传送到位时.例如, 这发生在必须从交换空间获取页的时候. 对于驱动编写者来说,
    最终结果是任何存取用户空间的函数必须是可重入的, 必须能够和其他驱动函数并行执行, 并且,
    特别的,必须在一个它能够合法地睡眠的位置.

    read 方法的任务是从设备拷贝数据到用户空间(使用copy_to_user), 而 write 方法必须从用户
    空间拷贝数据到设备(使用 copy_from_user).read 和 write 方法都在发生错误时返回一个负值.
    相反, 大于或等于 0 的返回值告知调用程序有多少字节已经成功传送. 如果一些数据成功传送接着发
    生错误, 返回值必须是成功传送的字节数, 错误不报告直到函数下一次调用.

### 读

    read 的返回值由调用的应用程序解释:

    1）如果这个值等于传递给 read 系统调用的 count 参数, 请求的字节数已经被传送.这是最好的情况.
    2）如果是正数, 但是小于 count, 只有部分数据被传送. 这可能由于几个原因, 依赖于设备. 常常,
    应用程序重新试着读取. 例如, 如果你使用 fread 函数来读取, 库函数重新发出系统调用直到请求
    的数据传送完成.
    3）如果值为 0, 到达了文件末尾(没有读取数据).
    4）一个负值表示有一个错误. 这个值指出了什么错误, 根据 <linux/errno.h>. 出错的典型返回值
    包括 -EINTR( 被打断的系统调用) 或者 -EFAULT( 坏地址 ).

    还有一种特殊情况："没有数据, 但是可能后来到达". 在这种情况下, read 系统调用应当阻塞.

    每个 scull_read 调用只处理单个数据量子, 不实现一个循环来收集所有的数据

    如果当前读取位置大于设备大小, scull 的 read 方法返回 0 来表示没有可用的数据(换句话说,
    我们在文件尾). 这个情况发生在如果进程 A 在读设备, 同时进程 B 打开它写,这样将设备截短为 0.
    进程 A 突然发现自己过了文件尾, 下一个读调用返回 0.

    ssize_t scull_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
    {
        struct scull_dev *dev = filp->private_data;
        struct scull_qset *dptr; /* the first listitem */
        int quantum = dev->quantum, qset = dev->qset;
        int itemsize = quantum * qset; /* how many bytes in the listitem */
        int item, s_pos, q_pos, rest;

        ssize_t retval = 0;

        if (down_interruptible(&dev->sem))
          return -ERESTARTSYS;

        if (*f_pos >= dev->size)
          goto out;

        if (*f_pos + count > dev->size)
          count = dev->size - *f_pos;

        /* find listitem, qset index, and offset in the quantum */
        item = (long)*f_pos / itemsize;
        rest = (long)*f_pos % itemsize;
        s_pos = rest / quantum;
        q_pos = rest % quantum;

        /* follow the list up to the right position (defined elsewhere) */
        dptr = scull_follow(dev, item);
        if (dptr == NULL || !dptr->data || ! dptr->data[s_pos])
        goto out; /* don't fill holes */

        /* read only up to the end of this quantum */
        if (count > quantum - q_pos)
          count = quantum - q_pos;

        if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count))
        {
          retval = -EFAULT;
          goto out;
        }

        *f_pos += count;
        retval = count;

      out:
        up(&dev->sem);
        return retval;
    }

### 写

    返回值：
    1）如果值等于 count, 要求的字节数已被传送.
    2）如果正值, 但是小于 count, 只有部分数据被传送. 程序最可能重试写入剩下的数据.
    3）如果值为 0, 什么没有写. 这个结果不是一个错误, 没有理由返回一个错误码. 再一次, 标准库
    重试写调用. 我们将在第 6 章查看这种情况的确切含义, 那里介绍了阻塞.
    4）一个负值表示发生一个错误; 如同对于读, 有效的错误值是定义于<linux/errno.h>中.

    ssize_t scull_write(struct file *filp, const char __user *buf, size_t count, loff_t *f_pos)
    {
      struct scull_dev *dev = filp->private_data;
      struct scull_qset *dptr;
      int quantum = dev->quantum, qset = dev->qset;
      int itemsize = quantum * qset;
      int item, s_pos, q_pos, rest;
      ssize_t retval = -ENOMEM; /* value used in "goto out" statements */

      if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

      /* find listitem, qset index and offset in the quantum */
      item = (long)*f_pos / itemsize;
      rest = (long)*f_pos % itemsize;
      s_pos = rest / quantum;
      q_pos = rest % quantum;

      /* follow the list up to the right position */
      dptr = scull_follow(dev, item);

      if (dptr == NULL)
        goto out;

      if (!dptr->data)
      {
        dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
        if (!dptr->data)
          goto out;
          memset(dptr->data, 0, qset * sizeof(char *));
      }

      if (!dptr->data[s_pos])
      {
        dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
        if (!dptr->data[s_pos])
          goto out;
      }

      /* write only up to the end of this quantum */
      if (count > quantum - q_pos)
        count = quantum - q_pos;

      if (copy_from_user(dptr->data[s_pos]+q_pos, buf, count))
      {
        retval = -EFAULT;
        goto out;
      }

      *f_pos += count;
      retval = count;
      /* update the size */
      if (dev->size < *f_pos)
        dev->size = *f_pos;

    out:
      up(&dev->sem);
      return retval;
    }

### readv 和 writev

    read 和 write的"矢量"版本使用一个结构数组, 每个包含一个缓存的指针和一个长度值.
    一个 readv 调用被期望来轮流读取指示的数量到每个缓存. 相反, writev 要收集每个缓存的内容到
    一起并且作为单个写操作送出它们.

    矢量操作的原型是:
    ssize_t (*readv) (struct file *filp, const struct iovec *iov, unsigned long count, loff_t *ppos);
    ssize_t (*writev) (struct file *filp, const struct iovec *iov, unsigned long count, loff_t *ppos);

    这里, filp 和 ppos 参数与 read 和 write 的相同. iovec 结构, 定义于
    <linux/uio.h>, 如同:
    struct iovec
    {
    void __user *iov_base; __kernel_size_t iov_len;
    };

    每个 iovec 描述了一块要传送的数据; 它开始于 iov_base (在用户空间)并且有 iov_len字节长.
    count 参数告诉有多少 iovec 结构.
