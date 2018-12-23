# Patch Guard Oops

### CRITICAL\_STRUCTURE\_CORRUPTION \(109\) 

This bugcheck is generated when the kernel detects that critical kernel code or data have been corrupted. There are generally three causes for a corruption: 

*  A driver has inadvertently or deliberately modified critical kernel code or data. See [http://www.microsoft.com/whdc/driver/kernel/64bitPatching.mspx](http://www.microsoft.com/whdc/driver/kernel/64bitPatching.mspx) 
* A developer attempted to set a normal kernel breakpoint using a kernel debugger that was not attached when the system was booted. Normal breakpoints, "bp", can only be set if the debugger is attached at boot time. Hardware breakpoints, "ba", can be set at any time. 
* A hardware corruption occurred, e.g. failing RAM holding kernel code or data. 

### Type of corrupted region

```c
	0   : A generic data region
	1   : Modification of a function or .pdata
	2   : A processor IDT
	3   : A processor GDT
	4   : Type 1 process list corruption
	5   : Type 2 process list corruption
	6   : Debug routine modification
	7   : Critical MSR modification
	8   : Object type
	9   : A processor IVT
	a   : Modification of a system service function
	b   : A generic session data region
	c   : Modification of a session function or .pdata
	d   : Modification of an import table
	e   : Modification of a session import table
	f   : Ps Win32 callout modification
	10  : Debug switch routine modification
	11  : IRP allocator modification
	12  : Driver call dispatcher modification
	13  : IRP completion dispatcher modification
	14  : IRP deallocator modification
	15  : A processor control register
	16  : Critical floating point control register modification
	17  : Local APIC modification
	18  : Kernel notification callout modification
	19  : Loaded module list modification
	1a  : Type 3 process list corruption
	1b  : Type 4 process list corruption
	1c  : Driver object corruption
	1d  : Executive callback object modification
	1e  : Modification of module padding
	1f  : Modification of a protected process
	20  : A generic data region
	21  : A page hash mismatch
	22  : A session page hash mismatch
	23  : Load config directory modification
	24  : Inverted function table modification
	25  : Session configuration modification
	26  : An extended processor control register
	27  : Type 1 pool corruption
	28  : Type 2 pool corruption
	29  : Type 3 pool corruption
	2a  : Type 4 pool corruption
	2b  : Modification of a function or .pdata
	2c  : Image integrity corruption
	2d  : Processor misconfiguration
	2e  : Type 5 process list corruption
	2f  : Process shadow corruption
	101 : General pool corruption
	102 : Modification of win32k.sys

```

