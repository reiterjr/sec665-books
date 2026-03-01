# KiSystemServiceRepeat

## The purpose

At some point, the code flow will make its way down to **`KiSystemServiceRepeat`** after the trap frame has been established in **`KiSystemServiceStart`**. One of the first things `KiSystemServiceRepeat` does is grab pointers to 2 of 3 arrays that are used to keep track of system service tables. The 3 arrays are **`KeServiceDescriptorTable`**, **`KeServiceDescriptorTableShadow`**, and **`KeServiceDescriptorTableFilter`**. Inside each of the tables, there will be a couple of entries where each entry will hold some useful information like the following:

- A pointer to the array of system calls that are implemented in that table
- How many system calls are present in that table
- A pointer to an array of byte arguments for each of the system calls in that table

Like most things, this data is held within a structure called **`_SYSTEM_SERVICE_TABLE`**.

```cpp
typedef struct _SYSTEM_SERVICE_TABLE {
    PULONG      ServiceTable;   // pointer to array of system calls in this table
    PULONG_PTR  CounterTable;
    ULONG_PTR   ServiceLimit;   // how many system calls are present in this table
    PBYTE       ArgumentTable;  // pointer to array of byte arguments for each system call
} SYSTEM_SERVICE_TABLE;
```

## Argument table

The argument table is very important. Its purpose is to tell the kernel how many arguments it needs to find on the user-mode stack. It will then take those arguments and bring them into the kernel's stack.

## In the debugger

In the kernel debugger, you can easily dump the table and see the pointers as well as some of the other data from the structs. First off, grab the address of the table: **`x nt!KiServiceTable`**. You can use that symbol and index your way into the table to find things like arguments for a syscall. Like so:

```text
dx (((int*)&(nt!KiServiceTable))[1] & 0xf)
(((int*)&(nt!KiServiceTable))[1] & 0xf) : 0

dx (((int*)&(nt!KiServiceTable))[2] & 0xf)
(((int*)&(nt!KiServiceTable))[2] & 0xf) : 2  // 2 stack args
```

It's quite a manual process doing that especially if there are hundreds of syscalls in that table. It would be much better to use the true power of WinDbg and its debugger data model like so:

```text
// make a pseudo variable
dx @$table = &nt!KiServiceTable
@$table = &nt!KiServiceTable : 0xfffff8012dee4eb0 [Type: void *]
// dump the table and shift right by 4, then add to base of the table
dx (((int(*)[90000])&(nt!KiServiceTable)))->Take(*(int*)&nt!KiServiceLimit)->Select(x => (x >> 4) + @$table)
    [0]              : 0xfffff8012e14b650 [Type: void *]
    [1]              : 0xfffff8012e155ce0 [Type: void *]
    [2]              : 0xfffff8012e506e10 [Type: void *]
    [3]              : 0xfffff8012e6f1640 [Type: void *]
    [4]              : 0xfffff8012e435a40 [Type: void *]
    [5]              : 0xfffff8012e218710 [Type: void *]
    [6]              : 0xfffff8012e419f60 [Type: void *]
```

The above output is cool and all but we can do better. The pointers can be resolved and we can see what lies behind them; the syscalls!

Let's again leverage the number of syscalls in the table (`ServiceLimit`) and dump everything, but then resolve the symbolic names.

```text
dx (((int(*)[90000])&(nt!KiServiceTable)))->Take(*(int*)&nt!KiServiceLimit)->Select(x => @$dumpit((x >> 4) + @$table))
    [0]              : nt!NtAccessCheck (fffff801`2e14b650)
    [1]              : nt!NtWorkerFactoryWorkerReady (fffff801`2e155ce0)
    [2]              : nt!NtAcceptConnectPort (fffff801`2e506e10)
    [3]              : nt!NtMapUserPhysicalPagesScatter (fffff801`2e6f1640)
    [4]              : nt!NtWaitForSingleObject (fffff801`2e435a40)
    [5]              : nt!NtCallbackReturn (fffff801`2e218710)
    [6]              : nt!NtReadFile (fffff801`2e419f60)
```

Reminder: this is only the table for service syscalls that come from `ntdll.dll` and **NOT** from `win32u.dll` into `win32k.sys`.

I leave that as an exercise to you all!

## Checking the syscall index

One of the next actions `KiSystemServiceRepeat` does is check to see if the syscall index is beyond this table. Remember, every table has a `ServiceLimit` that indicates how many syscalls there are. Let's see how it computes this in code.

```asm
cmp eax, [r10+rdi+10h]
```

`EAX` holds the syscall index. `R10` holds the table. `RDI` holds the table ID which could be one of four values from 00b to 11b. Obviously 10h is just 16 in decimal. So, the check here is accessing an offset into the `KeServiceDescriptorTable` to find the `ServiceLimit`.

```text
dps nt!KeServiceDescriptorTable
fffff801`2ec1e8c0  fffff801`2dee4eb0 nt!KiServiceTable
fffff801`2ec1e8c8  00000000`00000000
fffff801`2ec1e8d0  00000000`000001d7  // this is the ServiceLimit
fffff801`2ec1e8d8  fffff801`2dee5610 nt!KiArgumentTable
```

This can be validated because the `ServiceLimit` is a global symbol so check it out.

```text
dc nt!KiServiceLimit l1
fffff801`2dee560c  000001d7  // it's a match!!
```

After this check, a jump will be taken if it's out of range, meaning it is not in this table. Then when it jumps, the thread will be converted to a GUI thread by calling **`KiConvertToGuiThread`**. This is what that code block looks like:

```asm
mov     [rbp-80h], eax
mov     [rbp-78h], rcx
mov     [rbp-70h], rdx
mov     [rbp-68h], r8
mov     [rbp-60h], r9
call    KiConvertToGuiThread
or      eax, eax
mov     eax, [rbp-80h]
mov     rcx, [rbp-78h]
mov     rdx, [rbp-70h]
mov     r8, [rbp-68h]
mov     r9, [rbp-60h]
mov     [rbx+90h], rsp
jz      KiSystemServiceRepeat   // start the process all over after the conversion
```

Eventually, the jump will not be taken and we will wind up in this code block:

```asm
// KeServiceDescriptorTable + tableID (00h or 20h) = nt!KiServiceTable
mov     r10, [r10+rdi]
// movsxd will move a 32-bit number into 64-bit register keeping the sign
// so negative number will stay negative
// nt!KiServiceTable + syscallIndex * 4
// syscall index 23h - NtQueryVirtualMemory
// dd /c1 nt!KiServiceTable + 23h * 4 l1 = fffff801`14cdcf3c  0557bd02 <-- RVA!
movsxd  r11, dword ptr [r10+rax*4]
// RAX - RVA of syscall - 0557bd02
```

This RVA is interesting. The first byte holds the number of stack args. In this instance, it says 2. So, the kernel will be copying 2 values from the user mode stack. This is why the RVA is shifted right by 4 bits, to skip over this value.

```asm
mov     rax, r11
// RVA >> 4 = 557bd0
sar     r11, 4
// nt!KiServiceTable + RVA = fffff801`15234a80
// ln fffff801`15234a80
// (fffff801`15234a80) nt!NtQueryVirtualMemory <-- found it!!
add     r10, r11
cmp     edi, 20h ; ' '
jnz     short loc_140408ED0
```

After a few more checks are done, the syscall is eventually invoked using an indirect call like this:

```asm
mov rax, r10
call rax
```

At this time, all the proper registers RCX, RDX, R8, R9 have been populated with the proper syscall args, and the user mode stack was copied to the kernel stack.
