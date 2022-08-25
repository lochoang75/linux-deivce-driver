# load function:
- `insmod` can resolves undefined symbols against the table of public kernel symbols
- `modprobe` can do the same way as `insmod` but it also loads any other modules that require by the module you want to load.
# Include specify:
- `module.h` contains a great many definitions of symbols and functions needed by loadable modules.
- `moduleparam.h` can be use to enable the passing of parameters to the module at load time
- Some symbols can appear in device driver are:
    + `MODULE_LICENSE("GPL")` use to declare a module license of this module is GPL, it can be GPL, GPLv2, GPL and aditionals rights, Dual BSD/GPL, Dual MPL/GPL and Propritary.
    + `MODULE_AUTHOR` use to declare who wrote this module
    + `MODULE_DESCRIPTION` a human readable of what the module does
    + `MODULE_VERSION` for a code revision number
    + `MODULE_ALIAS` which this module can be know (alias name)
    + `MODULE_DEVICE_TABLE` let user space know which devices the module supports
# Basic function needed for a module:
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

