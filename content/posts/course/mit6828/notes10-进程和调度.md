---
title: "Notes10 进程和调度"
date: 2020-04-25T13:36:21+08:00
description: ""
draft: true
tags: []
categories: []
---





Kernel developers have used a variety of techniques to

improve concurrency, including fifine-grained locks, lock

free data structures, per-CPU data structures, and read

copy-update (RCU)



The success of RCU is, in part, due to its high perfor

mance in the presence of concurrent readers and updaters.

The RCU API facilitates this with two relatively simple

primitives: readers access data structures within *RCU*

*read-side critical sections*, while updaters use *RCU syn*

*chronization* to wait for all pre-existing RCU read-side

critical sections to complete.



The primary RCU requirement is support for concur

rent reading of a data structure, even during updates.

The second RCU requirement is low space and execu

tion overhead.



### Process scheduling

Process
  an abstract virtual machine, as if it had its own CPU and memory,
    not accidentally affected by other processes.
  motivated by isolation

Process API:
  fork
  exec
  exit
  wait
  kill
  sbrk
  getpid

Challenge: more processes than processors
  your laptop has two processors
  you want to run three programs: window system, editor, compiler
  we need to multiplex N processors among M processes
  called time-sharing, scheduling, context switching

xv6 solution:
  1 user thread and 1 kernel thread per process
  1 scheduler thread per processor
  n processors

What's a thread?
  a CPU core executing (with registers and stack), or
  a saved set of registers and a stack that could execute

Overview of xv6 processing switching
  user -> kernel thread (via system call or timer)
  kernel thread yields, due to pre-emption or waiting for I/O
  kernel thread -> scheduler thread
  scheduler thread finds a RUNNABLE kernel thread
  scheduler thread -> kernel thread
  kernel thread -> user

Each xv6 process has a proc->state
  RUNNING
  RUNNABLE
  SLEEPING
  ZOMBIE
  UNUSED

Note:
  xv6 has lots of kernel threads sharing the single kernel address space
  xv6 has only one user thread per process
  more serious O/S's (e.g. Linux) support multiple user threads per process

Context-switching was one of the hardest things to get right in xv6
  multi-core
  locking
  interrupts
  process termination

### Code

pre-emptive switch demonstration
  hog.c -- two CPU-bound processes
  my qemu has only one CPU
  let's look at how xv6 switches between them

swtch -- to scheduler thread
  a context holds a non-executing kernel thread's saved registers
    xv6 contexts always live on the stack
    context pointer is effectively the saved esp
    (remember that *user* registers are in trapframe on stack)
    proc.h, struct context
  two arguments: from and to contexts
    push registers and save esp in *from
    load esp from to, pop registers, return
  confirm switching to scheduler()
    p/x *mycpu()->scheduler
    p/x &scheduler
  stepi -- and look at swtch.S
    [draw two stacks]
    eip (saved by call instruction)
    ebp, ebx, esi, edi
    save esp in *from
    load esp from to argument
    x/8x $esp
    pops
    ret
    where -- we're now on scheduler's stack

scheduler()
  print p->state
  print p->name
  print p->pid
  swtch just returned from a *previous* scheduler->process switch
  scheduler releases old page table, cpu->proc
    switchkvm() in vm.c
  next a few times -- scheduler() finds other process
  print p->pid
  switchuvm() in vm.c
  stepi through swtch()
    what's on the thread stack? context/callrecords/trapframe
    returning from timer interrupt to user space
    where shows trap/yield/sched

Q: what is the scheduling policy? 
   will the thread that called yield() run immediately again?

Q: why does scheduler() release after loop, and re-acquire it immediately?
   To give other processors a chance to use the proc table
   Otherwise two cores and one process = deadlock

Q: why does the scheduler() briefly enable interrupts?
   There may be no RUNNABLE threads
     They may all be waiting for I/O, e.g. disk or console
   Enable interrupts so device has a chance to signal completion
     and thus wake up a thread

Q: why does yield() acquire ptable.lock, but scheduler() release it?
   unusual: the lock is released by a different thread than acquired it!
   why must the lock remain held across the swtch()?
   what if another core's scheduler() immediately saw the RUNNABLE process?

 sched() and scheduler() are "co-routines"
   caller knows what it is swtch()ing to
   callee knows where switch is coming from
   e.g. yield() and scheduler() cooperate about ptable.lock
   different from ordinary thread switching, where neither
     party typically knows which thread comes before/after

Q: how do we know scheduler() thread is ready for us to swtch() into?
   could it be anywhere other than swtch()?

these are some invariants that ptable.lock protects:
  if RUNNING, processor registers hold the values (not in context)
  if RUNNABLE, context holds its saved registers
  if RUNNABLE, no processor is using its stack
  holding the lock from yield() all the way to scheduler enforces:
    interrupts off, so timer can't invalidate swtch save/restore
    another CPU can't execute until after stack switch

Q: is there pre-emptive scheduling of kernel threads?
   what if timer interrupt while executing in the kernel?
   what does kernel thread stack look like?

Q: why forbid locks from being held when yielding the CPU?
   (other than ptable.lock)
   i.e. sched() checks that ncli == 1
   an acquire may waste a lot of time spinnning
   worse: deadlock, since acquire waits with interrupts off

Thread clean up
  look at kill(pid)
    stops the target process
  can kill() free killed process's resources? memory, FDs, &c?
    too hard: it might be running, holding locks, etc.
    so a process must kill itself
    trap() checks p->killed
    and calls exit()
  look at exit()
    the killed process runs exit()
    can a thread free its own stack?
      no: it is using it, and needs it to swtch() to scheduler().
    so exit() sets proc->state = ZOMBIE; parent finishes cleanup
    ZOMBIE child is guaranteed *not* to be executing / using stack!
  wait() does the final cleanup
    the parent is expected to call the wait() system call
    stack, pagetable, proc[] slot