# README

Principles About Linux

# Linux Source

[Linux Source Online](https://elixir.bootlin.com/linux/v3.10/source)

# Network

[SNMP](https://www.kernel.org/doc/html/v5.0/networking/snmp_counter.html)

# Operating System

## Terminology

|   terminology    | desc  |
| ---------------- | ----- |
| Operating system | Operating system: This refers to the software and files that are installed on a system so that it can boot and execute programs. It includes the kernel, administration tools, and system libraries. |
| Kernel           | The kernel is the program that manages the system, including (depending on the kernel model) hardware devices, memory, and CPU scheduling. It runs in a privileged CPU mode that allows direct access to hardware, called kernel mode. | 
| Process          | An OS abstraction and environment for executing a program. The program runs in user mode, with access to kernel mode (e.g., for performing device I/O) via system calls or traps into the kernel |
| Thread           | An executable context that can be scheduled to run on a CPU. The kernel has multiple threads, and a process contains one or more |
| Task             | A Linux runnable entity, which can refer to a process (with a single thread), a thread from a multithreaded process, or kernel threads |
| BPF program      | A kernel-mode program running in the BPF execution environment |
| Main memory      | The physical memory of the system (e.g., RAM) |
| Virtual memory   | An abstraction of main memory that supports multitasking and over-subscription. It is, practically, an infinite resource |
| Address space    | A virtual memory context |
| Kernel space     | The virtual memory address space for the kernel |
| User space       | The virtual memory address space for processes |
| User land        | User-level programs and libraries (/usr/bin, /usr/lib...) |
| Context switch   | A switch from running one thread or process to another. This is a normal function of the kernel CPU scheduler, and involves switching the set of running CPU registers (the thread context) to a new set. |
| Mode switch      | A switch between kernel and user modes |
| System call (syscall) | A well-defined protocol for user programs to request the kernel to perform privileged operations, including device I/O. |
| Processor        | Not to be confused with process, a processor is a physical chip containing one or more CPUs |
| CPU              | Central processing unit. This term refers to the set of functional units that execute instructions, including the registers and arithmetic logic unit(ALU). It is now often used to refer to either the processor or a virtual CPU |
| Trap             | A signal sent to the kernel to request a system routine (privileged action). Trap types include system calls, processor exceptions, and interrupts. |
| Hardware interrupt | A signal sent by physical devices to the kernel, usually to request servicing of I/O. An interrupt is a type of trap |
| Registers        |  Small storage locations on a CPU, used directly from CPU instructions for data processing.|
| File descriptor  | An identifier for a program to use in referencing an open file |

## Concepts

### Kernel

**The kernel model**: a monolithic kernel that manages CPU scheduling, memory, file systems, network protocols, and
system devices (disks, network interfaces, etc.).

![role_of_operating_system_kernel.png](./imgs/role_of_operating_system_kernel.png)
![img.png](imgs/ebpf-applications-kernel-model.png)

**Applications**: include all running user-level software, including databases, web servers, administration tools, and
operating system shells.

**System libraries**: used to provide a richer and easier programming interface than the system calls alone.
Applications can call system calls(syscalls) directly. For example, the Golang runtime has its own syscall layer that
doesn’t require the system library, libc.

**Extended BPF**: Linux has recently changed its model by allowing a new software type: Extended BPF, which enables
secure kernel-mode applications along with its own kernel API: BPF helpers. This allows some applications and system
functions to be rewritten in BPF, providing higher levels of security and performance.

**Kernel execution**: The kernel primarily executes on demand, when a user-level program makes a system call, or a
device sends an interrupt. Some kernel threads operate asynchronously for housekeeping, which may include the kernel
clock routine and memory management tasks, but these try to be lightweight and consume very little CPU resources.

### Kernel-Mode And User-Mode

**Kernel-Mode**: The kernel runs in a special CPU mode called kernel mode, allowing full access to devices and the
execution of privileged instructions.

**User-Mode**: User programs (processes) run in user mode, where they request privileged operations from the kernel via
system calls, such as for I/O.

![system_call_execution_mode.png](imgs/system_call_execution_mode.png)

In a traditional kernel, a system call is performed by switching to kernel mode and then executing the system call code.

Switching between user and kernel modes is a **mode switch**.

All system calls mode switch. Some system calls also **context switch**: those that are blocking, such as for disk and
network I/O, will context switch so that another thread can run while the first is blocked.

Since mode and context switches cost a small amount of overhead (CPU cycles), there are various optimizations to avoid
them, including:

- **User-mode syscalls**: It is possible to implement some syscalls in a user-mode library alone. The Linux kernel does
  this by exporting a virtual dynamic shared object (vDSO) that is mapped into the process address space, which contains
  syscalls such as gettimeofday(2)
  and getcpu(2)
- **Memory mappings**:Used for demand paging, it can also be used for data stores and other I/O, avoiding syscall
  overheads.
- **Kernel bypass**: This allows user-mode programs to access devices directly, bypassing syscalls and the typical
  kernel code path. For example, DPDK for networking: the Data Plane Development Kit.
- **Kernel-mode applications**: the extended BPF technology

### System Call

System calls request the kernel to perform privileged system routines.

| Key System Call | Description |
| ----------- | ----------- |
| read(2)     | Read bytes |
| write(2)    | Write bytes |
| open(2)     | Open a file |
| close(2)    |  Close a file |
| fork(2)     | Create a new process |
| clone(2)    | Create a new process or thread |
| exec(2)     | Execute a new program |
| connect(2)  | Connect to a network host |
| accept(2)   | Accept a network connection |
| stat(2)     | Fetch file statistics |
| ioctl(2)    | Set I/O properties, or other miscellaneous functions |
| mmap(2)     | Map a file to the memory address space |
| brk(2)      | Extend the heap pointer |
| futex(2)    | Fast user-space mutex |

Operating systems generally include a C standard library that provides easier-to-use interfaces for many common
syscalls (e.g., the libc or glibc libraries). You can learn more in its man page.

Many of these system calls have an obvious purpose. Here are a few whose common usage may be less obvious:

- **ioctl(2)**: This is commonly used to request miscellaneous actions from the kernel, especially for system
  administration tools, where another (more obvious) system call isn’t suitable.
- **mmap(2)**: This is commonly used to map executables and libraries to the process address space, and for
  memory-mapped files. It is sometimes used to allocate the working memory of a process, instead of the brk(2)-based
  malloc(2), to reduce the syscall rate and improve performance (which doesn’t always work due to the trade-off
  involved: memory-mapping management).
- **brk(2)**: This is used to extend the heap pointer, which defines the size of the working memory of the process. It
  is typically performed by a system memory allocation library, when a malloc(3) (memory allocate) call cannot be
  satisfied from the existing space in the heap.
- **futex(2)**: This syscall is used to handle part of a user space lock: the part that is likely to block.

The ioctl(2) syscall may be the most difficult to learn, due to its ambiguous nature. As an example of its usage, the
Linux perf(1) tool performs privileged actions to coordinate performance instrumentation. Instead of system calls being
added for each action, a single system call is added: perf_event_open(2), which returns a file descriptor for use with
ioctl(2).This ioctl(2) can then be called using different arguments to perform the different desired actions. For
example, ioctl(fd, PERF_EVENT_IOC_ENABLE) enables instrumentation. The arguments, in this example PERF_EVENT_IOC_ENABLE,
can be more easily added and changed by the developer.




