# The `SYSCALL` instruction

## Let's get stuck in

The absolute best place to understand what is really happening with this instruction is to look at the Intel manual for it. Let's take a look at this right from there:

> "SYSCALL invokes an OS system-call handler at privilege level 0. It does so by loading RIP from the IA32_LSTAR MSR (after saving the address of the instruction following SYSCALL into RCX). (The WRMSR instruction ensures that the IA32_LSTAR MSR always contain a canonical address.)"

## Things to note

OK, so we have some things to take away from that:

- **System-call handler**
- **IA32_LSTAR MSR**
- **Return address saved in RCX**

Let's keep going:

> "SYSCALL also saves RFLAGS into R11 and then masks RFLAGS using the IA32_FMASK MSR (MSR address C0000084H); specifically, the processor clears in RFLAGS every bit corresponding to a bit that is set in the IA32_FMASK MSR. SYSCALL loads the CS and SS selectors ..."

OK, that isn't entirely useful anymore. We have what we need to move on with this.

## System-call handler

From the notes above, we need to know what this routine is if we want to dive into this any deeper. To find out what the handler is for **IA32_LSTAR MSR**, we need to be in a kernel debugger session. We then need to use a **CPL0**-only instruction to read from the **MSR**: **`rdmsr`**. In a kernel driver, the equivalent would be a compiler intrinsic **`__readmsr`**.

```text
0: kd> rdmsr C0000082H
msr[c0000082] = fffff804`74a1a2c0
```

We now have the address of the system-call handler, but what is it really? Let's find out back in WinDbg. To do this, we want to use the **`ln`** command to list the nearest symbol at that address.

```text
0: kd> ln fffff804`74a1a2c0
Browse module
Set bu breakpoint

(fffff804`74a1a2c0)   nt!KiSystemCall64   |  (fffff804`74a1a500)   nt!KiSystemServiceUser
Exact matches:
```

For this remote system, VBS is not enabled and as such, the handler is **`nt!KiSystemCall64`**.
