### sysfs文件系统

​	**sysfs是一个特殊文件系统，并没有一个实际存放文件的介质。sysfs的信息来源是kobject层次结构，读一个sysfs文件，就是动态的从kobject结构提取信息，生成文件。**

​	**kobject 是Linux 2.6引入的新的设备管理机制，在内核中由struct kobject表示。通过这个数据结构使所有设备在底层都具有统一的接口，kobject提供基本的对象管理，是构成Linux2.6设备模型的核心结构，Kobject是组成设备模型的基本结构。类似于C++中的基类，它嵌入于更大的对象的对象中，用来描述设备模型的组件。如bus,devices, drivers 等。都是通过kobject连接起来了，形成了一个树状结构。这个树状结构就与/sys向对应。**

​	**sys文件系统除了在内核空间所展现的合纵连横作用外，而且以文件目录层次结构的形式向用户空间提供了系统硬件设备之间的一个拓扑图，这种文件形式的接口也让用户空间与内核空间的数据对象的交互成为可能。这种数据对象之间的交互的一个具体的例子就是，通过sysfs文件系统可以取代ioctl的功能，通过iocol函数向设备发出命令的过程可以被一个简单的shell命令所取代。**

**sysfs功能概括的说有三点：**

- 建立系统中总线、驱动、设备三者之间的桥梁；
- 向用户空间展示内核中各种设备的拓扑图；
- 提供给用户空间对设备获取信息和操作的接口，部分取代ioctl功能。

#### sys文件系统的初始化

**sys文件系统的初始化发生在Linux系统的启动阶段。**

```c
//fs/sysfs/mount.c
int __init sysfs_init(void)
{
	int err = -ENOMEM;

	sysfs_dir_cachep = kmem_cache_create("sysfs_dir_cache",
					      sizeof(struct sysfs_dirent),
					      0, 0, NULL);
	if (!sysfs_dir_cachep)
		goto out;

	err = sysfs_inode_init();
	if (err)
		goto out_err;

	err = register_filesystem(&sysfs_fs_type); //注册一个文件系统
	if (!err) {
		sysfs_mnt = kern_mount(&sysfs_fs_type);
		if (IS_ERR(sysfs_mnt)) {
			printk(KERN_ERR "sysfs: could not mount!\n");
			err = PTR_ERR(sysfs_mnt);
			sysfs_mnt = NULL;
			unregister_filesystem(&sysfs_fs_type);
			goto out_err;
		}
	} else
		goto out_err;
out:
	return err;
out_err:
	kmem_cache_destroy(sysfs_dir_cachep);
	sysfs_dir_cachep = NULL;
	goto out;
}
```

**函数向系统注册一个类型为sysfs_fs_type的文件系统。**

```c
//fs/sysfs/mount.c
static struct file_system_type sysfs_fs_type = {
	.name		= "sysfs",
	.mount		= sysfs_mount,
	.kill_sb	= sysfs_kill_sb,
};
```

#### sysfs数据结构

kobject与sysfs之间的关联不是自动建立的。独立的kobject实例默认情况下并不集成到sysfs。要使一个对象在sysfs文件系统中可见，需要调用kobject_add。但如果kobject是某个内核子系统的成员，那么向sysfs的注册是自动进行的。

##### 目录项

sysfs_dirent是组成sysfs单元的基本数据结构，它是sysfs文件夹或文件在内存中的代表。sysfs_dirent只表示文件类型（文件夹/普通文件/二进制文件/链接文件）及层级关系，其它信息都保存在对应的inode中。我们创建或删除一个sysfs文件或文件夹事实上只是对以sysfs_dirent为节点的树的节点的添加或删除。

```c
<sysfs.h>
struct sysfs_dirent {
	atomic_t s_count; 
    atomic_t s_active;
    struct sysfs_dirent *s_parent;  //父节点
    struct sysfs_dirent *s_sibling; //子节点
    const char *s_name; //文件、目录或符号链接的名称。
    union { 			//关联数据类型
        struct sysfs_elem_dir s_dir;
        struct sysfs_elem_symlink s_symlink;
        struct sysfs_elem_attr s_attr;
        struct sysfs_elem_bin_attr s_bin_attr;
    };
    unsigned int s_flags; //数据项类型及标志位
    ino_t s_ino;
    umode_t s_mode;
    struct iattr *s_iattr;
}; 
```

##### 属性 

###### 向sysfs中添加文件

kobject已经被映射为文件目录。所有的对象层次一个不少的映射成sys下的目标结构。sysfs仅仅是一个漂亮的树，但是没有提供实际数据的文件。因此，需要`kobj_type`来进行支持，有关sysfs的成员如下：

```c
// include/linux/kobject.h
struct kobj_type {
    // ...
    /* 该类型kobj的sysfs操作接口。 */
    const struct sysfs_ops *sysfs_ops;
    /* 该类型kobj自带的缺省属性（文件），这些属性文件在注册kobj时，直接pop为该目录下的文件。 */
    struct attribute **default_attrs;
};
```

下面我们讲解表示属性的数据结构，以及用于声明新属性的机制。

```c
struct attribute {
    const char *name; //设定该属性文件的名字
    struct module *owner; //设定该文件的属主
    mode_t mode; //设定该文件的文件操作权限
};
```

也可以借助下列数据结构，来定义一个属性组： 

```c
struct attribute_group { 
	const char * name; 
    struct attribute ** attrs; 
}; 
```

###### 属性读写方法sysfs_ops

虽然default_attrs列出了默认的属性，sysfs_ops则描述了如何使用它们。

```c
#include <linux/sysfs.h>
struct sysfs_ops {
    /* 读取该 sysfs 文件时该方法被调用 */
    ssize_t (*show)(struct kobject *, struct attribute *, char *);
    /* 写入该 sysfs 文件时该方法被调用 */
    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

提供适当的show和store方法，由声明新属性类型的代码负责。 

对于二进制属性，情况有所不同。这里，用于读取和修改数据的方法，通常对每个属性都是不同的。这一点反映到了数据结构中，其中明确指定了读取、写入和内存映射方法： 

```c
struct bin_attribute { 
    struct attribute attr; 
    size_t size; 
    void *private; 
    ssize_t (*read)(struct kobject *, struct bin_attribute *, char *, loff_t, size_t); 
    ssize_t (*write)(struct kobject *, struct bin_attribute *, char *, loff_t, size_t); 
    int (*mmap)(struct kobject *, struct bin_attribute *attr, struct vm_area_struct *vma); 
}; 
```

size表示与属性关联的二进制数据的长度，而private（通常）指向数据实际存储的位置。

