---
title: "Lab3 - 实验笔记"
date: 2020-04-25T13:34:13+08:00
description: ""
draft: false
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

lab3的实验主要涉及到x86的硬件机制：中断、异常等等。因此，需要自己实现中断向量表、保存和恢复上下文、系统调用和页异常相关的函数。

lab3也新增了用户进程的概念，需要实现相关函数。

### user environments

jos的user environments是tcb的概念，用于描述一个进程实体。其结构如下所示：

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;				// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;			// Number of times environment has run
	int env_cpunum;				// The CPU that the env is running on

	// Address space
	pde_t *env_pgdir;			// Kernel virtual address of page dir

	// Exception handling
	void *env_pgfault_upcall;	// Page fault upcall entry point

	// Lab 4 IPC
	bool env_ipc_recving;		// Env is blocked receiving
	void *env_ipc_dstva;		// VA at which to map received page
	uint32_t env_ipc_value;		// Data value sent to us
	envid_t env_ipc_from;		// envid of the sender
	int env_ipc_perm;			// Perm of page mapping received
};
```

### 中断和异常

中断描述符表：x86最多有256个表项；

TSS：包含了段选择子和栈的基址；

0-31号中断：同步异常；

31号以上中断：软中断和硬中断；

嵌套中断和异常：

> **Exercise 1.** Modify `mem_init()` in `kern/pmap.c` to allocate and map the `envs` array. This array consists of exactly `NENV` instances of the `Env` structure allocated much like how you allocated the `pages` array. Also like the `pages` array, the memory backing `envs` should also be mapped user read-only at `UENVS` (defined in `inc/memlayout.h`) so user processes can read from this array.

为`NENV`个`struct Env`分配内存空间，然后将其映射到位于用户地址空间的虚拟地址UENVS。

```c
//分配内存
envs = (struct Env*)boot_alloc(sizeof(struct Env)*NENV);
memset(envs,0,sizeof(struct Env)*NENV);
//做映射
boot_map_region(kern_pgdir,UENVS,ROUNDUP(sizeof(struct Env)*NENV,PGSIZE),PADDR(envs),PTE_P|PTE_U);
```

>**Exercise 2.** In the file `env.c`, finish coding the following functions:
>
>* `env_init()`
>
>  Initialize all of the `Env` structures in the `envs` array and add them to the `env_free_list`. Also calls `env_init_percpu`, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
>
>* `env_setup_vm()`
>
>  Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
>
>* `region_alloc()`
>
>  Allocates and maps physical memory for an environment
>
>* `load_icode()`
>
>  You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
>
>* `env_create()`
>
>  Allocate an environment with `env_alloc` and call `load_icode` to load an ELF binary into it.
>
>* `env_run()`
>
>  Start a given environment running in user mode.

函数较多，实现简单，这里不再贴代码。

* `env_init`函数的实现类似于`page_init`函数，主要将空闲的`struct Env`插入到`env_free_list`链表上；

* `env_setup_vm`函数为该进程分配一个页目录表，并将内核页目录表内容拷贝到该进程的页目录表；
* `region_alloc`函数为进程分配内存，并将其映射到其地址空间，其参数是`va`和`len`；
* `load_icode`函数解析elf文件头部信息，将其加载到指定的地址上；修改进程的eip值；为该进程创建用户栈；
* `env_alloc`函数工作流程：调用`env_setup_vm`完成进程的页目录表的分配；分配一个id；设置该进程的寄存器的值，包括：ds、es、ss、esp、cs等；
* `env_create`函数先后调用了`env_alloc`函数和`load_icode`函数；
* `env_run`函数根据进程是否发生切换来决定是否更新cr3寄存器的值；

> **Exercise 4.** Edit `trapentry.S` and `trap.c` and implement the features described above. The macros `TRAPHANDLER` and `TRAPHANDLER_NOEC` in `trapentry.S` should help you, as well as the T_* defines in `inc/trap.h`. You will need to add an entry point in `trapentry.S` (using those macros) for each trap defined in `inc/trap.h`, and you'll have to provide `_alltraps` which the `TRAPHANDLER` macros refer to. You will also need to modify `trap_init()` to initialize the `idt` to point to each of these entry points defined in `trapentry.S; the `SETGATE` macro will be helpful here.

具体代码没有粘贴，记录一下具体的中断流程：

1. 中断或者异常发生，cpu判断当前特权级来决定是否切换到内核栈；
2. 如下图所示，如果发生特权级切换需要特别保存ss和esp寄存器（中断嵌套的话只能发生在内核态）；
3. 保存剩余的寄存器到内核栈中（具体可以查看int指令执行的具体内容）；
4. 根据中段号跳转到IDT的相应entry；
5. 压入中断号；
6. 保存用户上下文；
7. 中断处理函数；
8. 恢复用户上下文；
9. 执行iret指令，从内核栈中pop出ss和esp寄存器；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200425200239.png)

### 页异常

页异常的中断向量号是14，当页异常发生时，处理器会自动保存其虚拟地址到`cr2`寄存器；

> **Exercise 5.** Modify `trap_dispatch()` to dispatch page fault exceptions to `page_fault_handler()`. 

```c
		case T_PGFLT:{
			page_fault_handler(tf);
			break;
		}
```

### 断点异常

断点异常的中断向量号是3，调试程序时需要在相应的指令下插入该条指令（int 0x3），触发软件异常，可以调试程序。

> **Exercise 6.** Modify `trap_dispatch()` to make breakpoint exceptions invoke the kernel monitor.

```c
		case T_BRKPT:{
			monitor(tf);
			break;
		}
```

### 系统调用

> **Exercise 7.** Add a handler in the kernel for interrupt vector `T_SYSCALL`. You will have to edit `kern/trapentry.S` and `kern/trap.c`'s `trap_init()`. You also need to change `trap_dispatch()` to handle the system call interrupt by calling `syscall()` (defined in `kern/syscall.c`) with the appropriate arguments, and then arranging for the return value to be passed back to the user process in `%eax`. Finally, you need to implement `syscall()` in `kern/syscall.c`. Make sure `syscall()` returns `-E_INVAL` if the system call number is invalid. You should read and understand `lib/syscall.c` (especially the inline assembly routine) in order to confirm your understanding of the system call interface. Handle all the system calls listed in `inc/syscall.h` by invoking the corresponding kernel function for each call.

实现系统调用部分，为用户程序提供内核服务，具体看代码:-）。

#### 用户程序入口

主要关注几个文件的代码，分别是`lib/entry.S`、`lib/libmain.c`和`lib/exit.c`。

> **Exercise 8.** Add the required code to the user library, then boot your kernel. You should see `user/hello` print "`hello, world`" and then print "`i am environment 00001000`". `user/hello` then attempts to "exit" by calling `sys_env_destroy()` (see `lib/libmain.c` and `lib/exit.c`).

```c
void
libmain(int argc, char **argv)
{
	// set thisenv to point at our Env structure in envs[].
	// LAB 3: Your code here.
	envid_t id = sys_getenvid();
	thisenv = &envs[ENVX(id)];

	// save the name of the program so that panic() can use it
	if (argc > 0)
		binaryname = argv[0];

	// call user main routine
	umain(argc, argv);

	// exit gracefully
	exit();
}
```

#### 内存保护机制

1. 用户进程传进来的指针一定要检查，首先检查该指针不能在内核地址空间，然后判断是否存在该虚拟地址，最后检查读写权限；
2. 内核不能够发生页错误；

> **Exercise 9.** Change `kern/trap.c` to panic if a page fault happens in kernel mode.
>
> Read `user_mem_assert` in `kern/pmap.c` and implement `user_mem_check` in that same file.
>
> Change `kern/syscall.c` to sanity check arguments to system calls.

内核不能够发生页错误。

```c
	if(tf->tf_cs == GD_KT){
		panic("kernel page fault\n");
	}
```

用户进程传进来的指针检查函数：

```c
int user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	uintptr_t s = ROUNDDOWN((uintptr_t)va,PGSIZE);
	uintptr_t e = ROUNDUP((uintptr_t)va+len,PGSIZE);
	pte_t *pte_entry;
	for(;s<e;s+=PGSIZE){
		// cprintf("va is %x ULIM is %x\n",s,ULIM);
		page_lookup(env->env_pgdir,(void *)s,&pte_entry); 
        // 1. 首先检查该指针不能在内核地址空间
        // 2. 然后判断是否存在该虚拟地址
        // 3. 最后检查读写权限
		if(s>ULIM || pte_entry == NULL || (PGOFF(*pte_entry)&perm) != perm){
			user_mem_check_addr = MAX(s,(uintptr_t)va);
			return -E_FAULT;
		}
	}
	return 0;
}
```



