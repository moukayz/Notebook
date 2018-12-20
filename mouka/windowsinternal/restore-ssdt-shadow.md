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
tableRva = ntTableBase - ntBase; // --> 0x81300
// table offset to file base in ntoskrnl.exe file
tableOffset = RvaToOffset(tableRva);  // --> 0x80900
```

Then  **read** the kernel executable to memory \(just like we read ntdll.dll in [SSDT HOOK](ssdt-hook.md#get-ssdt-index-from-ntdll-dll)\) to get original SSDT base in file \( the default path is "c:\windows\system32\ntoskrnl.exe", but to double-check, we get its image path from kernel module information \)

```c
// Get pSystemInfoBuffer by ZwQuerySystemInformation routine
PUCHAR ntFilename = pSystemInfoBuffer->Module[0].FullPathName;
PVOID pNtFileBase = ReadFile(ntFilename); 
ULONG_PTR ssdtFileBase = pNtFileBase + tableOffset;
```

and analyze its PE header to get its **ImageBase** field.

```c
PIMAGE_DOS_HEADER pDosHeader = DOS_HEADER( pNtFileBase );
PIMAGE_NT_HEADERS pNtHeaders = NT_HEADERS( pNtFileBase );

ULONG_PTR ntDefaultBase = pNtHeaders->OptionalHeader.ImageBase;
// --> 0x140000000
```

Then read the first bytes of the SSDT in ntoskrnl.exe file buffer. \( For simplicity, I read them in a binary file editor which is the same as in a program\)

```c
// bytes begin at address of ssdtFileBase
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

$$
0040d9a0 = 000000014048ECA0 - 0000000140000000 - 81300
$$

**which is** 

$$
originalOffset = originalBytes  - (ntDefaultBase + tableRva)
$$

So we can conclude that the **originalBytes** in nt file is actually the default address of its corresponding kernel function .

Now we can calculate the original SSDT value of any SSDT function with its index even if SSDT has been modified in runtime.

```c
// Get default function address with index
defaultAddress = (PULONGULONG)ssdtFileBase + SSDTIndex;  
originalOffset = defaultAddress - (ntDefaultBase + tableRva); 
originalValue =  originalOffset << 4;
```

#### Combine together

So the steps to get the original value of any SSDT function is:

1. Get kernel base address `ntBase` and SSDT base address `ssdtBase` \( like in  [Hook SSDT\(Shadow\)](ssdt-hook.md#find-the-base-address-of-ntoskrnl-exe-and-win-32-k-sys)\), 
2. Calculate the RVA between them: `tableRva`
3. Read ntoskrnl.exe in memory and get its default image base address `ntDefaultBase` and original SSDT buffer `ssdtFileBase`
4. For any function we can calculate its default SSDT value with its SSDT index ,which is `((PULONG_PTR)ssdtFileBase + Index-ntDefaultBase-tableRva) << 4`

### x86

In X86 platform the thing is very similar.

First we need to get the kernel base and SSDT base , as shown in Windbg:

```cpp
0: kd> lmDmnt
Browse full module list
start    end        module name
83e4b000 8425d000   nt         (pdb symbols)          C:\Program Files (x86)\Windows Kits\10\Debuggers\x86\sym\ntkrpamp.pdb\C820DD65C4BC4499A56D7610BE16FD082\ntkrpamp.pdb
0: kd> dps nt!KeServiceDescriptorTable
83fb4b00  83ec9d9c nt!KiServiceTable
83fb4b04  00000000
83fb4b08  00000191
83fb4b0c  83eca3e4 nt!KiArgumentTable
```

we have

```c
ntBase = 0x83e4b000 ;
ntTableBase = 0x83ec9d9c ;
// So the offset is  
tableRva = ntTableBase - ntBase; // --> 0x7ed9c
// table offset to file base in ntoskrnl.exe file
tableOffset = RvaToOffset(tableRva);  // --> 0x7e59c
```

Then read the nt file into memory\( **note that we need to get the right file path from `SystemModuleInformation` by `ZwQuerySystemInformation`\)**

```c
// Get pSystemInfoBuffer by ZwQuerySystemInformation routine
PUCHAR ntFilename = pSystemInfoBuffer->Module[0].FullPathName;
PVOID pNtFileBase = ReadFile(ntFilename); 
ULONG_PTR ssdtFileBase = pNtFileBase + tableOffset;
```

and analyze its PE header to get its **ImageBase** field.

```c
PIMAGE_DOS_HEADER pDosHeader = DOS_HEADER( pNtFileBase );
PIMAGE_NT_HEADERS pNtHeaders = NT_HEADERS( pNtFileBase );

ULONG_PTR ntDefaultBase = pNtHeaders->OptionalHeader.ImageBase;
// --> 0x400000
```

The first few bytes at `ssdtFileBase` are : \( Opened in binary editor\)

```c
30 9C 67 00 0D 14 4C 00 6E 9B 60 00 8A 58 42 00
07 B5 67 00 96 E3 4F 00 A1 BB 6E 00 EA BB 6E 00
```

and the first few bytes at real SSDT in kernel memory are：

```c
0: kd> dd /c1 nt!KiServiceTable l5
83ec9d9c  840c4c30
83ec9da0  83f0c40d
83ec9da4  84054b6e
83ec9da8  83e7088a
83ec9dac  840c6507
```

So we have

`840c4c30 = 00679c30 - 00400000 + 83e4b000`

which is

$$
originalValue = defaultAddress - ntDefaultBase + ntBase
$$

Then we can calculate the original SSDT value of any SSDT function with its index.

```c
// Get default function address with index
defaultAddress = (PULONG)ssdtFileBase + SSDTIndex;  
originalValue = defaultAddress - (ntDefaultBase + tableRva); 
```

## Unhook Shadow

Unhooking Shadow SSDT is very similar to unhooking SSDT. The steps are almost the same.

### x64

1. Find win32k base address `w32kBase` and Shadow SSDT base address `w32kTable` \( look [Find SSDT\(Shadow\) base address](ssdt-hook.md#find-ssdt-shadow-base-address) and Find [win32k.sys base address](ssdt-hook.md#find-the-base-address-of-ntoskrnl-exe-and-win-32-k-sys)\)
2. Calculate the RVA between them: `w32kTableRva`
3. Read win32k.sys into memory and get its default image base `w32kDefaultBase` and original Shadow SSDT `w32kTableFileBase`
4. For any function in Shadow SSDT, we can calculate its original value by its index, which is `((PULONG_PTR)w32kTableFileBase + Index - w32kDefaultBase - w32kTableRva) << 4`

### x86

1. ...
2. ...
3. ...
4. The original value in Shadow SSDT is `(PULONG_PTR)w32kTableFileBase + Index - w32kDefaultBase - w32kTableRva`

