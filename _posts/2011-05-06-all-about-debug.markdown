---
author: sunsj1231
comments: true
date: 2011-05-06 07:35:56
layout: post
slug: all-about-debug
title: all about debug
wordpress_id: 44001
categories:
- coding
---

[http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/)

[http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/)

[http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/)


## [<!-- more -->How debuggers work: Part 1 – Basics](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/)


This is the first part in a series of articles on how debuggers work. I’m still not sure how many articles the series will contain and what topics it will cover, but I’m going to start with the basics.





### In this part


I’m going to present the main building block of a debugger’s implementation on Linux – the ptracesystem call. All the code in this article is developed on a 32-bit Ubuntu machine. Note that the code is very much platform specific, although porting it to other platforms shouldn’t be too difficult.









### Motivation


To understand where we’re going, try to imagine what it takes for a debugger to do its work. A debugger can start some process and debug it, or attach itself to an existing process. It can single-step through the code, set breakpoints and run to them, examine variable values and stack traces. Many debuggers have advanced features such as executing expressions and calling functions in the debbugged process’s address space, and even changing the process’s code on-the-fly and watching the effects.

Although modern debuggers are complex beasts [[1]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id7), it’s surprising how simple is the foundation on which they are built. Debuggers start with only a few basic services provided by the operating system and the compiler/linker, all the rest is just [a simple matter of programming](http://en.wikipedia.org/wiki/Small_matter_of_programming).









### Linux debugging – ptrace


The Swiss army knife of Linux debuggers is the ptrace system call [[2]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id8). It’s a versatile and rather complex tool that allows one process to control the execution of another and to peek and poke at its innards [[3]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id9). ptrace can take a mid-sized book to explain fully, which is why I’m just going to focus on some of its practical uses in examples.

Let’s dive right in.









### Stepping through the code of a process


I’m now going to develop an example of running a process in "traced" mode in which we’re going to single-step through its code – the machine code (assembly instructions) that’s executed by the CPU. I’ll show the example code in parts, explaining each, and in the end of the article you will find a link to download a complete C file that you can compile, execute and play with.

The high-level plan is to write code that splits into a child process that will execute a user-supplied command, and a parent process that traces the child. First, the main function:




    
    int main(int argc, char** argv)
    {
        pid_t child_pid;
    
        if (argc < 2) {
            fprintf(stderr, "Expected a program name as argument\n");
            return -1;
        }
    
        child_pid = fork();
        if (child_pid == 0)
            run_target(argv[1]);
        else if (child_pid > 0)
            run_debugger(child_pid);
        else {
            perror("fork");
            return -1;
        }
    
        return 0;
    }





Pretty simple: we start a new child process with fork [[4]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id10). The if branch of the subsequent condition runs the child process (called "target" here), and the else if branch runs the parent process (called "debugger" here).

Here’s the target process:




    
    void run_target(const char* programname)
    {
        procmsg("target started. will run '%s'\n", programname);
    
        /* Allow tracing of this process */
        if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0) {
            perror("ptrace");
            return;
        }
    
        /* Replace this process's image with the given program */
        execl(programname, programname, 0);
    }





The most interesting line here is the ptrace call. ptrace is declared thus (in sys/ptrace.h):




    
    long ptrace(enum __ptrace_request request, pid_t pid,
                     void *addr, void *data);





The first argument is a _request_, which may be one of many predefined PTRACE_* constants. The second argument specifies a process ID for some requests. The third and fourth arguments are address and data pointers, for memory manipulation. The ptrace call in the code snippet above makes the PTRACE_TRACEME request, which means that this child process asks the OS kernel to let its parent trace it. The request description from the man-page is quite clear:


> Indicates that this process is to be traced by its parent. Any signal (except SIGKILL) delivered to this process will cause it to stop and its parent to be notified via wait(). **Also, all subsequent calls to exec() by this process will cause a SIGTRAP to be sent to it, giving the parent a chance to gain control before the new program begins execution**. A process probably shouldn’t make this request if its parent isn’t expecting to trace it. (pid, addr, and data are ignored.)


I’ve highlighted the part that interests us in this example. Note that the very next thing run_targetdoes after ptrace is invoke the program given to it as an argument with execl. This, as the highlighted part explains, causes the OS kernel to stop the process just before it begins executing the program in execl and send a signal to the parent.

Thus, time is ripe to see what the parent does:




    
    void run_debugger(pid_t child_pid)
    {
        int wait_status;
        unsigned icounter = 0;
        procmsg("debugger started\n");
    
        /* Wait for child to stop on its first instruction */
        wait(&wait_status);
    
        while (WIFSTOPPED(wait_status)) {
            icounter++;
            /* Make the child execute another instruction */
            if (ptrace(PTRACE_SINGLESTEP, child_pid, 0, 0) < 0) {
                perror("ptrace");
                return;
            }
    
            /* Wait for child to stop on its next instruction */
            wait(&wait_status);
        }
    
        procmsg("the child executed %u instructions\n", icounter);
    }





Recall from above that once the child starts executing the exec call, it will stop and be sent theSIGTRAP signal. The parent here waits for this to happen with the first wait call. wait will return once something interesting happens, and the parent checks that it was because the child was stopped (WIFSTOPPED returns true if the child process was stopped by delivery of a signal).

What the parent does next is the most interesting part of this article. It invokes ptrace with thePTRACE_SINGLESTEP request giving it the child process ID. What this does is tell the OS – _please restart the child process, but stop it after it executes the next instruction_. Again, the parent waits for the child to stop and the loop continues. The loop will terminate when the signal that came out of thewait call wasn’t about the child stopping. During a normal run of the tracer, this will be the signal that tells the parent that the child process exited (WIFEXITED would return true on it).

Note that icounter counts the amount of instructions executed by the child process. So our simple example actually does something useful – given a program name on the command line, it executes the program and reports the amount of CPU instructions it took to run from start to finish. Let’s see it in action.









### A test run


I compiled the following simple program and ran it under the tracer:




    
    #include <stdio.h>
    
    int main()
    {
        printf("Hello, world!\n");
        return 0;
    }





To my surprise, the tracer took quite long to run and reported that there were more than 100,000 instructions executed. For a simple printf call? What gives? The answer is very interesting [[5]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id11). By default, gcc on Linux links programs to the C runtime libraries dynamically. What this means is that one of the first things that runs when any program is executed is the dynamic library loader that looks for the required shared libraries. This is quite a lot of code – and remember that our basic tracer here looks at each and every instruction, not of just the main function, but _of the whole process_.

So, when I linked the test program with the -static flag (and verified that the executable gained some 500KB in weight, as is logical for a static link of the C runtime), the tracing reported only 7,000 instructions or so. This is still a lot, but makes perfect sense if you recall that libc initialization still has to run before main, and cleanup has to run after main. Besides, printf is a complex function.

Still not satisfied, I wanted to see something _testable_ – i.e. a whole run in which I could account for every instruction executed. This, of course, can be done with assembly code. So I took this version of "Hello, world!" and assembled it:




    
    section    .text
        ; The _start symbol must be declared for the linker (ld)
        global _start
    
    _start:
    
        ; Prepare arguments for the sys_write system call:
        ;   - eax: system call number (sys_write)
        ;   - ebx: file descriptor (stdout)
        ;   - ecx: pointer to string
        ;   - edx: string length
        mov    edx, len
        mov    ecx, msg
        mov    ebx, 1
        mov    eax, 4
    
        ; Execute the sys_write system call
        int    0x80
    
        ; Execute sys_exit
        mov    eax, 1
        int    0x80
    
    section   .data
    msg db    'Hello, world!', 0xa
    len equ    $ - msg





Sure enough. Now the tracer reported that 7 instructions were executed, which is something I can easily verify.









### Deep into the instruction stream


The assembly-written program allows me to introduce you to another powerful use of ptrace – closely examining the state of the traced process. Here’s another version of the run_debuggerfunction:




    
    void run_debugger(pid_t child_pid)
    {
        int wait_status;
        unsigned icounter = 0;
        procmsg("debugger started\n");
    
        /* Wait for child to stop on its first instruction */
        wait(&wait_status);
    
        while (WIFSTOPPED(wait_status)) {
            icounter++;
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
            unsigned instr = ptrace(PTRACE_PEEKTEXT, child_pid, regs.eip, 0);
    
            procmsg("icounter = %u.  EIP = 0x%08x.  instr = 0x%08x\n",
                        icounter, regs.eip, instr);
    
            /* Make the child execute another instruction */
            if (ptrace(PTRACE_SINGLESTEP, child_pid, 0, 0) < 0) {
                perror("ptrace");
                return;
            }
    
            /* Wait for child to stop on its next instruction */
            wait(&wait_status);
        }
    
        procmsg("the child executed %u instructions\n", icounter);
    }





The only difference is in the first few lines of the while loop. There are two new ptrace calls. The first one reads the value of the process’s registers into a structure. user_regs_struct is defined insys/user.h. Now here’s the fun part – if you look at this header file, a comment close to the top says:




    
    /* The whole purpose of this file is for GDB and GDB only.
       Don't read too much into it. Don't use it for
       anything other than GDB unless know what you are
       doing.  */





Now, I don’t know about you, but it makes _me_ feel we’re on the right track ![:-)](http://eli.thegreenplace.net/wp-includes/images/smilies/icon_smile.gif) Anyway, back to the example. Once we have all the registers in regs, we can peek at the current instruction of the process by calling ptrace with PTRACE_PEEKTEXT, passing it regs.eip (the extended instruction pointer on x86) as the address. What we get back is the instruction [[6]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id12). Let’s see this new tracer run on our assembly-coded snippet:




    
    $ simple_tracer traced_helloworld
    [5700] debugger started
    [5701] target started. will run 'traced_helloworld'
    [5700] icounter = 1.  EIP = 0x08048080.  instr = 0x00000eba
    [5700] icounter = 2.  EIP = 0x08048085.  instr = 0x0490a0b9
    [5700] icounter = 3.  EIP = 0x0804808a.  instr = 0x000001bb
    [5700] icounter = 4.  EIP = 0x0804808f.  instr = 0x000004b8
    [5700] icounter = 5.  EIP = 0x08048094.  instr = 0x01b880cd
    Hello, world!
    [5700] icounter = 6.  EIP = 0x08048096.  instr = 0x000001b8
    [5700] icounter = 7.  EIP = 0x0804809b.  instr = 0x000080cd
    [5700] the child executed 7 instructions





OK, so now in addition to icounter we also see the instruction pointer and the instruction it points to at each step. How to verify this is correct? By using objdump -d on the executable:




    
    $ objdump -d traced_helloworld
    
    traced_helloworld:     file format elf32-i386
    
    Disassembly of section .text:
    
    08048080 <.text>:
     8048080:     ba 0e 00 00 00          mov    $0xe,%edx
     8048085:     b9 a0 90 04 08          mov    $0x80490a0,%ecx
     804808a:     bb 01 00 00 00          mov    $0x1,%ebx
     804808f:     b8 04 00 00 00          mov    $0x4,%eax
     8048094:     cd 80                   int    $0x80
     8048096:     b8 01 00 00 00          mov    $0x1,%eax
     804809b:     cd 80                   int    $0x80





The correspondence between this and our tracing output is easily observed.









### Attaching to a running process


As you know, debuggers can also attach to an already-running process. By now you won’t be surprised to find out that this is also done with ptrace, which can get the PTRACE_ATTACH request. I won’t show a code sample here since it should be very easy to implement given the code we’ve already gone through. For educational purposes, the approach taken here is more convenient (since we can stop the child process right at its start).









### The code


The complete C source-code of the simple tracer presented in this article (the more advanced, instruction-printing version) is available [here](http://eli.thegreenplace.net/files/prog_code/articles_code/debugger/simple_tracer.c). It compiles cleanly with -Wall -pedantic --std=c99on version 4.4 of gcc.









### Conclusion and next steps


Admittedly, this part didn’t cover much – we’re still far from having a real debugger in our hands. However, I hope it has already made the process of debugging at least a little less mysterious.ptrace is truly a versatile system call with many abilities, of which we’ve sampled only a few so far.

Single-stepping through the code is useful, but only to a certain degree. Take the C "Hello, world!" sample I demonstrated above. To get to main it would probably take a couple of thousands of instructions of C runtime initialization code to step through. This isn’t very convenient. What we’d ideally want to have is the ability to place a breakpoint at the entry to main and step from there. Fair enough, and in the next part of the series I intend to show how breakpoints are implemented.









### References


I’ve found the following resources and articles useful in the preparation of this article:



	
  * [Playing with ptrace, Part I](http://www.linuxjournal.com/article/6100?page=0,1)

	
  * [Process tracing using ptrace](http://linuxgazette.net/81/sandeep.html)

	
  * [How debugger works](http://www.alexonlinux.com/how-debugger-works)




![http://eli.thegreenplace.net/wp-content/uploads/hline.jpg](http://eli.thegreenplace.net/wp-content/uploads/hline.jpg)










[[1]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id1)


I didn’t check but I’m sure the LOC count of gdb is at least in the six-figures range.












[[2]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id2)


Run man 2 ptrace for complete enlightment.












[[3]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id3)


_Peek_ and _poke_ are well-known system programming [jargon](http://www.jargon.net/jargonfile/p/peek.html) for directly reading and writing memory contents.












[[4]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id4)


This article assumes some basic level of Unix/Linux programming experience. I assume you know (at least conceptually) about fork, the exec family of functions and Unix signals.












[[5]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id5)


At least if you’re as obsessed with low-level details as I am ![:-)](http://eli.thegreenplace.net/wp-includes/images/smilies/icon_smile.gif)












[[6]](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/#id6)


A word of warning here: as I noted above, a lot of this is highly platform specific. I’m making some simplifying assumptions – for example, x86 instructions don’t have to fit into 4 bytes (the size ofunsigned on my 32-bit Ubuntu machine). In fact, many won’t. Peeking at instructions meaningfully requires us to have a complete disassembler at hand. We don’t have one here, but real debuggers do.









## [How debuggers work: Part 2 – Breakpoints](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/)


This is the second part in a series of articles on how debuggers work. Make sure you read [the first part](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/) before this one.





### In this part


I’m going to demonstrate how breakpoints are implemented in a debugger. Breakpoints are one of the two main pillars of debugging – the other being able to inspect values in the debugged process’s memory. We’ve already seen a preview of the other pillar in part 1 of the series, but breakpoints still remain mysterious. By the end of this article, they won’t be.









### Software interrupts


To implement breakpoints on the x86 architecture, software interrupts (also known as "traps") are used. Before we get deep into the details, I want to explain the concept of interrupts and traps in general.

A CPU has a single stream of execution, working through instructions one by one [[1]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id7). To handle asynchronous events like IO and hardware timers, CPUs use interrupts. A hardware interrupt is usually a dedicated electrical signal to which a special "response circuitry" is attached. This circuitry notices an activation of the interrupt and makes the CPU stop its current execution, save its state, and jump to a predefined address where a handler routine for the interrupt is located. When the handler finishes its work, the CPU resumes execution from where it stopped.

Software interrupts are similar in principle but a bit different in practice. CPUs support special instructions that allow the software to simulate an interrupt. When such an instruction is executed, the CPU treats it like an interrupt – stops its normal flow of execution, saves its state and jumps to a handler routine. Such "traps" allow many of the wonders of modern OSes (task scheduling, virtual memory, memory protection, debugging) to be implemented efficiently.

Some programming errors (such as division by 0) are also treated by the CPU as traps, and are frequently referred to as "exceptions". Here the line between hardware and software blurs, since it’s hard to say whether such exceptions are really hardware interrupts or software interrupts. But I’ve digressed too far away from the main topic, so it’s time to get back to breakpoints.









### int 3 in theory


Having written the previous section, I can now simply say that breakpoints are implemented on the CPU by a special trap called int 3. int is x86 jargon for "trap instruction" – a call to a predefined interrupt handler. x86 supports the int instruction with a 8-bit operand specifying the number of the interrupt that occurred, so in theory 256 traps are supported. The first 32 are reserved by the CPU for itself, and number 3 is the one we’re interested in here – it’s called "trap to debugger".

Without further ado, I’ll quote from the bible itself [[2]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id8):


> The INT 3 instruction generates a special one byte opcode (CC) that is intended for calling the debug exception handler. (This one byte form is valuable because it can be used to replace the first byte of any instruction with a breakpoint, including other one byte instructions, without over-writing other code).


The part in parens is important, but it’s still too early to explain it. We’ll come back to it later in this article.









### int 3 in practice


Yes, knowing the theory behind things is great, OK, but what does this really mean? How do we useint 3 to implement breakpoints? Or to paraphrase common programming Q&A jargon – _Plz show me the codes!_

In practice, this is really very simple. Once your process executes the int 3 instruction, the OS stops it [[3]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id9). On Linux (which is what we’re concerned with in this article) it then sends the process a signal – SIGTRAP.

That’s all there is to it – honest! Now recall from the first part of the series that a tracing (debugger) process gets notified of all the signals its child (or the process it attaches to for debugging) gets, and you can start getting a feel of where we’re going.

That’s it, no more computer architecture 101 jabber. It’s time for examples and code.









### Setting breakpoints manually


I’m now going to show code that sets a breakpoint in a program. The target program I’m going to use for this demonstration is the following:




    
    section    .text
        ; The _start symbol must be declared for the linker (ld)
        global _start
    
    _start:
    
        ; Prepare arguments for the sys_write system call:
        ;   - eax: system call number (sys_write)
        ;   - ebx: file descriptor (stdout)
        ;   - ecx: pointer to string
        ;   - edx: string length
        mov     edx, len1
        mov     ecx, msg1
        mov     ebx, 1
        mov     eax, 4
    
        ; Execute the sys_write system call
        int     0x80
    
        ; Now print the other message
        mov     edx, len2
        mov     ecx, msg2
        mov     ebx, 1
        mov     eax, 4
        int     0x80
    
        ; Execute sys_exit
        mov     eax, 1
        int     0x80
    
    section    .data
    
    msg1    db      'Hello,', 0xa
    len1    equ     $ - msg1
    msg2    db      'world!', 0xa
    len2    equ     $ - msg2





I’m using assembly language for now, in order to keep us clear of compilation issues and symbols that come up when we get into C code. What the program listed above does is simply print "Hello," on one line and then "world!" on the next line. It’s very similar to the program demonstrated in the previous article.

I want to set a breakpoint after the first printout, but before the second one. Let’s say right after the first int 0x80 [[4]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id10), on the mov edx, len2 instruction. First, we need to know what address this instruction maps to. Running objdump -d:




    
    traced_printer2:     file format elf32-i386
    
    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      0 .text         00000033  08048080  08048080  00000080  2**4
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      1 .data         0000000e  080490b4  080490b4  000000b4  2**2
                      CONTENTS, ALLOC, LOAD, DATA
    
    Disassembly of section .text:
    
    08048080 <.text>:
     8048080:     ba 07 00 00 00          mov    $0x7,%edx
     8048085:     b9 b4 90 04 08          mov    $0x80490b4,%ecx
     804808a:     bb 01 00 00 00          mov    $0x1,%ebx
     804808f:     b8 04 00 00 00          mov    $0x4,%eax
     8048094:     cd 80                   int    $0x80
     8048096:     ba 07 00 00 00          mov    $0x7,%edx
     804809b:     b9 bb 90 04 08          mov    $0x80490bb,%ecx
     80480a0:     bb 01 00 00 00          mov    $0x1,%ebx
     80480a5:     b8 04 00 00 00          mov    $0x4,%eax
     80480aa:     cd 80                   int    $0x80
     80480ac:     b8 01 00 00 00          mov    $0x1,%eax
     80480b1:     cd 80                   int    $0x80





So, the address we’re going to set the breakpoint on is 0×8048096. Wait, this is not how real debuggers work, right? Real debuggers set breakpoints on lines of code and on functions, not on some bare memory addresses? Exactly right. But we’re still far from there – to set breakpoints like_real_ debuggers we still have to cover symbols and debugging information first, and it will take another part or two in the series to reach these topics. For now, we’ll have to do with bare memory addresses.

At this point I really want to digress again, so you have two choices. If it’s really interesting for you to know _why_ the address is 0×8048096 and what does it mean, read the next section. If not, and you just want to get on with the breakpoints, you can safely skip it.









### Digression – process addresses and entry point


Frankly, 0×8048096 itself doesn’t mean much, it’s just a few bytes away from the beginning of the text section of the executable. If you look carefully at the dump listing above, you’ll see that the text section starts at 0×08048080. This tells the OS to map the text section starting at this address in the virtual address space given to the process. On Linux these addresses can be absolute (i.e. the executable isn’t being relocated when it’s loaded into memory), because with the virtual memory system each process gets its own chunk of memory and sees the whole 32-bit address space as its own (called "linear" address).

If we examine the ELF [[5]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id11) header with readelf, we get:




    
    $ readelf -h traced_printer2
    ELF Header:
      Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
      Class:                             ELF32
      Data:                              2's complement, little endian
      Version:                           1 (current)
      OS/ABI:                            UNIX - System V
      ABI Version:                       0
      Type:                              EXEC (Executable file)
      Machine:                           Intel 80386
      Version:                           0x1
      Entry point address:               0x8048080
      Start of program headers:          52 (bytes into file)
      Start of section headers:          220 (bytes into file)
      Flags:                             0x0
      Size of this header:               52 (bytes)
      Size of program headers:           32 (bytes)
      Number of program headers:         2
      Size of section headers:           40 (bytes)
      Number of section headers:         4
      Section header string table index: 3





Note the "entry point address" section of the header, which also points to 0×8048080. So if we interpret the directions encoded in the ELF file for the OS, it says:



	
  1. Map the text section (with given contents) to address 0×8048080

	
  2. Start executing at the entry point – address 0×8048080


But still, why 0×8048080? For historic reasons, it turns out. Some googling led me to a few sources that claim that the first 128MB of each process’s address space were reserved for the stack. 128MB happens to be 0×8000000, which is where other sections of the executable may start. 0×8048080, in particular, is the default entry point used by the Linux ld linker. This entry point can be modified by passing the -Ttext argument to ld.

To conclude, there’s nothing really special in this address and we can freely change it. As long as the ELF executable is properly structured and the entry point address in the header matches the real beginning of the program’s code (text section), we’re OK.









### Setting breakpoints in the debugger with int 3


To set a breakpoint at some target address in the traced process, the debugger does the following:



	
  1. Remember the data stored at the target address

	
  2. Replace the first byte at the target address with the int 3 instruction


Then, when the debugger asks the OS to run the process (with PTRACE_CONT as we saw in the previous article), the process will run and eventually hit upon the int 3, where it will stop and the OS will send it a signal. This is where the debugger comes in again, receiving a signal that its child (or traced process) was stopped. It can then:



	
  1. Replace the int 3 instruction at the target address with the original instruction

	
  2. Roll the instruction pointer of the traced process back by one. This is needed because the instruction pointer now points _after_ the int 3, having already executed it.

	
  3. Allow the user to interact with the process in some way, since the process is still halted at the desired target address. This is the part where your debugger lets you peek at variable values, the call stack and so on.

	
  4. When the user wants to keep running, the debugger will take care of placing the breakpoint back (since it was removed in step 1) at the target address, unless the user asked to cancel the breakpoint.


Let’s see how some of these steps are translated into real code. We’ll use the debugger "template" presented in part 1 (forking a child process and tracing it). In any case, there’s a link to the full source code of this example at the end of the article.




    
    /* Obtain and show child's instruction pointer */
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
    procmsg("Child started. EIP = 0x%08x\n", regs.eip);
    
    /* Look at the word at the address we're interested in */
    unsigned addr = 0x8048096;
    unsigned data = ptrace(PTRACE_PEEKTEXT, child_pid, (void*)addr, 0);
    procmsg("Original data at 0x%08x: 0x%08x\n", addr, data);





Here the debugger fetches the instruction pointer from the traced process, as well as examines the word currently present at 0×8048096. When run tracing the assembly program listed in the beginning of the article, this prints:




    
    [13028] Child started. EIP = 0x08048080
    [13028] Original data at 0x08048096: 0x000007ba





So far, so good. Next:




    
    /* Write the trap instruction 'int 3' into the address */
    unsigned data_with_trap = (data & 0xFFFFFF00) | 0xCC;
    ptrace(PTRACE_POKETEXT, child_pid, (void*)addr, (void*)data_with_trap);
    
    /* See what's there again... */
    unsigned readback_data = ptrace(PTRACE_PEEKTEXT, child_pid, (void*)addr, 0);
    procmsg("After trap, data at 0x%08x: 0x%08x\n", addr, readback_data);





Note how int 3 is inserted at the target address. This prints:




    
    [13028] After trap, data at 0x08048096: 0x000007cc





Again, as expected – 0xba was replaced with 0xcc. The debugger now runs the child and waits for it to halt on the breakpoint:




    
    /* Let the child run to the breakpoint and wait for it to
    ** reach it
    */
    ptrace(PTRACE_CONT, child_pid, 0, 0);
    
    wait(&wait_status);
    if (WIFSTOPPED(wait_status)) {
        procmsg("Child got a signal: %s\n", strsignal(WSTOPSIG(wait_status)));
    }
    else {
        perror("wait");
        return;
    }
    
    /* See where the child is now */
    ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
    procmsg("Child stopped at EIP = 0x%08x\n", regs.eip);





This prints:




    
    Hello,
    [13028] Child got a signal: Trace/breakpoint trap
    [13028] Child stopped at EIP = 0x08048097





Note the "Hello," that was printed before the breakpoint – exactly as we planned. Also note where the child stopped – just after the single-byte trap instruction.

Finally, as was explained earlier, to keep the child running we must do some work. We replace the trap with the original instruction and let the process continue running from it.




    
    /* Remove the breakpoint by restoring the previous data
    ** at the target address, and unwind the EIP back by 1 to
    ** let the CPU execute the original instruction that was
    ** there.
    */
    ptrace(PTRACE_POKETEXT, child_pid, (void*)addr, (void*)data);
    regs.eip -= 1;
    ptrace(PTRACE_SETREGS, child_pid, 0, &regs);
    
    /* The child can continue running now */
    ptrace(PTRACE_CONT, child_pid, 0, 0);





This makes the child print "world!" and exit, just as planned.

Note that we don’t restore the breakpoint here. That can be done by executing the original instruction in single-step mode, then placing the trap back and only then do PTRACE_CONT. The debug library demonstrated later in the article implements this.









### More on int 3


Now is a good time to come back and examine int 3 and that curious note from Intel’s manual. Here it is again:


> This one byte form is valuable because it can be used to replace the first byte of any instruction with a breakpoint, including other one byte instructions, without over-writing other code


int instructions on x86 occupy two bytes – 0xcd followed by the interrupt number [[6]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id12). int 3could’ve been encoded as cd 03, but there’s a special single-byte instruction reserved for it – 0xcc.

Why so? Because this allows us to insert a breakpoint without ever overwriting more than one instruction. And this is important. Consider this sample code:




    
        .. some code ..
        jz    foo
        dec   eax
    foo:
        call  bar
        .. some code ..





Suppose we want to place a breakpoint on dec eax. This happens to be a single-byte instruction (with the opcode 0x48). Had the replacement breakpoint instruction been longer than 1 byte, we’d be forced to overwrite part of the next instruction (call), which would garble it and probably produce something completely invalid. But what is the branch jz foo was taken? Then, without stopping on dec eax, the CPU would go straight to execute the invalid instruction after it.

Having a special 1-byte encoding for int 3 solves this problem. Since 1 byte is the shortest an instruction can get on x86, we guarantee than only the instruction we want to break on gets changed.









### Encapsulating some gory details


Many of the low-level details shown in code samples of the previous section can be easily encapsulated behind a convenient API. I’ve done some encapsulation into a small utility library calleddebuglib – its code is available for download at the end of the article. Here I just want to demonstrate an example of its usage, but with a twist. We’re going to trace a program written in C.









### Tracing a C program


So far, for the sake of simplicity, I focused on assembly language targets. It’s time to go one level up and see how we can trace a program written in C.

It turns out things aren’t very different – it’s just a bit harder to find where to place the breakpoints. Consider this simple program:




    
    #include <stdio.h>
    
    void do_stuff()
    {
        printf("Hello, ");
    }
    
    int main()
    {
        for (int i = 0; i < 4; ++i)
            do_stuff();
        printf("world!\n");
        return 0;
    }





Suppose I want to place a breakpoint at the entrance to do_stuff. I’ll use the old friend objdump to disassemble the executable, but there’s a lot in it. In particular, looking at the text section is a bit useless since it contains a lot of C runtime initialization code I’m currently not interested in. So let’s just look for do_stuff in the dump:




    
    080483e4 <do_stuff>:
     80483e4:     55                      push   %ebp
     80483e5:     89 e5                   mov    %esp,%ebp
     80483e7:     83 ec 18                sub    $0x18,%esp
     80483ea:     c7 04 24 f0 84 04 08    movl   $0x80484f0,(%esp)
     80483f1:     e8 22 ff ff ff          call   8048318 <puts@plt>
     80483f6:     c9                      leave
     80483f7:     c3                      ret





Alright, so we’ll place the breakpoint at 0×080483e4, which is the first instruction of do_stuff. Moreover, since this function is called in a loop, we want to keep stopping at the breakpoint until the loop ends. We’re going to use the debuglib library to make this simple. Here’s the complete debugger function:




    
    void run_debugger(pid_t child_pid)
    {
        procmsg("debugger started\n");
    
        /* Wait for child to stop on its first instruction */
        wait(0);
        procmsg("child now at EIP = 0x%08x\n", get_child_eip(child_pid));
    
        /* Create breakpoint and run to it*/
        debug_breakpoint* bp = create_breakpoint(child_pid, (void*)0x080483e4);
        procmsg("breakpoint created\n");
        ptrace(PTRACE_CONT, child_pid, 0, 0);
        wait(0);
    
        /* Loop as long as the child didn't exit */
        while (1) {
            /* The child is stopped at a breakpoint here. Resume its
            ** execution until it either exits or hits the
            ** breakpoint again.
            */
            procmsg("child stopped at breakpoint. EIP = 0x%08X\n", get_child_eip(child_pid));
            procmsg("resuming\n");
            int rc = resume_from_breakpoint(child_pid, bp);
    
            if (rc == 0) {
                procmsg("child exited\n");
                break;
            }
            else if (rc == 1) {
                continue;
            }
            else {
                procmsg("unexpected: %d\n", rc);
                break;
            }
        }
    
        cleanup_breakpoint(bp);
    }





Instead of getting our hands dirty modifying EIP and the target process’s memory space, we just usecreate_breakpoint, resume_from_breakpoint and cleanup_breakpoint. Let’s see what this prints when tracing the simple C code displayed above:




    
    $ bp_use_lib traced_c_loop
    [13363] debugger started
    [13364] target started. will run 'traced_c_loop'
    [13363] child now at EIP = 0x00a37850
    [13363] breakpoint created
    [13363] child stopped at breakpoint. EIP = 0x080483E5
    [13363] resuming
    Hello,
    [13363] child stopped at breakpoint. EIP = 0x080483E5
    [13363] resuming
    Hello,
    [13363] child stopped at breakpoint. EIP = 0x080483E5
    [13363] resuming
    Hello,
    [13363] child stopped at breakpoint. EIP = 0x080483E5
    [13363] resuming
    Hello,
    world!
    [13363] child exited





Just as expected!









### The code


[Here are](http://eli.thegreenplace.net/files/prog_code/articles_code/debugger/debuggers_part2_code.tgz) the complete source code files for this part. In the archive you’ll find:



	
  * debuglib.h and debuglib.c – the simple library for encapsulating some of the inner workings of a debugger

	
  * bp_manual.c – the "manual" way of setting breakpoints presented first in this article. Uses thedebuglib library for some boilerplate code.

	
  * bp_use_lib.c – uses debuglib for most of its code, as demonstrated in the second code sample for tracing the loop in a C program.










### Conclusion and next steps


We’ve covered how breakpoints are implemented in debuggers. While implementation details vary between OSes, when you’re on x86 it’s all basically variations on the same theme – substituting int3 for the instruction where we want the process to stop.

That said, I’m sure some readers, just like me, will be less than excited about specifying raw memory addresses to break on. We’d like to say "break on do_stuff", or even "break on _this_ line indo_stuff" and have the debugger do it. In the next article I’m going to show how it’s done.









### References


I’ve found the following resources and articles useful in the preparation of this article:



	
  * [How debugger works](http://www.alexonlinux.com/how-debugger-works)

	
  * [Understanding ELF using readelf and objdump](http://www.linuxforums.org/articles/understanding-elf-using-readelf-and-objdump_125.html)

	
  * [Implementing breakpoints on x86 Linux](http://mainisusuallyafunction.blogspot.com/2011/01/implementing-breakpoints-on-x86-linux.html)

	
  * [NASM manual](http://www.nasm.us/xdoc/2.09.04/html/nasmdoc0.html)

	
  * [SO discussion of the ELF entry point](http://stackoverflow.com/questions/2187484/elf-binary-entry-point)

	
  * [This Hacker News discussion](http://news.ycombinator.net/item?id=2131894) of the first part of the series

	
  * [GDB Internals](http://www.deansys.com/doc/gdbInternals/gdbint_toc.html)




![http://eli.thegreenplace.net/wp-content/uploads/hline.jpg](http://eli.thegreenplace.net/wp-content/uploads/hline.jpg)










[[1]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id1)


On a high-level view this is true. Down in the gory details, many CPUs today execute multiple instructions in parallel, some of them [not in their original order](http://en.wikipedia.org/wiki/Out-of-order_execution).












[[2]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id2)


The bible in this case being, of course, Intel’s Architecture software developer’s manual, volume 2A.












[[3]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id3)


How can the OS stop a process just like that? The OS registered its own handler for int 3 with the CPU, that’s how!












[[4]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id4)


Wait, int again? Yes! Linux uses int 0x80 to implement system calls from user processes into the OS kernel. The user places the number of the system call and its arguments into registers and executes int 0x80. The CPU then jumps to the appropriate interrupt handler, where the OS registered a procedure that looks at the registers and decides which system call to execute.












[[5]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id5)


[ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (Executable and Linkable Format) is the file format used by Linux for object files, shared libraries and executables.












[[6]](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/#id6)


An observant reader can spot the translation of int 0x80 into cd 80 in the dumps listed above.












## [How debuggers work: Part 3 – Debugging information](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/)


This is the third part in a series of articles on how debuggers work. Make sure you read [the first](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/) and[the second](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/) parts before this one.





### In this part


I’m going to explain how the debugger figures out where to find the C functions and variables in the machine code it wades through, and the data it uses to map between C source code lines and machine language words.









### Debugging information


Modern compilers do a pretty good job converting your high-level code, with its nicely indented and nested control structures and arbitrarily typed variables into a big pile of bits called machine code, the sole purpose of which is to run as fast as possible on the target CPU. Most lines of C get converted into several machine code instructions. Variables are shoved all over the place – into the stack, into registers, or completely optimized away. Structures and objects don’t even _exist_ in the resulting code – they’re merely an abstraction that gets translated to hard-coded offsets into memory buffers.

So how does a debugger know where to stop when you ask it to break at the entry to some function? How does it manage to find what to show you when you ask it for the value of a variable? The answer is – debugging information.

Debugging information is generated by the compiler together with the machine code. It is a representation of the relationship between the executable program and the original source code. This information is encoded into a pre-defined format and stored alongside the machine code. Many such formats were invented over the years for different platforms and executable files. Since the aim of this article isn’t to survey the history of these formats, but rather to show how they work, we’ll have to settle on something. This something is going to be DWARF, which is almost ubiquitously used today as the debugging information format for ELF executables on Linux and other Unix-y platforms.









### The DWARF in the ELF




![http://eli.thegreenplace.net/wp-content/uploads/2011/02/dwarf_logo.gif](http://eli.thegreenplace.net/wp-content/uploads/2011/02/dwarf_logo.gif)


According to [its Wikipedia page](http://en.wikipedia.org/wiki/DWARF), DWARF was designed alongside ELF, although it can in theory be embedded in other object file formats as well [[1]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id7).

DWARF is a complex format, building on many years of experience with previous formats for various architectures and operating systems. It has to be complex, since it solves a very tricky problem – presenting debugging information from any high-level language to debuggers, providing support for arbitrary platforms and ABIs. It would take much more than this humble article to explain it fully, and to be honest I don’t understand all its dark corners well enough to engage in such an endeavor anyway [[2]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id8). In this article I will take a more hands-on approach, showing just enough of DWARF to explain how debugging information works in practical terms.









### Debug sections in ELF files


First let’s take a glimpse of where the DWARF info is placed inside ELF files. ELF defines arbitrary sections that may exist in each object file. A _section header table_ defines which sections exist and their names. Different tools treat various sections in special ways – for example the linker is looking for some sections, the debugger for others.

We’ll be using an executable built from this C source for our experiments in this article, compiled intotracedprog2:




    
    #include <stdio.h>
    
    void do_stuff(int my_arg)
    {
        int my_local = my_arg + 2;
        int i;
    
        for (i = 0; i < my_local; ++i)
            printf("i = %d\n", i);
    }
    
    int main()
    {
        do_stuff(2);
        return 0;
    }





Dumping the section headers from the ELF executable using objdump -h we’ll notice several sections with names beginning with .debug_ – these are the DWARF debugging sections:




    
    26 .debug_aranges 00000020  00000000  00000000  00001037
                     CONTENTS, READONLY, DEBUGGING
    27 .debug_pubnames 00000028  00000000  00000000  00001057
                     CONTENTS, READONLY, DEBUGGING
    28 .debug_info   000000cc  00000000  00000000  0000107f
                     CONTENTS, READONLY, DEBUGGING
    29 .debug_abbrev 0000008a  00000000  00000000  0000114b
                     CONTENTS, READONLY, DEBUGGING
    30 .debug_line   0000006b  00000000  00000000  000011d5
                     CONTENTS, READONLY, DEBUGGING
    31 .debug_frame  00000044  00000000  00000000  00001240
                     CONTENTS, READONLY, DEBUGGING
    32 .debug_str    000000ae  00000000  00000000  00001284
                     CONTENTS, READONLY, DEBUGGING
    33 .debug_loc    00000058  00000000  00000000  00001332
                     CONTENTS, READONLY, DEBUGGING





The first number seen for each section here is its size, and the last is the offset where it begins in the ELF file. The debugger uses this information to read the section from the executable.

Now let’s see a few practical examples of finding useful debug information in DWARF.









### Finding functions


One of the most basic things we want to do when debugging is placing breakpoints at some function, expecting the debugger to break right at its entrance. To be able to perform this feat, the debugger must have some mapping between a function name in the high-level code and the address in the machine code where the instructions for this function begin.

This information can be obtained from DWARF by looking at the .debug_info section. Before we go further, a bit of background. The basic descriptive entity in DWARF is called the Debugging Information Entry (DIE). Each DIE has a tag – its type, and a set of attributes. DIEs are interlinked via sibling and child links, and values of attributes can point at other DIEs.

Let’s run:




    
    objdump --dwarf=info tracedprog2





The output is quite long, and for this example we’ll just focus on these lines [[3]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id9):




    
    <1><71>: Abbrev Number: 5 (DW_TAG_subprogram)
        <72>   DW_AT_external    : 1
        <73>   DW_AT_name        : (...): do_stuff
        <77>   DW_AT_decl_file   : 1
        <78>   DW_AT_decl_line   : 4
        <79>   DW_AT_prototyped  : 1
        <7a>   DW_AT_low_pc      : 0x8048604
        <7e>   DW_AT_high_pc     : 0x804863e
        <82>   DW_AT_frame_base  : 0x0      (location list)
        <86>   DW_AT_sibling     : <0xb3>
    
    <1><b3>: Abbrev Number: 9 (DW_TAG_subprogram)
        <b4>   DW_AT_external    : 1
        <b5>   DW_AT_name        : (...): main
        <b9>   DW_AT_decl_file   : 1
        <ba>   DW_AT_decl_line   : 14
        <bb>   DW_AT_type        : <0x4b>
        <bf>   DW_AT_low_pc      : 0x804863e
        <c3>   DW_AT_high_pc     : 0x804865a
        <c7>   DW_AT_frame_base  : 0x2c     (location list)





There are two entries (DIEs) tagged DW_TAG_subprogram, which is a function in DWARF’s jargon. Note that there’s an entry for do_stuff and an entry for main. There are several interesting attributes, but the one that interests us here is DW_AT_low_pc. This is the program-counter (EIP in x86) value for the beginning of the function. Note that it’s 0x8048604 for do_stuff. Now let’s see what this address is in the disassembly of the executable by running objdump -d:




    
    08048604 <do_stuff>:
     8048604:       55           push   ebp
     8048605:       89 e5        mov    ebp,esp
     8048607:       83 ec 28     sub    esp,0x28
     804860a:       8b 45 08     mov    eax,DWORD PTR [ebp+0x8]
     804860d:       83 c0 02     add    eax,0x2
     8048610:       89 45 f4     mov    DWORD PTR [ebp-0xc],eax
     8048613:       c7 45 (...)  mov    DWORD PTR [ebp-0x10],0x0
     804861a:       eb 18        jmp    8048634 <do_stuff+0x30>
     804861c:       b8 20 (...)  mov    eax,0x8048720
     8048621:       8b 55 f0     mov    edx,DWORD PTR [ebp-0x10]
     8048624:       89 54 24 04  mov    DWORD PTR [esp+0x4],edx
     8048628:       89 04 24     mov    DWORD PTR [esp],eax
     804862b:       e8 04 (...)  call   8048534 <printf@plt>
     8048630:       83 45 f0 01  add    DWORD PTR [ebp-0x10],0x1
     8048634:       8b 45 f0     mov    eax,DWORD PTR [ebp-0x10]
     8048637:       3b 45 f4     cmp    eax,DWORD PTR [ebp-0xc]
     804863a:       7c e0        jl     804861c <do_stuff+0x18>
     804863c:       c9           leave
     804863d:       c3           ret





Indeed, 0x8048604 is the beginning of do_stuff, so the debugger can have a mapping between functions and their locations in the executable.









### Finding variables


Suppose that we’ve indeed stopped at a breakpoint inside do_stuff. We want to ask the debugger to show us the value of the my_local variable. How does it know where to find it? Turns out this is much trickier than finding functions. Variables can be located in global storage, on the stack, and even in registers. Additionally, variables with the same name can have different values in different lexical scopes. The debugging information has to be able to reflect all these variations, and indeed DWARF does.

I won’t cover all the possibilities, but as an example I’ll demonstrate how the debugger can findmy_local in do_stuff. Let’s start at .debug_info and look at the entry for do_stuff again, this time also looking at a couple of its sub-entries:




    
    <1><71>: Abbrev Number: 5 (DW_TAG_subprogram)
        <72>   DW_AT_external    : 1
        <73>   DW_AT_name        : (...): do_stuff
        <77>   DW_AT_decl_file   : 1
        <78>   DW_AT_decl_line   : 4
        <79>   DW_AT_prototyped  : 1
        <7a>   DW_AT_low_pc      : 0x8048604
        <7e>   DW_AT_high_pc     : 0x804863e
        <82>   DW_AT_frame_base  : 0x0      (location list)
        <86>   DW_AT_sibling     : <0xb3>
     <2><8a>: Abbrev Number: 6 (DW_TAG_formal_parameter)
        <8b>   DW_AT_name        : (...): my_arg
        <8f>   DW_AT_decl_file   : 1
        <90>   DW_AT_decl_line   : 4
        <91>   DW_AT_type        : <0x4b>
        <95>   DW_AT_location    : (...)       (DW_OP_fbreg: 0)
     <2><98>: Abbrev Number: 7 (DW_TAG_variable)
        <99>   DW_AT_name        : (...): my_local
        <9d>   DW_AT_decl_file   : 1
        <9e>   DW_AT_decl_line   : 6
        <9f>   DW_AT_type        : <0x4b>
        <a3>   DW_AT_location    : (...)      (DW_OP_fbreg: -20)
    <2><a6>: Abbrev Number: 8 (DW_TAG_variable)
        <a7>   DW_AT_name        : i
        <a9>   DW_AT_decl_file   : 1
        <aa>   DW_AT_decl_line   : 7
        <ab>   DW_AT_type        : <0x4b>
        <af>   DW_AT_location    : (...)      (DW_OP_fbreg: -24)





Note the first number inside the angle brackets in each entry. This is the nesting level – in this example entries with <2> are children of the entry with <1>. So we know that the variable my_local(marked by the DW_TAG_variable tag) is a child of the do_stuff function. The debugger is also interested in a variable’s type to be able to display it correctly. In the case of my_local the type points to another DIE – <0x4b>. If we look it up in the output of objdump we’ll see it’s a signed 4-byte integer.

To actually locate the variable in the memory image of the executing process, the debugger will look at the DW_AT_location attribute. For my_local it says DW_OP_fbreg: -20. This means that the variable is stored at offset -20 from the DW_AT_frame_base attribute of its containing function – which is the base of the frame for the function.

The DW_AT_frame_base attribute of do_stuff has the value 0x0 (location list), which means that this value actually has to be looked up in the location list section. Let’s look at it:




    
    $ objdump --dwarf=loc tracedprog2
    
    tracedprog2:     file format elf32-i386
    
    Contents of the .debug_loc section:
    
        Offset   Begin    End      Expression
        00000000 08048604 08048605 (DW_OP_breg4: 4 )
        00000000 08048605 08048607 (DW_OP_breg4: 8 )
        00000000 08048607 0804863e (DW_OP_breg5: 8 )
        00000000 <End of list>
        0000002c 0804863e 0804863f (DW_OP_breg4: 4 )
        0000002c 0804863f 08048641 (DW_OP_breg4: 8 )
        0000002c 08048641 0804865a (DW_OP_breg5: 8 )
        0000002c <End of list>





The location information we’re interested in is the first one [[4]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id10). For each address where the debugger may be, it specifies the current frame base from which offsets to variables are to be computed as an offset from a register. For x86, bpreg4 refers to esp and bpreg5 refers to ebp.

It’s educational to look at the first several instructions of do_stuff again:




    
    08048604 <do_stuff>:
     8048604:       55          push   ebp
     8048605:       89 e5       mov    ebp,esp
     8048607:       83 ec 28    sub    esp,0x28
     804860a:       8b 45 08    mov    eax,DWORD PTR [ebp+0x8]
     804860d:       83 c0 02    add    eax,0x2
     8048610:       89 45 f4    mov    DWORD PTR [ebp-0xc],eax





Note that ebp becomes relevant only after the second instruction is executed, and indeed for the first two addresses the base is computed from esp in the location information listed above. Once ebpis valid, it’s convenient to compute offsets relative to it because it stays constant while esp keeps moving with data being pushed and popped from the stack.

So where does it leave us with my_local? We’re only really interested in its value after the instruction at 0x8048610 (where its value is placed in memory after being computed in eax), so the debugger will be using the DW_OP_breg5: 8 frame base to find it. Now it’s time to rewind a little and recall that the DW_AT_location attribute for my_local says DW_OP_fbreg: -20. Let’s do the math: -20 from the frame base, which is ebp + 8. We get ebp - 12. Now look at the disassembly again and note where the data is moved from eax – indeed, ebp - 12 is where my_local is stored.









### Looking up line numbers


When we talked about finding functions in the debugging information, I was cheating a little. When we debug C source code and put a breakpoint in a function, we’re usually not interested in the first_machine code_ instruction [[5]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id11). What we’re _really_ interested in is the first _C code_ line of the function.

This is why DWARF encodes a full mapping between lines in the C source code and machine code addresses in the executable. This information is contained in the .debug_line section and can be extracted in a readable form as follows:




    
    $ objdump --dwarf=decodedline tracedprog2
    
    tracedprog2:     file format elf32-i386
    
    Decoded dump of debug contents of section .debug_line:
    
    CU: /home/eliben/eli/eliben-code/debugger/tracedprog2.c:
    File name           Line number    Starting address
    tracedprog2.c                5           0x8048604
    tracedprog2.c                6           0x804860a
    tracedprog2.c                9           0x8048613
    tracedprog2.c               10           0x804861c
    tracedprog2.c                9           0x8048630
    tracedprog2.c               11           0x804863c
    tracedprog2.c               15           0x804863e
    tracedprog2.c               16           0x8048647
    tracedprog2.c               17           0x8048653
    tracedprog2.c               18           0x8048658





It shouldn’t be hard to see the correspondence between this information, the C source code and the disassembly dump. Line number 5 points at the entry point to do_stuff – 0x8040604. The next line, 6, is where the debugger should really stop when asked to break in do_stuff, and it points at0x804860a which is just past the prologue of the function. This line information easily allows bi-directional mapping between lines and addresses:



	
  * When asked to place a breakpoint at a certain line, the debugger will use it to find which address it should put its trap on (remember our friend int 3 from the previous article?)

	
  * When an instruction causes a segmentation fault, the debugger will use it to find the source code line on which it happened.










### libdwarf – Working with DWARF programmatically


Employing command-line tools to access DWARF information, while useful, isn’t fully satisfying. As programmers, we’d like to know how to write actual code that can read the format and extract what we need from it.

Naturally, one approach is to grab the DWARF specification and start hacking away. Now, remember how everyone keeps saying that you should never, ever parse HTML manually but rather use a library? Well, with DWARF it’s even worse. DWARF is _much_ more complex than HTML. What I’ve shown here is just the tip of the iceberg, and to make things even harder, most of this information is encoded in a very compact and compressed way in the actual object file [[6]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id12).

So we’ll take another road and use a library to work with DWARF. There are two major libraries I’m aware of (plus a few less complete ones):



	
  1. BFD (libbfd) is used by the [GNU binutils](http://www.gnu.org/software/binutils/), including objdump which played a star role in this article, ld (the GNU linker) and as (the GNU assembler).

	
  2. libdwarf – which together with its big brother libelf are used for the tools on Solaris and FreeBSD operating systems.


I’m picking libdwarf over BFD because it appears less arcane to me and its license is more liberal (LGPL vs. GPL).

Since libdwarf is itself quite complex it requires a lot of code to operate. I’m not going to show all this code here, but [you can download](http://eli.thegreenplace.net/files/prog_code/articles_code/debugger/dwarf_get_func_addr.c) and run it yourself. To compile this file you’ll need to havelibelf and libdwarf installed, and pass the -lelf and -ldwarf flags to the linker.

The demonstrated program takes an executable and prints the names of functions in it, along with their entry points. Here’s what it produces for the C program we’ve been playing with in this article:




    
    $ dwarf_get_func_addr tracedprog2
    DW_TAG_subprogram: 'do_stuff'
    low pc  : 0x08048604
    high pc : 0x0804863e
    DW_TAG_subprogram: 'main'
    low pc  : 0x0804863e
    high pc : 0x0804865a





The documentation of libdwarf (linked in the References section of this article) is quite good, and with some effort you should have no problem pulling any other information demonstrated in this article from the DWARF sections using it.









### Conclusion and next steps


Debugging information is a simple concept in principle. The implementation details may be intricate, but in the end of the day what matters is that we now know how the debugger finds the information it needs about the original source code from which the executable it’s tracing was compiled. With this information in hand, the debugger bridges between the world of the user, who thinks in terms of lines of code and data structures, and the world of the executable, which is just a bunch of machine code instructions and data in registers and memory.

This article, with its two predecessors, concludes an introductory series that explains the inner workings of a debugger. Using the information presented here and some programming effort, it should be possible to create a basic but functional debugger for Linux.

As for the next steps, I’m not sure yet. Maybe I’ll end the series here, maybe I’ll present some advanced topics such as backtraces, and perhaps debugging on Windows. Readers can also suggest ideas for future articles in this series or related material. Feel free to use the comments or send me an email.









### References





	
  * objdump man page

	
  * Wikipedia pages for [ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) and [DWARF](http://en.wikipedia.org/wiki/DWARF).

	
  * [Dwarf Debugging Standard home page](http://dwarfstd.org/) – from here you can obtain the excellent DWARF tutorial by Michael Eager, as well as the DWARF standard itself. You’ll probably want version 2 since it’s what gcc produces.

	
  * [libdwarf home page](http://reality.sgiweb.org/davea/dwarf.html) – the download package includes a comprehensive reference document for the library

	
  * [BFD documentation](http://sourceware.org/binutils/docs-2.21/bfd/index.html)




![http://eli.thegreenplace.net/wp-content/uploads/hline.jpg](http://eli.thegreenplace.net/wp-content/uploads/hline.jpg)













[[1]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id1)


DWARF is an open standard, published [here](http://dwarfstd.org/) by the DWARF standards committee. The DWARF logo displayed above is taken from that website.












[[2]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id2)


At the end of the article I’ve collected some useful resources that will help you get more familiar with DWARF, if you’re interested. Particularly, start with the DWARF tutorial.












[[3]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id3)


Here and in subsequent examples, I’m placing (...) instead of some longer and un-interesting information for the sake of more convenient formatting.












[[4]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id4)


Because the DW_AT_frame_base attribute of do_stuff contains offset 0x0 into the location list. Note that the same attribute for main contains the offset 0x2c which is the offset for the second set of location expressions.












[[5]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id5)


Where the function prologue is usually executed and the local variables aren’t even valid yet.












[[6]](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/#id6)


Some parts of the information (such as location data and line number data) are encoded as instructions for a specialized virtual machine. Yes, really.





