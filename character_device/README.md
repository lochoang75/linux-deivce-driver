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
- Some field in **struct file**:
    + `mode_t f_mode`: identifies the file as either readable or writable(or both), by mean of the bits FMODE_READ and FMODE_WRITE
    + `loff_t f_pos`: current reading or writing position. loff_t is 64-bit value on all platforms. To change current file position should you `llseek` instead of change it directly.
    + `unsigned int f_flags`: file flags such as O_RDONLY, O_NONBLOCK, O_SYNC... and some seldom used flags define in <linux/fcntl.h>
    + `struct file_operations *f_op`: the file operation associated with the file. We can change the file operations associated with our file, and the new methods will be effective after you return to the caller.
    + `void *private_data`: can be use accross system calls and is used by most of our sample modules.
    + `struct dentry *f_dentry`: the directory entry structure associated with the file.
- Some field need to know about `inode structure`:
    + `dev_t i_rdev`: this field contains actual device number
    + `struct cdev *i_cdev`: struct cdev is the kernel's internal structure that represents char devices;
    + Macro to obtain the major and minor number from an inode:
    `unsigned int iminor(struct inode *inode);`
    `unsigned int imajor(struct inode *indoe);`
## Char Device Regsitration
    + To registration, first our code should include <linux/cdev.h>, where the structure and it's associated helper functions are defined.
    + Two ways of allocating and initializing one of these structures.
        - stadalone cdev structure at run time:
            ```C
            struct cdev *my_cdev = cdev_alloc( );
            my_cdev->ops = &my_fops;
            ```
        - the old way (just to read old source, before kernel 2.6):
            ```C
            int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);
            ```
    + To embed the cdev structure within a deice-specific structure of our own. We initialize the structure that already allocated.
        - `void cdev_init(stuct cdev cdev, struct file_operations fops);`
    + Next set `owner` field of struct cdev to THIS_MODULE
    + Finally, we will add the module as below:
        - `int cdev_add(struct cdev *dev, dev_t num, unsigned int count);`
        - `dev` is a cdev structure
        - `num` is the first device number to which this device responds.
        - `count is the number of device numbers that should be associated with the device. Often is one.
        - cdev_add can failed, if failed it will return a negative error code.
        - Should call cdev_add when your driver is completely ready to handle operations on the device.
    + To remove a char device use:
        - `void cdev_del(struct cdev *dev);`
## Open and Release
- The open method is provided for a driver to do any initialization in preparation for later operations.
    + Open should:
        - Check for device-specific errors (such as device-not-ready or similar hardware problems)
        - Initialize the device if it is being opened for the first time
        - Update the f_op pointer, if necessary
        - Allocate and fill any data structure to be put in filp->private_data
    + Prototype: `int (*open)(struct inode inode, struct file filp);`
        - The inode argument has the information we need in the form of its i_cdev field
        - macro `container_of` int <linux/kernel.h> let we convert pointer from field pointer to container type
- The release method is the reverse of open, can be named as device_close instead of `device_release`
    + Device release should:
        - Deallocate anything that open allocated in filp->private_data
        - Shutdown the device on last close
    + Not all `close()` system call cause release method to be invoked. Kernel keep a counter of how many times a file structure is being used.
## Read and write
- The `read` and `write` methods both perform a similar task, copying data from and to application code.
- Prototype:
    + `ssize_t read(struct file filp, char user buff, size_t count, loff_t *offp);`
    + `ssize_t write(struct file filp, const char user buff, size_t count, loff_t *offp);
- For both methods, `filp` is the file pointer and `count` is the size of the requested data transfer.
- The `buff` argument points to the user buffer holding the data to be written or the empty bufffer where the newly read data should be placed.
- `offp` is a pointer to a "long offset type" object that indicates the file position the user is accessing.
- The return value is a "signed size type"; its use is discussed later.
- `buff` is userspace pointer, therefore, it can't be directly dereferenced by kernel code. Because:
    + Depending on which architecture your driver is running on and how the kernel was configured, the userspace pointer may not be valid while running in kernel mode at all.
    + Event if the pointer does mean the same thing in kernel space, userspace memory is paged, and the memory in question might not be resident in RAM when the system call is made. Attempting to reference the userspace memory directy could generate a page fault.
    + The pointer in question has been supplied by a user program, which could be buggy or malicious.
- To copy data from userspace to kernelspace use: `unsigned long copy_from_user(void *to, const void user *from, unsigned long count);`
- To copy data from kernelspace to userspace use: `unsigned long copy_to_user(void *to, const void user *from, unsigned long count);`
- We don't need to check the userspace pointer in these method, they can do it themself and throw -EAFAULT if user pointer is invalid.
### The read method
- The return value for read is interpreted by te calling application program:
    + If the value equals the count argument passed to the read system call, the requested number of bytes has been transferred. This is the optimal case.
    + If the value is positive but smaller than count, only part of the data has been transferred. Most of cases it require a retries the read.
    + If the value is 0, end-of-file was reached (and no data was read).
    + A negative means there was an error, the values define in `<linux/errno.h>`



