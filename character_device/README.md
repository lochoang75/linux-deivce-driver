# Chacracter device driver:
## Major and Minor Numbers:
- Modern Linux kernels allow multiple drivers to share major numbers, but most devices that you will see are still organized on the **one-major-one-driver** principle.
- The major number is used by the kernel to determine exactly which device is being referred to.
### The Internal Representation of Device Numbers:
- From kernel 2.6.0 dev_t is a 32-bit quanity with 12 bits set aside for the major number and 20 for the minor number.
- To get kernel major use: `MAJOR(dev_t dev);`
- To get kernel minor use: `MINOR(dev_t dev);`
- To convert major and minor to dev_t: `MKDEV(int major, int minor)`
### Allocating and Freeing Device Numbers:
- Driver need to obatain one or more device numbers to work with: `int register_chrdev_region(dev_t first, unsigned int count, char *name);`
    + **first** is the beginning device number of the range you would like to allocate.
    + **count** is the total number of contiguous device numbers you are requesting.
    + **name** is the name of the device that should be associated with this number range appear in procdevices and sysfs
    + Function will return 0 if successful, in case of error a negative error code will be returned.
- To request kernel to allocate a major number for you on the fly: `int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name)`
    + **dev** is an output only parameter
- We should free them when they are no longer in use: `void unregister_chrdev_region(dev_t first, unsigend int count);`
    + usal call in clean up function
### Dynamic allocation of Major numbers:
- Major device number an be allocate dynamic using: `alloc_chrdev_region`
- The disadvantage of dynamic assignment is that you can't create the device nodes in advance, because the major number assigned to your module will vary.
## Some important data structure:
- The **file_operations** structure is how a char driver sets up this connection.
- The **file_operations** define in `<linux/fs.h>` is a collection of function pointers.
- Each field in the structure must point to the function in the driver that implements a specific operation, or left NULL for unsupported operations.
- Some field in file_operations:
    + `struct module *owner`: Pointer to the module that "owns" the structure. Used to prevent the module from being unloaded while it's openrations are in use.
    + `loff_t (*llseek) (struct file *, loff_t, int);` this method is used to change th current read/write position in a file. If success return a (positive) return value.
    If error this function will return a negative return value.
    + `ssize_t (*read) (struct file, char user, size_t, loff_t*);` this method used to retrieve data from device. A nonnegative return value represents the number of bytes successfully read.
    + `ssize_t (*aio_read)(struct kiocb, char user, size_t, loff_t);` Initiates an asynchronous read. Because it's async, read operation might not complete before the function returns.
    If this method is `NULL` this mean method is not support, `read` will be performed instead.
    + `ssize_t (*aio_write)(struct kiocb, const char user, size_t, loff_t*);` Initiates an asynchronous write operation on the device.
    + `int (*readdir) (struct file, void, filldir_t);` This field should be NULL for device files; it is used for reading directories and is useful only for file systems.
    + `unsigned int (*poll) (struct file, struct poll_table_struct);` the poll method is the back end for three system call poll, epoll, and select. The **poll** method should return a bit mask indicating whether nonblocking reads or writes are possible, and, possibly, provide the kernel with information that can be used to put the calling process to sleep until I/O becomes possible.
    + `int (*ioctl) (struct inode, struct file, unsigned int, unsigned long);` The ioctl system call offers a way to issue device-specific commands.
    + `int (*mmap) (struct file, struct vm_area_struct);` used to request a mapping of device memory to process's address space.
    + `int (*flush) (struct file *)` The flush operation is invoked when a process closes its copy of a file descriptor for a device, it should execute (and wait for) any outstading operations on the device.
    + `int (*release) (struct inode, struct file);` invoke when file structure is being released.
    + `int (*fsync) (struct file, struct dentry, int);` backend of the fsync system call, which a user calls to flush any pending data.
    + `int (*aio_fsync)(struct kiocb *, int);` async version of fsync method.
    + `int (*fasync) (int, struct file *, int);` notify the device of a change in its FASYNC flag.
    + `int (*lock) (struct file, int, struct file_lock);` use to implement file locking.
    + `ssize_t (*readv) (struct file, const struct iovec, unsigned long, loff_t*)`
    + `ssize_t (*writev) (struct file, const struct iovec, unsigned long, loff_t*);` Implement scatter/gather read and write operations.
    + `ssize_t (*sendfile) (struct file, loff_t, size_t, read_actor_t, void *);` Implements the read side of the sendifle system call.
    + `ssize_t (*sendpage) (struct file, struct page, int, size_t, loff_t *, int);` send one page at the time to corresponding file.
    + `unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);` find a suitable location in the process's address space to map in a memory segment on device.
    + `int (*check_flags)(int)` allow module to check the flags passed to an fcntl(F_SETFL...) call
    + `int (*dir_notify)(struct file *, unsigned long);`call when use fcntl to requesst directory implement dir_notify.

