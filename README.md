# Custom-Linux-Syscall-Process-Info

An educational Linux kernel project implementing a custom system call (ARM64) to securely extract deep process telemetry directly from the `task_struct`. Built for learning kernel architecture, memory boundaries, and RCU locking.

## What is this?

I wrote a custom system call (`sys_process_info`) for the Linux kernel, compiled and tested on a Raspberry Pi 5 (ARM64).

Instead of reading from the `/proc` filesystem like normal user-space tools, this syscall dives directly into the kernel's core `task_struct`, locks the process, extracts scheduling and credential data, and safely passes it back across the memory boundary to user-space.

## Core concepts learned:

* **User/Kernel Memory Boundaries:** Using `__user` annotations, fixed-size UAPI types (`__u32`, `__u64`), and `copy_to_user()` to prevent hardware-level memory violations (like ARM PAN).
* **Concurrency & Locking:** Using `rcu_read_lock()` and `rcu_dereference()` to safely read live process data without panicking the kernel if the process terminates mid-read.
* **Kernel Namespaces:** Digging into the credentials subsystem (`__task_cred`) and using `from_kuid_munged()` to safely translate internal Kernel UIDs to standard user IDs.
* **Circular Linked Lists:** Using kernel macros like `list_for_each()` to navigate process family trees (children/siblings).

## Files Modified/Created

* **`include/uapi/linux/process_info.h`**: The shared struct definition.
* **`process_info/process_info.c`**: The actual `SYSCALL_DEFINE2` logic.
* **`arch/arm64/tools/syscall_64.tbl`**: Added my syscall ID.
* **`include/linux/syscalls.h`**: Added the function prototype.
* **`Kbuild`**: Wired the new folder into the main compilation tree.

## Quick User-Space Test

```c
#include <stdio.h> 
#include <unistd.h> 
#include <sys/syscall.h> 
#include <linux/process_info.h> // Custom UAPI header

#define SYS_PROCESS_INFO 463

int main() { 
    struct process_info_struct info = {0};

    // Testing against PID 1 (usually init/systemd)
    if (syscall(SYS_PROCESS_INFO, 1, &info) == 0) { 
        printf("PID: %d | Parent: %d | UID: %u | Children: %d\n", 
               info.pid, info.parent_pid, info.uid, info.child_count);
    } else {
        perror("Syscall failed");
    }

    return 0;
}
