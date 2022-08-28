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

