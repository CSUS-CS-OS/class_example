# Page Fault Exception Handling

## Overview
This example demonstrates how to handle page faults - one of the most important exceptions in operating systems. When a program tries to access memory that hasn't been allocated or is not currently in physical RAM, the CPU generates a page fault exception. The OS must then decide whether to allocate the page, load it from disk, or terminate the program.

## Learning Objectives
- Understand page faults and virtual memory
- Learn how to register page fault exception handlers
- See the page fault exception (exception 14) flow
- Understand when page faults occur and how to handle them
- See the role of paging in memory management

## Build and Run
```bash
make
spede-target   # Separate terminal
spede-run -d
```

## What is a Page Fault?

A page fault occurs when:
1. CPU tries to access a virtual memory address
2. The memory management unit (MMU) looks up the address in the page table
3. The page table entry indicates:
   - Page is not in physical memory
   - Page is not present at all
   - Access permissions are violated (e.g., write to read-only page)

When any of these conditions occur, the CPU:
1. Stops executing user code
2. Pushes the fault address into the CR2 register
3. Generates exception 14 (Page Fault)
4. Jumps to the page fault handler

## Page Fault Exception (Exception 14)

Like other exceptions:
- Hardware pushes EFLAGS, CS, EIP
- CPU looks up IDT[14]
- Jumps to page fault handler
- Handler receives the faulting address in CR2 register

## Error Code on Page Fault

The processor also pushes an error code with page fault:
```text
Bit 0: Present (P) - 1 if page table entry is present
Bit 1: Write (W) - 1 if fault was on write, 0 if on read  
Bit 2: User (U) - 1 if fault occurred in user mode
Bits 3+: Reserved
```

Example:
- Error 0 (binary 000): Page not present, read access, kernel mode
- Error 6 (binary 110): Page not present, write access, user mode

## Handling Strategies

### 1. Demand Paging
Only allocate pages when accessed:
- Page fault occurs
- Handler allocates a physical page
- Fills the page table entry
- User code resumes

### 2. Copy-on-Write
Pages are shared until modified:
- Two processes share same physical page
- One process writes
- Page fault handler
- Allocate new page for writing process
- Now each has their own page

### 3. Memory-Mapped Files
- File mapped to virtual address space
- Initial pages not in memory
- Page fault handler
- Loads page from disk file
- Process sees file as memory

### 4. Stack Growth
- Stack allocated as needed
- Access below stack causes fault
- Handler allocates new stack pages
- Common in recursive functions

## What This Example Does

The example:
1. Registers a page fault handler in IDT[14]
2. Intentionally accesses invalid memory
3. Page fault handler catches the exception
4. Prints diagnostic information
5. Could recover or terminate

## Accessing the Faulting Address

The page fault handler can read the address that caused the fault:

```c
void page_fault_handler() {
    unsigned int faulting_address = read_CR2();
    printf("Page fault at address: 0x%x\n", faulting_address);
}
```

CR2 (Control Register 2) contains the virtual address that triggered the fault.

## Page Table Entry Format

On x86 (4KB pages):
```text
31-12: Physical page number (PPN)
   11: Available for OS
   10: Available for OS
    9: Available for OS  
    8: Global
    7: Page size (0 for 4KB)
    6: Dirty (written)
    5: Accessed (read or written)
    4: Cache disabled
    3: Write-through
    2: User/Supervisor
    1: Read/Write permission
    0: Present (1 = present, 0 = not present)
```

When bit 0 (Present) = 0, accessing the page causes a page fault.

## Paging Architecture

### Virtual to Physical Translation
```text
Virtual Address
    |
    v
+--------+
| CR3    | (Points to Page Directory)
+--------+
    |
    v
Page Directory (4MB each)
    |
    v
Page Table (4KB each)
    |
    v
Physical Page (4KB)
```

### 32-bit Virtual Address Breakdown
```text
|Dir (10 bits)|Table (10 bits)|Offset (12 bits)|
10-bit index  10-bit index    Byte offset
   into PD        into PT        in page
```

## Progression
- **Before**: `4.KernelProcess` - Process management
- **Next**: `6.MemPolicy` - Memory allocation policies
- **Next**: `7.Syscall` - System calls

## Related Concepts
- Virtual memory
- Memory translation and TLB (Translation Lookaside Buffer)
- Page replacement algorithms
- Segmentation faults and memory protection
- Swap space on disk

## Why Page Faults Matter

Page faults are fundamental to:
- **Virtual memory**: Programs see more memory than physically available
- **Memory protection**: Invalid access is caught and handled
- **Efficient allocation**: Memory allocated only when needed
- **Multitasking**: Each process has isolated address space
- **Dynamic linking**: Libraries loaded on demand

## Real-World Examples

### Stack Overflow Detection
- Stack grows downward  
- Guard page below stack not present
- Stack grows too much
- Access guard page
- Page fault handler detects stack overflow

### Lazy Process Initialization
- Process forked but pages not copied
- Accesses page
- Copy-on-write page fault
- Page allocated and copied

### Memory-Mapped Database
- Large file mapped to address space
- Only working set in memory
- Access causes page fault
- Handler loads from file
- OS manages disk I/O transparently

## GDB Debugging Page Faults

```bash
spede-run -d
(gdb) b page_fault_handler
(gdb) b exception_14
(gdb) print read_CR2()    # Get faulting address
(gdb) print error_code    # Get error code
```

---

Page faults are one of the most important exceptions and a key part of modern virtual memory systems!
