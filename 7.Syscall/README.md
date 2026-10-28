# System Calls (Syscall)

## Overview
This module demonstrates how system calls work. A system call is the primary mechanism by which user programs request services from the kernel (like I/O, memory allocation, etc.). The module shows how to:
- Transition from user mode to kernel mode
- Define and implement system call handlers
- Use INT instruction to invoke system calls
- Return from system calls with proper privilege restoration

## Learning Objectives
- Understand system calls as the user-kernel interface
- Learn how to implement system call dispatch
- See how privilege levels are managed
- Understand mode transitions (user ↔ kernel)
- Practice mixing user and kernel code
- Learn about system call conventions (how parameters are passed)

## Build and Run
```bash
make
spede-target   # Separate terminal
spede-run -d
```

## What is a System Call?
A system call is a request from a user program to the kernel:
- User code cannot directly access hardware
- User code cannot directly modify other processes' memory
- User code needs kernel to provide these services
- System calls are the controlled interface

### Why System Calls?
1. **Protection**: Kernel validates all requests
2. **Isolation**: Prevents processes from interfering
3. **Resource management**: Kernel allocates and tracks resources
4. **Security**: Kernel enforces access control

## How System Calls Work

### Traditional Approach (Intel)
1. User program loads parameters into registers
2. User program loads system call number into EAX
3. User program executes `INT 0x80` instruction
4. Hardware generates interrupt
5. CPU switches to kernel mode
6. Kernel looks up system call handler by number
7. Kernel executes the handler
8. Kernel stores return value in EAX
9. Kernel executes IRET to return to user code
10. CPU switches back to user mode

### Stack Layout During System Call
```text
Before INT:
  EAX = system call number
  EBX, ECX, EDX, ESI, EDI = parameters
  ESP = user stack pointer

After INT (kernel handler):
  Stack has: EFLAGS, CS, EIP (pushed by hardware)
  ESP = kernel stack pointer (switched automatically)
  
After IRET:
  Back to user mode
  EAX = return value
  User stack restored
```

## System Call Numbers
Common system calls (Linux on x86):
- 1: exit
- 3: read
- 4: write
- 11: execve
- 33: dup2
- 37: kill
- 45: brk (set heap limit)
- 54: ioctl

In this example, custom system call numbers are defined for teaching purposes.

## Implementation

### System Call Handler
```c
void syscall_handler(unsigned int syscall_num, unsigned int param1,
                     unsigned int param2, unsigned int param3) {
   switch(syscall_num) {
        case 1: // exit
            terminate_process();
            break;
        case 2: // read
            return read_from_device(param1, param2, param3);
            break; 
        // ... more handlers
   }
}
```

### INT Handler Setup
```c
// Register syscall handler
fill_gate(&IDT_p[0x80], (int)syscall_entry, get_cs(), ACC_INTR_GATE, 3);  // Privilege 3 allows user code
```
*Note: Most interrupts are privilege 0 (kernel only). System calls are usually privilege 3 (user can execute).*

## System Call ABI (Application Binary Interface)

### Parameter Passing (Typical Convention)
```text
Parameter 1: EBX
Parameter 2: ECX
Parameter 3: EDX
Parameter 4: ESI
Parameter 5: EDI
Parameter 6: EBP
Return value: EAX
Error code: Also in EAX (negative indicates error)
```

## User vs. Kernel Mode

### User Mode (Privilege Level 3)
- Cannot execute privileged instructions (CLI, STI, LGDT, etc.)
- Cannot access kernel memory
- Cannot directly access I/O ports
- Cannot modify page tables
- Cannot change privilege level
- Must use system calls to request services

### Kernel Mode (Privilege Level 0)
- Can execute all instructions
- Can access all memory
- Can access I/O ports
- Can modify page tables
- Can change privilege level
- Provides services to user mode

## Transitions Between Modes

### User → Kernel
- INT instruction (system call)
- Interrupt or exception occurrence
- Hardware automatically:
  - Saves user state (CS:EIP, EFLAGS)
  - Switches to kernel stack
  - Switches to kernel mode (CPL=0)
  - Jumps to handler

### Kernel → User
- IRET instruction
- Hardware automatically:
  - Restores user state from stack
  - Switches to user stack
  - Switches to user mode (CPL=3)
  - Jumps to user code

## System Call Optimization

### Fast System Calls (SYSENTER/SYSEXIT)
Modern x86 provides faster system call mechanism:
- Traditional INT: ~1000+ CPU cycles
- SYSENTER/SYSEXIT: ~100 CPU cycles
- Used by Linux, Windows
- Requires special register setup

## Common System Calls on Real Systems

**File I/O:**
- open, close, read, write, seek, stat

**Process Management:**
- fork, exec, exit, wait, kill, getpid

**Memory Management:**
- brk, mmap, munmap, mprotect

**Signal Handling:**
- signal, sigaction, sigprocmask

**IPC:**
- pipe, socket, sendto, recvfrom, select

## Error Handling

System calls typically return:
- Negative values for errors (e.g., -1 means error)
- Positive or 0 for success
- Error codes stored separately (errno in C libraries)

## GDB Debugging System Calls

```bash
spede-run -d
(gdb) b syscall_entry       # Breakpoint at entry
(gdb) b syscall_handler     # Breakpoint at C handler
(gdb) print $eax            # System call number
(gdb) print $ebx/$ecx/$edx  # Parameters
```

## Progression
- **Before**: `6.MemPolicy` - Memory management
- **Next**: `8.UserProcess` - User-mode processes
- **After**: Device drivers and advanced I/O

## Why System Calls Matter

System calls are:
- **The fundamental OS interface**: Every useful program needs them
- **Security boundary**: Where OS validates requests
- **Performance critical**: Heavily optimized in real systems
- **Portable**: POSIX standardizes many system calls
- **Complex**: Requires careful privilege management

## Advanced Topics

### Vdso (Virtual Dynamic Shared Object)
- Some system calls implemented in user space
- No mode switch overhead
- Examples: gettimeofday, getcpuclockid

### Seccomp (Secure Computing Mode)
- Restrict available system calls
- Used for sandboxing
- Filter syscalls based on security policy

### System Call Tracing
- `strace` tool shows all system calls
- Useful for debugging
- Performance analysis

---
System calls are the bridge between user programs and the OS kernel. Understanding them is essential for systems programming!
