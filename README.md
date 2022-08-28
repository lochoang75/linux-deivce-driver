# Building and running module
## Load function:
- `insmod` can resolves undefined symbols against the table of public kernel symbols
- `modprobe` can do the same way as `insmod` but it also loads any other modules that require by the module you want to load.
## Include specify:
- `module.h` contains a great many definitions of symbols and functions needed by loadable modules.
- `moduleparam.h` can be use to enable the passing of parameters to the module at load time
- Some symbols can appear in device driver are:
    + `MODULE_LICENSE("GPL")` use to declare a module license of this module is GPL, it can be GPL, GPLv2, GPL and aditionals rights, Dual BSD/GPL, Dual MPL/GPL and Propritary.
    + `MODULE_AUTHOR` use to declare who wrote this module
    + `MODULE_DESCRIPTION` a human readable of what the module does
    + `MODULE_VERSION` for a code revision number
    + `MODULE_ALIAS` which this module can be know (alias name)
    + `MODULE_DEVICE_TABLE` let user space know which devices the module supports
## Basic function needed for a module:
- Init function:
    + Initialization functions should be declared `static`, since they are not meant to be visible outside the specific file
    + Init function will follow prototype: `static int init initialization_function(void)`
    + Init function will register to kernel use macro: `module_init(initialization_function)`
    + Can use tag: `initdata` for data used only during initialization
- Clean up function:
    + Clean up function is use to unregisters interfaces and return all resources to the system before the module is removed
    + Clean up function is defined as: `static void exit cleanup_function(void)`
    + Clean up function register use macro: `module_exit(cleanup_function)`
    + The `exit` modifier marks the code as being for module unload only.
    + If your module is built directly into the kernel, or if your kernel is configured to disallow the unloading of modules, functions marked `exit` are simply discarded.
    + If our module does not define a cleanup function, kernel does not allow it to be unloaded.
- Error handling during initialization:
    + The registration could failed.
    + Module must always check return values, and be sure that the requested operations have actually succeeded.
    + If failed to load we have two actions to concern:
        - Do we can continue initilizing itself anyway ?
        - If we can't continue to initialize after some particular type of failure, we must undo any registration activities performed before the failure.
        - Error recovery is sometime best handled with the `goto` statement. In kernel, `goto` is often used as shown here to deal with errors.
        - Error code of init function define in `<linux/errno.h>`
        - If there is many item to clean up within the initialization, call clean up function, which must check the status of each item before undoing registration, can minimize
        the code duplication.
## Module loading races:
- Some other part of the kernel can make use of any facility you register immediately after that registration as completed.
- Do not register any facility until all of your internal initialization needed to support that facility has been completed.
- Must consider what happens if your initialization function decides to fail but some part of the kernel is already making use of a facility your module has registered.
## Module parameters:
- Can be assign at load time by `insmod` or `modprobe`
- Register by macro: `module_param(name, type, perm)`
## Doing it in userspace:
- Sometimes writing a so-called userspace device driver is a wise alternative to kernel hacking.
- The reasons are:
    + The full C library can be linked in.
    + The programmer can run a conventional debugger on the driver code without having go through contortioins to debug a running kernel
    + If a user space driver hangs, you can simply kill it. Problem with the driver are unlikely to hang the entire system, unless the hardware being controlled
    is really misbehaving.
    + User memory is swappable, unlike kernel memory.
    + A well-designed driver program can still, like kernel-space drivers, allow concurrent access to a device.
    + If you must write a closed-source driver, the userspace option makes it easier for you to avoid ambigous licensing situations and problems with changing
    kernel interfaces.
- Some drawbacks:
    + Interrupts are not available in user space.
    + Direct access to memory is possible only by mmapping devmem, and only a privileged user can do that.
    + Access to I/O ports is only after calling ioperm or iopl.
    + Response time is slower, because a context switch is required to transfer information or actions between the client and the hardware.
    + If the driver has been swapped to disk, response time is unacceptably long. Using the mlock system call a userspace program depends on a lot of library code.
    + The most important devices can't be handled in user space, including, but not limited to, network interfaces and block devices.
