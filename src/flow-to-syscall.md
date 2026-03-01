# The flow to a syscall

## Native processes

We already saw a very high level graphic depicting the flow for syscalls, but now let's dive into a real example on the native side of things, since that's the most common in our world of implant dev.

## Allocating memory

Let's say that you want to allocate a page or so of memory for shellcode execution or the manual mapping of a COFF object, EXE image or DLL image. To do so, you'd have to call `VirtualAlloc` (staying inside the local process), the highest level API we can call, implemented in `kernelbase.dll`, but ultimately forwarded to `ntdll.dll`.

Let's check it out on the C-side.

```cpp
INT
__cdecl
main(VOID)
{
  // alloc a single page
  LPVOID pBuffer = VirtualAlloc(NULL, PAGE_SIZE, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
  // error check here
  // do shady things
  // cleanup, overwrite with garbage data
  VirtualFree(..);
  pBuffer = LPVOID();
  // go home
  return ERROR_SUCCESS;
}
```

It's a simple program that allocates one page of memory. Now we can follow this in WinDbg... the best debugger on the planet.

I launched a new executable under WinDbg and set a BP on `kernelbase!VirtualAlloc` using the command: `bp kernelbase!VirtualAlloc`

F5 the program and let the BP hit.

The BP will get hit and at that time you can see what calls it makes. We are looking for some `Nt` prefixed function in the output.

```text
0:000> uf /c kernelbase!VirtualAlloc
KERNELBASE!VirtualAlloc (00007ffc`b1bb18a0)
  KERNELBASE!VirtualAlloc+0x41 (00007ffc`b1bb18e1):
    // bingo!
    call to ntdll!NtAllocateVirtualMemory (00007ffc`b402d060)
  KERNELBASE!VirtualAlloc+0x5e (00007ffc`b1bb18fe):
    call to KERNELBASE!BaseSetLastNTError (00007ffc`b1b7b300)
  KERNELBASE!VirtualAlloc+0x4e30d (00007ffc`b1bffbad):
    call to ntdll!RtlSetLastWin32Error (00007ffc`b3fe0770)
```

Cool. Now we can jump into NTDLL and take a look from there.

From here, set a `BP` on that routine or just hit `TC` to trace to the first call instruction found.

Here is what things will look like at the `BP`.

```text
ntdll!NtAllocateVirtualMemory:
00007ffc`b402d060 4c8bd1           mov     r10, rcx. // <--- first param
00007ffc`b402d063 b818000000       mov     eax, 18h  // <--- syscall number
00007ffc`b402d068 f604250803fe7f01 test    byte ptr [7FFE0308h], 1
00007ffc`b402d070 7503             jne     ntdll!NtAllocateVirtualMemory+0x15 (7ffcb402d075)
00007ffc`b402d072 0f05             syscall 
00007ffc`b402d074 c3               ret     
00007ffc`b402d075 cd2e             int     2Eh
00007ffc`b402d077 c3               ret     
00007ffc`b402d078 0f1f840000000000 nop     dword ptr [rax+rax]
```

You can continue to single step up to the `syscall` if you'd like. You won't be able to jump with the `syscall` into the kernel just yet, but we will dive into that later and take this all the way home!

One thing I like to check out around this time is the call stack. Run `k` to show the call stack up to this point.

```text
0:000> k
 # Child-SP          RetAddr               Call Site
00 000000f4`8970fb18 00007ffc`b1bb18e8     ntdll!NtAllocateVirtualMemory
01 000000f4`8970fb20 00007ff7`63a91e84     KERNELBASE!VirtualAlloc+0x48
[..SNIP..]
05 000000f4`8970fb60 00007ff7`63a9408c     Shellcode!main+0x254
[..SNIP..]
07 000000f4`8970fc80 00007ffc`b3fe2651     KERNEL32!BaseThreadInitThunk+0x14
08 000000f4`8970fcb0 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

Cool. Now we can start to dive a bit deeper into things over the next several chapters.
