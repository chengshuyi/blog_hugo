---
title: "Notes3 Shell"
date: 2020-04-25T13:35:31+08:00
description: ""
draft: true
tags: []
categories: []
---

Overview Diagram
  user / kernel
  process = address space + thread(s)
    process is a running program
  app -> printf() -> write() -> SYSTEM CALL -> sys_write() -> ...
  user-level libraries are app's private business
  kernel internal functions are not callable by user
  xv6 has a few dozen system calls; Linux a few hundred
  details today are mostly about UNIX system-call API
    basis for xv6, Linux, OSX, POSIX standard, &c
    jos has very different system-calls; you'll build UNIX calls over jos

Homework solution

* Let's review Homework 2 (sh.c)
  * exec
    why two execv() arguments?
    what happens to the arguments?
    what happens when exec'd process finishes?
    can execv() return?
    how is the shell able to continue after the command finishes?
  * redirect
    how does exec'd process learn about redirects? [kernel fd tables]
    does the redirect (or error exit) affect the main shell?
  * pipe
    ls | wc -l
    what if ls produces output faster than wc consumes it?
    what if ls is slower than wc?
    how does each command decide when to exit?
    what if reader didn't close the write end? [try it]
    what if writer didn't close the read end?
    how does the kernel know when to free the pipe buffer?

  * how does the shell know a pipeline is finished?
    e.g. ls | sort | tail -1

  * what's the tree of processes?
    sh parses as: ls | (sort | tail -1)
          sh
          sh1
      ls      sh2
          sort   tail

  * does the shell need to fork so many times?
    - what if sh didn't fork for pcmd->left? [try it]
      i.e. called runcmd() without forking?
    - what if sh didn't fork for pcmd->right? [try it]
      would user-visible behavior change?
      sleep 10 | echo hi

  * why wait() for pipe processes only after both are started?
    what if sh wait()ed for pcmd->left before 2nd fork? [try it]
      ls | wc -l
      cat < big | wc -l

  * the point: the system calls can be combined in many ways
    to obtain different behaviors.

Let's look at the challenge problems

 * How to implement sequencing with ";"?
   gcc sh.c ; ./a.out
   echo a ; echo b
   why wait() before scmd->right? [try it]

 * How to implement "&"?
   $ sleep 5 & 
   $ wait
   the implementation of & and wait is in main -- why?
   What if a background process exits while sh waits for a foreground process?

 * How to implement nesting?
   $ (echo a; echo b) | wc -l
   my ( ... ) implementation is only in sh's parser, not runcmd()
   it's neat that sh pipe code doesn't have to know it's applying to a sequence

 * How do these differ? 
   echo a > x ; echo b > x
   ( echo a ; echo b ) > x
   what's the mechanism that avoids overwriting?

UNIX system call observations

* The fork/exec split looks wasteful -- fork() copies mem, exec() discards.
  why not e.g. pid = forkexec(path, argv, fd0, fd1) ?
  the fork/exec split is useful:
    fork(); I/O redirection; exec()
      or fork(); complex nested command; exit.
      as in ( cmd1 ; cmd2 ) | cmd3
    fork() alone: parallel processing
    exec() alone: /bin/login ... exec("/bin/sh")
  fork is cheap for small programs -- on my machine:
    fork+exec takes 400 microseconds (2500 / second)
    fork alone takes 80 microseconds (12000 / second)
    some tricks are involved -- you'll implement them in jos!

* The file descriptor design:
  * FDs are a level of indirection
    - a process's real I/O environment is hidden in the kernel
    - preserved over fork and exec
    - separates I/O setup from use
    - imagine writefile(filename, offset, buf size)
  * FDs help make programs more general purpose: don't need special cases for
    files vs console vs pipe

* Philosophy: small set of conceptually simple calls that combine well
  e.g. fork(), open(), dup(), exec()
  command-line design has a similar approach
    ls | wc -l

* Why must kernel support pipes -- why not have sh simulate them, e.g.
  ls > tempfile ; wc -l < tempfile

* System call interface simple, just ints and char buffers.  why not have open()
  return a pointer reference to a kernel file object?

* The core UNIX system calls are ancient; have they held up well?
  yes; very successful
    and evolved well over many years
  history: design caters to command-line and s/w development
    system call interface is easy for programmers to use
    command-line users like named files, pipelines, &c
    important for development, debugging, server maintenance
  but the UNIX ideas are not perfect:
    programmer convenience is often not very valuable for system-call API
      programmers use libraries e.g. Python that hide sys call details
      apps may have little to do with files &c, e.g. on smartphone
    some UNIX abstractions aren't very efficient
      fork() for multi-GB process is very slow
      FDs hide specifics that may be important
        e.g. block size for on-disk files
        e.g. timing and size of network messages
  so there has been lots of work on alternate plans
    sometimes new system calls and abstractions for existing UNIX-like kernels
    sometimes entirely new approaches to what a kernel should do
  ask "why this way? wouldn't design X be better?"

OS organization

* main goal: isolation

  * Processors provide user/kernel mode
     kernel mode: can execute "privileged" instructions
       e.g., setting kernel/user bit
     user mode: cannot execute privileged instructions

  * Operating system runs in kernel mode
    - kernel is "trusted"
      can set user/kernel bit
      direct hardware access
	  
  * Applications run in user mode
    - kernel sets up per-process isolated address space
    - system calls switch between user and kernel mode
      the application executes a special instruction to enter kernel
      hardware switches to kernel mode
      but only at an entry point specified by the kernel

* What to put in the kernel?

  * xv6 follows a traditional design: all of the OS runs in kernel mode
    - one big program with file system, drivers, &c
    - this design is called a monolithic kernel
    - kernel interface == system call interface
    - good: easy for subsystems to cooperate
      one cache shared by file system and virtual memory
    - bad: interactions are complex
      leads to bugs
      no isolation within kernel

  * microkernel design
    - many OS services run as ordinary user programs
      file system in a file server
    - kernel implements minimal mechanism to run services in user space
      processes with memory
      inter-process communication (IPC)
    - kernel interface != system call interface		
    - good: more isolation
    - bad: may be hard to get good performance

  * exokernel: no abstractions
    apps can use hardware semi-directly, but O/S isolates
    e.g. app can read/write own page table, but O/S audits
    e.g. app can read/write disk blocks, but O/S tracks block owners
    good: more flexibility for demanding applications
    jos will be a mix of microkernel and exokernel

* Can one have process isolation WITHOUT h/w-supported kernel/user mode?
  yes!
  see Singularity O/S, later in semester
  but h/w user/kernel mode is the most popular plan

Next lecture: isolation mechanisms and xv6's use of them