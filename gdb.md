



System call-
A system call (syscall) is simply a program asking the computer's core operating system (the kernel) for help. It's the only way a regular program can safely access essential services, like reading a file or sending network data. This request makes the computer temporarily switch into a highly protected mode to do the job.

The initial thing that you ask the Gemini for this question, you get the answer ptrace. ptrace is a syscall.
The next thing i've dont I went to the manual of the ptrace: https://man7.org/linux/man-pages/man2/ptrace.2.html
The first line on the ptrace describes in a good way what does this function(syscall) do:
"The ptrace() system call provides a means by which one process
       (the "tracer") may observe and control the execution of another
       process (the "tracee"), and examine and change the tracee's memory
       and registers."


So after this prolog, important note is that if we want to trace a executable, we trace a thread not a multiple thread, we can
trace multiple threads if we want to, by attaching the thread to a ptrace call, and we can not trace and as a result these threads won't be debugged at all.

signature:
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);

How does the ptrace communicate with other processes?
the ptrace signal to the process that it wants to attach it, by passing PTRACE_ATTACH:

ptrace(PTRACE_ATTACH, target_pid, 0, 0);

What happens is the following:
1. the kernel checks if the process exists, and if we have the permission to trace it.
2. The kernel set special flag, that marks the process as one that will be traced.
3. A signal sent to SIGSTOP from the kernel to the process to stop the process that we want to trace.
4. Meanwhile the tracer, calls a syscall waitpid(), and waits untill the kernel wakes it up(because the tracee needs to be stopped)
5. When the tracee finally stopped the kernel wakes up the tracer by returning the pid of the process we want to trace(tracee)
6. The kernel populates the status variable passed to waitpid() with the information that the tracee has stopped. This status often indicates that the process was stopped by a signal, typically reporting it as SIGTRAP.

At this point tracee is stopped and the tracer returned to execute with the info of pid and the SIGTRAP, and we are ready to do next operations.


PEEK VS POKE DATA:


Signatures:
long value = ptrace(PTRACE_PEEKDATA, target_pid, (void*)target_addr, NULL);

long data_to_write = 0xCC; // e.g., the INT 3 instruction for a breakpoint
ptrace(PTRACE_POKEDATA, target_pid, (void*)target_addr, (void*)data_to_write);

How does the PEEKDATA/POKEDATA works:
1. GDB Requests: GDB sends the PTRACE_PEEKDATA request, specifying the target process (PID) and the desired virtual address.

2. Kernel Check: The Kernel ensures the target process is stopped.

3. Address Translation: The Kernel uses the tracee's Page Tables to convert the virtual address into a physical RAM address.

4. Data Fetch: The Kernel reads one machine word (e.g., 8 bytes) directly from the corresponding physical RAM location.

5. Return Value: The Kernel sends this fetched data back to GDB as the return value of the ptrace system call.


Note: that if we want to pokedata the kernel must validate that we have the right permissions for this.


Single step:
The single step relies on the ptrace syscall and the CPU's Trap Flag (TF):

1. Setup
GDB Request: GDB calls ptrace with PTRACE_SINGLESTEP.

Kernel Action: The Kernel sets the Trap Flag (TF) in the tracee's CPU registers.

Resume: GDB tells the tracee to resume execution.

2. Execution & Halt
Execute One: The CPU executes one machine instruction.

Hardware Trap: Due to the active TF, the CPU immediately triggers a Trap Exception.

Kernel Stops: The Kernel handles the trap, clears the TF, stops the tracee, and sends a SIGTRAP signal.

3. Debugger Control
GDB Regains Control: GDB unblocks via waitpid() upon receiving the SIGTRAP.

Inspect State: GDB uses PTRACE_GETREGS to read the new Instruction Pointer (IP) and update the debug view.

