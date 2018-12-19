# Restore SSDT\(Shadow\)

**If the SSDT\(Shadow\) was hooked by bad guys, how can we restore it to the original state?** 😟 

## Unhook SSDT

### x64

To unhook SSDT, we must obtain the original value of the SSDT function offset which has been replaced. Well it's a little trick to do this. 

First, we have to calculate the offset from SSDT base address to kernel base address, for example in WinDbg \( Machine OS is Win7 x64\):

```text
1: kd> lmDmnt
Browse full module list
start             end                 module name
fffff800`03e50000 fffff800`0443a000   nt         (pdb symbols)          C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\sym\ntkrnlmp.pdb\3844DBB920174967BE7AA4A2C20430FA2\ntkrnlmp.pdb

1: kd> dps nt!KeServiceDescriptorTable
fffff800`04101840  fffff800`03ed1300 nt!KiServiceTable
fffff800`04101848  00000000`00000000
fffff800`04101850  00000000`00000191
fffff800`04101858  fffff800`03ed1f8c nt!KiArgumentTable
```

We have 

```c
ntBase = 0xfffff80003e19000;
ntTableBase = 0xfffff80003e9a300;
// So the offset is  
tableRva = ntTableBase - ntTableBase; // --> 0x81300
// table offset in ntoskrnl.exe file
tableOffset = RvaToOffset(tableRva);  // --> 0x80300
```

Then  **read** the kernel executable \( c:\windows\system32\ntoskrnl.exe\) to memory \(just like we read ntdll.dll in [SSDT HOOK](ssdt-hook.md#get-ssdt-index-from-ntdll-dll)\) to get original SSDT base 

```c
PVOID pNtFileBase = ReadFile(ntfilename); 
ULONG_PTR ssdtFileBase = pNtUserBase + tableRva;
```

and analyze its PE header to get its **ImageBase** field.

```c
PIMAGE_DOS_HEADER pDosHeader = DOS_HEADER( pNtUserBase );
PIMAGE_NT_HEADERS pNtHeaders = NT_HEADERS( pNtUserBase );

ULONG_PTR ntDefaultBase = pNtHeaders->OptionalHeader.ImageBase;
// --> 0x140000000
```

Then read the first bytes of the SSDT in ntoskrnl.exe file buffer. \( Instead I read them in a binary file editor which is the same as in a program\)

```c
A0 EC 48 40 01 00 00 00 C0 68 37 40 01 00 00 00
A0 81 07 40 01 00 00 00 80 9A 36 40 01 00 00 00
A0 B7 39 40 01 00 00 00 A0 29 39 40 01 00 00 00
```

and compare them with the real SSDT contents in kernel memory. \( Shows in windbg\)

```text
0: kd> dd /c 1 nt!KiServiceTable l5
fffff800`03ed1300  040d9a00
fffff800`03ed1304  02f55c00
fffff800`03ed1308  fff6ea00
fffff800`03ed130c  02e87805
fffff800`03ed1310  031a4a06
```

as mentioned in [SSDT HOOK](ssdt-hook.md#combine-index-with-ssdt-shadow), the real kernel function address equals value &gt;&gt; 4 plus table base address.

for example the first SSDT function offset is 

```c
ULONGLONG originalOffset = 0x0040d9a0; //  0x040d9a00 >> 4
```

and the first 8 bytes in  SSDT in ntoskrnl.exe file buffer showed above are

```c
ULONGLONG originalBytes = 0x000000014048ECA0
```

 **Here comes the climax of the play.**

```cpp
0x0040d9a0 = 0x000000014048ECA0 - 0x0000000140000000 - 0x81300;
// which is
originalOffset = originalBytes  - (ntDefaultBase + tableRva);
// which is
originalOffset = originalBytes - ssdtDefaultBase
```

So we can conclude that the **originalBytes** in ntos file is actually the default address of its corresponding kernel function .

Now we can calculate the original SSDT value of any SSDT function with its index even if SSDT has been modified in runtime.

```c
// Get default function address with index
defaultAddress = (PULONGULONG)ssdtFileBase + SSDTIndex;  
originalOffset = defaultAddress - (ntDefaultBase + tableRva); 
originalValue =  originalOffset << 4;
```

#### Combine together

So the steps to get the original value of any SSDT function is:

1. Get kernel base address and SSDT base address \( like in  [Hook SSDT\(Shadow\)](ssdt-hook.md#find-the-base-address-of-ntoskrnl-exe-and-win-32-k-sys)\), 
2. Then calculate the RVA between them
3. Read ntoskrnl.exe in memory and get its default image base address and original SSDT buffer
4. Finally for any function we can calculate its default SSDT value with its SSDT index.

