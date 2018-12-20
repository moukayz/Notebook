# Hook SSDT\(Shadow\)

\[TOC\]

## Basic struct

```c
struct SSDTStruct
{
    LONG* pServiceTable;
    PVOID pCounterTable;
#ifdef _WIN64
    ULONGLONG NumberOfServices;
#else
    ULONG NumberOfServices;
#endif
    PCHAR pArgumentTable;
};
```

## Find SSDT\(Shadow\) base address

### x86

**SSDT:**

```text
1: kd> dps nt!KeServiceDescriptorTable
83f86b00  83e7ccbc nt!KiServiceTable
83f86b04  00000000
83f86b08  00000191
83f86b0c  83e7d304 nt!KiArgumentTable

1: kd> dd /c 1 nt!KiServiceTable l2
83e7ccbc  84098ffa    # Function offset
83e7ccc0  83ed5217
```

**SSDT Shadow:**

```text
1: kd> dps nt!KeServiceDescriptorTableShadow
83f86b40  83e7ccbc nt!KiServiceTable    # SSDT 
83f86b44  00000000
83f86b48  00000191
83f86b4c  83e7d304 nt!KiArgumentTable
83f86b50  9be58000 win32k!W32pServiceTable    # SSDT Shadow
83f86b54  00000000
83f86b58  00000339
83f86b5c  9be5902c win32k!W32pArgumentTable

1: kd> dd /c 1 win32k!W32pServiceTable l2
9be58000  9bde19df
9be58004  9bdfa31b
```

**Before dump SSDT Shadow, you have to attach to any user-mode GUI process \(like winlogon.exe\)**

```text
1: kd> !process 0 0 winlogon.exe
PROCESS 882d3d20  SessionId: 1  Cid: 01d4    Peb: 7ffdc000  ParentCid: 0188
    DirBase: bf153080  ObjectTable: 9b33f300  HandleCount:  55.
    Image: winlogon.exe

1: kd> .process /p 882d3d20  
Implicit process is now 882d3d20
.cache forcedecodeuser done
```

**Function Index to real function address:**

```c
realAddress = (PVOID)ntTable[FunctionIndex]
```

**Find table address in a driver**

Because **nt!KeServiceDescriptorTable** is exported by kernel, we can get its address in the program

```c
UNICODE_STRING routineName;
RtlInitUnicodeString(&routineName, L"KeServiceDescriptorTable");
pServiceDescriptorTable = (SSDTStruct*)MmGetSystemRoutineAddress(&routineName);
```

Then we can get the address of **nt!KeServiceDescriptorTableShadow** by add the offset \(like previously windbg shows\)

```c
pServiceDescriptorTableShadow = (PVOID)((ULONG_PTR)pServiceDescriptorTable + 0x40)
```

Then Get the addresses of two tables

```c
pNtTable = pServiceDescriptorTable;
pW32kTable = (PVOID)((ULONG_PTR)pServiceDescriptorTable + 0x10)
```

### x64

**SSDT:**

```text
1: kd> dps nt!KeServiceDescriptorTable
fffff800`040c7840  fffff800`03e97300 nt!KiServiceTable    # SSDT base address
fffff800`040c7848  00000000`00000000
fffff800`040c7850  00000000`00000191
fffff800`040c7858  fffff800`03e97f8c nt!KiArgumentTable

1: kd> dd /c 1 nt!KiServiceTable l10
fffff800`03e97300  040d9a00    # Function offset
fffff800`03e97304  02f55c00
fffff800`03e97308  fff6ea00
fffff800`03e9730c  02e87805
fffff800`03e97310  031a4a06
fffff800`03e97314  03116a05
fffff800`03e97318  02bb9901
...
```

**SSDT Shadow:**

```text
1: kd> dps nt!KeServiceDescriptorTableShadow
fffff800`040c7880  fffff800`03e97300 nt!KiServiceTable        # SSDT base address
fffff800`040c7888  00000000`00000000
fffff800`040c7890  00000000`00000191
fffff800`040c7898  fffff800`03e97f8c nt!KiArgumentTable
fffff800`040c78a0  fffff960`00111f00 win32k!W32pServiceTable    # SSDT Shadow base address
fffff800`040c78a8  00000000`00000000
fffff800`040c78b0  00000000`0000033b
fffff800`040c78b8  fffff960`00113c1c win32k!W32pArgumentTable

1: kd> dd /c 1 win32k!W32pServiceTable l10
fffff960`00111f00  fff3a740        # GDI Function offset
fffff960`00111f04  fff0b501
fffff960`00111f08  000206c0
fffff960`00111f0c  001021c0
...
```

**Before dump SSDT Shadow, you have to attach to any user-mode GUI process \(like winlogon.exe\)**

```text
1: kd> !process 0 0 winlogon.exe
PROCESS fffffa80042a8b30
    SessionId: 1  Cid: 01ec    Peb: 7fffffde000  ParentCid: 0198
    DirBase: 1bf29000  ObjectTable: fffff8a000a01580  HandleCount: 111.
    Image: winlogon.exe

1: kd> .process /p fffffa80042a8b30
Implicit process is now fffffa80`042a8b30
.cache forcedecodeuser done

1: kd> .reload
```

**Function Index to real function address:**

```c
readAddress = (ULONG_PTR)(ntTable[FunctionIndex] >> 4) + SSDT(Shadow)BaseAddress;
```

**Find table address in a driver**

Find **nt!KeServiceDescriptorTable** and **nt!KeServiceDescriptorTableShadow** in kernel function **nt!KiSystemServiceStart**

```text
0: kd> u nt!KiSystemServiceStart
nt!KiSystemServiceStart:
fffff800`03e9575e 4889a3d8010000  mov     qword ptr [rbx+1D8h],rsp
fffff800`03e95765 8bf8            mov     edi,eax
fffff800`03e95767 c1ef07          shr     edi,7
fffff800`03e9576a 83e720          and     edi,20h
fffff800`03e9576d 25ff0f0000      and     eax,0FFFh
nt!KiSystemServiceRepeat:
fffff800`03e95772 4c8d15c7202300  lea     r10,[nt!KeServiceDescriptorTable (fffff800`040c7840)]
fffff800`03e95779 4c8d1d00212300  lea     r11,[nt!KeServiceDescriptorTableShadow
```

As shown above, we can search the whole kernel address from the kernel base for **KiSystemServiceStart** 's pattern

```c
const unsigned char KiSystemServiceStartPattern[] = { 
    0x8B, 0xF8,                     // mov edi,eax
    0xC1, 0xEF, 0x07,               // shr edi,7
    0x83, 0xE7, 0x20,               // and edi,20h
    0x25, 0xFF, 0x0F, 0x00, 0x00    // and eax,0fffh  
};

bool found = false;
ULONG KiSSSOffset;
for(KiSSSOffset = 0; KiSSSOffset < kernelSize - signatureSize; KiSSSOffset++)
{
    if(RtlCompareMemory(
        ((unsigned char*)kernelBase + KiSSSOffset), 
        KiSystemServiceStartPattern, signatureSize) == signatureSize)
    {
        found = true;
        break;
    }
}
if(!found)
    return NULL;
```

After find the target function, we can retrieve the address of **nt!KeServiceDescriptorTableShadow** in the instruction:

```text
4c8d1d00212300  lea     r11,[nt!KeServiceDescriptorTableShadow
```

```c
ULONG_PTR addressAfterPattern = kernelBase + kiSSSOffset + signatureSize
ULONG_PTR address = addressAfterPattern + 7 // Skip lea r10,[nt!KeServiceDescriptorTable]
LONG relativeOffset = 0;
// lea r11, KeServiceDescriptorTableShadow
if((*(unsigned char*)address == 0x4c) &&
   (*(unsigned char*)(address + 1) == 0x8d) &&
   (*(unsigned char*)(address + 2) == 0x1d))
{
    relativeOffset = *(LONG*)(address + 3);
}
if(relativeOffset == 0)
    return NULL;

SSDTStruct* shadow = (SSDTStruct*)( address + relativeOffset + 7 );
```

Then we can get the addresses of **nt!KiServiceTable\(SSDT\)** and **win32k!W32pServiceTable\(SSDT Shadow\)**

```c
PVOID ntTable = (PVOID)shadow;
PVOID win32kTable = (PVOID)((ULONG_PTR)shadow + 0x20);    // Offset showed in Windbg
```

## Find the base address of ntoskrnl.exe and win32k.sys

First, we use the undocumented function **ZwQuerySystemInformation** with parameter of _**SystemModuleInformation**_ to get information of system modules

```c
// Get SystemInfo size first
ZwQuerySystemInformation(SystemModuleInformation,
                      &SystemInfoBufferSize,
                      0,
                      &SystemInfoBufferSize);
// Allocate buffer for system info
pSystemInfoBuffer = (PSYSTEM_MODULE_INFORMATION)ExAllocatePool(NonPagedPool, SystemInfoBufferSize * 2);
// Query SystemIfno
status = ZwQuerySystemInformation(SystemModuleInformation,
             pSystemInfoBuffer,
             SystemInfoBufferSize * 2,
             &SystemInfoBufferSize);
```

while the struct of **SYSTEM\_MODULE\_INFORMATION** is

```c
typedef struct _SYSTEM_MODULE_INFORMATION
{
    ULONG Count;
    SYSTEM_MODULE_ENTRY Module[0];
} SYSTEM_MODULE_INFORMATION, *PSYSTEM_MODULE_INFORMATION;

typedef struct _SYSTEM_MODULE_ENTRY
{
    HANDLE Section;
    PVOID MappedBase;
    PVOID ImageBase;
    ULONG ImageSize;
    ULONG Flags;
    USHORT LoadOrderIndex;
    USHORT InitOrderIndex;
    USHORT LoadCount;
    USHORT OffsetToFileName;
    UCHAR FullPathName[256];
} SYSTEM_MODULE_ENTRY, *PSYSTEM_MODULE_ENTRY;
```

Usually the module **ntoskrnl.exe** is always the first module in the buffer, which is`Module[0]`, so:

```c
PVOID ntBase = pSystemInfoBuffer->Module[0].ImageBase;
```

But the position of **win32k.sys** in the buffer is not sure, so we have to traverse the buffer to find it by compare the **FullPathName** field of each **SYSTEM\_MODULE\_ENTRY** with **"win32k.sys"**

```c
PSYSTEM_MODULE_ENTRY moduleInfo = pSystemInfoBuffer->Module;
UNICODE_STRING win32kPath = RTL_CONSTANT_STRING(L"\\SystemRoot\\system32\\win32k.sys");
UNICODE_STRING moduleName;
ANSI_STRING ansiModuleName;
while ( TRUE )
{
    RtlInitAnsiString( &ansiModuleName, (PCSZ)moduleInfo->FullPathName );
    RtlAnsiStringToUnicodeString( &moduleName, &ansiModuleName, TRUE );
    if ( RtlCompareUnicodeString( &win32kPath, &moduleName, TRUE ) == 0)
    {
        PVOID win32kBase = moduleInfo->ImageBase;
        break;
    }
    moduleInfo = moduleInfo + 1;    // Iterate to next module entry
}
```

## Find SSDT function index by function name

#### Index in ntdll.dll

Almost all SSDT function have a stub function which has the same name and is exported by **NTDLL.DLL**

eg. the first few lines of user-mode function **NtCreateFile** exported by NTDLL.DLL \(_You have to attach to user-mode process to view symbols in ntdll_\)

```text
0: kd> u ntdll!NtCreateFile
ntdll!ZwCreateFile:
00000000`774d1860 4c8bd1          mov     r10,rcx
00000000`774d1863 b852000000      mov     eax,52h        # This is the SSDT index 
00000000`774d1868 0f05            syscall
00000000`774d186a c3              ret
00000000`774d186b 0f1f440000      nop     dword ptr [rax+rax]
```

As above shows, `mov eax, 52h` transfers the system call number to **EAX**, and call `syscall` to trap into the kernel, then kernel will use the number `52h` to find corresponding kernel function in SSDT

```text
0: kd> dd /c 1 nt!KiServiceTable l53    # Function index begin from 0
fffff800`03e97300  040d9a00
fffff800`03e97304  02f55c00
fffff800`03e97308  fff6ea00
fffff800`03e9730c  02e87805
fffff800`03e97310  031a4a06
......
fffff800`03e97438  031bab01
fffff800`03e9743c  02efec80
fffff800`03e97440  02d52300
fffff800`03e97444  04637102
fffff800`03e97448  03071007            # This is the index of nt!NtCreateFile (52h)

0: kd> u FFFFF8000419E400
nt!NtCreateFile:
fffff800`0419e400 4c8bdc          mov     r11,rsp
fffff800`0419e403 4881ec88000000  sub     rsp,88h
fffff800`0419e40a 33c0            xor     eax,eax
fffff800`0419e40c 498943f0        mov     qword ptr [r11-10h],rax
fffff800`0419e410 c744247020000000 mov     dword ptr [rsp+70h],20h
```

**SO, we can get any SSDT function's index from its name by looking into its user-mode stub function exported by NTDLL.DLL!**

#### Get SSDT index from ntdll.dll

First we must read **ntdll.dll** into memory

```c
UNICODE_STRING FileName;
OBJECT_ATTRIBUTES ObjectAttributes;
RtlInitUnicodeString(&FileName, L"\\SystemRoot\\system32\\ntdll.dll");
InitializeObjectAttributes(&ObjectAttributes, &FileName,
                           OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
                           NULL, NULL);

if(KeGetCurrentIrql() != PASSIVE_LEVEL)
{
    return STATUS_UNSUCCESSFUL;
}

HANDLE FileHandle;
IO_STATUS_BLOCK IoStatusBlock;
// Open ntdll.dll file 
NTSTATUS NtStatus = ZwCreateFile(&FileHandle,        
                                 GENERIC_READ,
                                 &ObjectAttributes,
                                 &IoStatusBlock, NULL,
                                 FILE_ATTRIBUTE_NORMAL,
                                 FILE_SHARE_READ,
                                 FILE_OPEN,
                                 FILE_SYNCHRONOUS_IO_NONALERT,
                                 NULL, 0);
if(NT_SUCCESS(NtStatus))
{
    // Get ntdll.dll file size
    FILE_STANDARD_INFORMATION StandardInformation = { 0 };
    NtStatus = ZwQueryInformationFile(FileHandle, &IoStatusBlock, &StandardInformation, sizeof(FILE_STANDARD_INFORMATION), FileStandardInformation);
    if(NT_SUCCESS(NtStatus))
    {
        FileSize = StandardInformation.EndOfFile.LowPart;
        FileData = (unsigned char*)RtlAllocateMemory(true, FileSize);

        LARGE_INTEGER ByteOffset;
        ByteOffset.LowPart = ByteOffset.HighPart = 0;
        // Read ntdll.dll into buffer
        NtStatus = ZwReadFile(FileHandle,
                              NULL, NULL, NULL,
                              &IoStatusBlock,
                              FileData,
                              FileSize,
                              &ByteOffset, NULL);
    }

    ZwClose(FileHandle);
}
```

Then get the real function address of any exported function .

To do this, we have to get its **Export Address Table** from its **PE header**

```c
 //Verify DOS Header
PIMAGE_DOS_HEADER pdh = (PIMAGE_DOS_HEADER)FileData;
//Verify PE Header
PIMAGE_NT_HEADERS pnth = (PIMAGE_NT_HEADERS)(FileData + pdh->e_lfanew);
//Verify Export Directory
PIMAGE_DATA_DIRECTORY pdd = NULL;
if(pnth->OptionalHeader.Magic == IMAGE_NT_OPTIONAL_HDR64_MAGIC)
    pdd = ((PIMAGE_NT_HEADERS64)pnth)->OptionalHeader.DataDirectory;
else
    pdd = ((PIMAGE_NT_HEADERS32)pnth)->OptionalHeader.DataDirectory;
ULONG ExportDirRva = pdd[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress;
ULONG ExportDirSize = pdd[IMAGE_DIRECTORY_ENTRY_EXPORT].Size;
ULONG ExportDirOffset = RvaToOffset(pnth, ExportDirRva, FileSize);

//Read Export Directory
PIMAGE_EXPORT_DIRECTORY ExportDir = (PIMAGE_EXPORT_DIRECTORY)(FileData + ExportDirOffset);
ULONG NumberOfNames = ExportDir->NumberOfNames;
ULONG AddressOfFunctionsOffset = RvaToOffset(pnth, ExportDir->AddressOfFunctions, FileSize);
ULONG AddressOfNameOrdinalsOffset = RvaToOffset(pnth, ExportDir->AddressOfNameOrdinals, FileSize);
ULONG AddressOfNamesOffset = RvaToOffset(pnth, ExportDir->AddressOfNames, FileSize);

ULONG* AddressOfFunctions = (ULONG*)(FileData + AddressOfFunctionsOffset);
USHORT* AddressOfNameOrdinals = (USHORT*)(FileData + AddressOfNameOrdinalsOffset);
ULONG* AddressOfNames = (ULONG*)(FileData + AddressOfNamesOffset);
```

and get the file offset of the function address from its name

```c
// Find RVA of exported function whose name is ExportName
ULONG ExportOffset = PE_ERROR_VALUE;
for(ULONG i = 0; i < NumberOfNames; i++)
{
    ULONG CurrentNameOffset = RvaToOffset(pnth, AddressOfNames[i], FileSize);
    if(CurrentNameOffset == PE_ERROR_VALUE)
        continue;
    const char* CurrentName = (const char*)(FileData + CurrentNameOffset);
    ULONG CurrentFunctionRva = AddressOfFunctions[AddressOfNameOrdinals[i]];
    if(CurrentFunctionRva >= ExportDirRva && CurrentFunctionRva < ExportDirRva + ExportDirSize)
        continue; //we ignore forwarded exports
    //compare the export name to the requested export
    if(!strcmp(CurrentName, ExportName))  
    {
        // Convert function RVA to offset
        ExportOffset = RvaToOffset(pnth, CurrentFunctionRva, FileSize);
        break;
    }
}
```

Then get the real address of exported function in memory \( in ntdll.dll \)

```c
ExportFunctionAddress = (PVOID)((ULONG_PTR)FileData + ExportOffset)
```

finally search the whole function for pattern `mov eax, xx`, retrieve the SSDT index

```c
for(int i = 0; i < 32 && ExportOffset + i < FileSize; i++)
{
    if(ExportData[i] == 0xC2 || ExportData[i] == 0xC3)  //RET
        break;
    if(ExportData[i] == 0xB8)  //mov eax,XX    -- b8 FFFFFFFF
    {
        SsdtOffset = *(int*)(ExportData + i + 1);
        break;
    }
}
```

#### Combine index with SSDT\(Shadow\)

x64

```c
realNtFunction = (PVOID)(ntTable[SsdtOffset] >> 4 + ntTable);
```

x86

```c
realNtFunction = (PVOID)(ntTable[SsdtOffset]);
```

**Shadow is similar**

## Hook SSDT

{% hint style="info" %}
**!!! Yon have to first disable PatchGuard or enable DebugMode on your system !!!** 😀 
{% endhint %}

In order to unhook SSDT in future, define some hook structs as below \(**HOOKOPCODES** is our shellcode for x64 hook\)

```c
#pragma pack(push,1)        // This is very important !!!!
struct HOOKOPCODES
{
#ifdef _WIN64
    unsigned short int mov;
#else
    unsigned char mov;
#endif
    ULONG_PTR addr;
    unsigned char push;
    unsigned char ret;
};
#pragma pack(pop)

typedef struct HOOKSTRUCT
{
    ULONG_PTR addr;
    HOOKOPCODES hook;
    unsigned char orig[sizeof(HOOKOPCODES)];
    //SSDT extension
    int SSDTindex;
    LONG SSDTold;
    LONG SSDTnew;
    ULONG_PTR SSDTaddress;
}HOOK, *PHOOK;
```

### Disable Write Protection

Because SSDT are read-only kernel memory, so you have to disable **write protection** before modifying it , we use **MDL** to do this. Below is the helper function to copy data to any read-only kernel memory

```c
NTSTATUS SuperCopyMemory(
    IN VOID UNALIGNED* Destination, 
    IN CONST VOID UNALIGNED* Source, 
    IN ULONG Length)
{
    //Change memory properties.
    PMDL g_pmdl = IoAllocateMdl(Destination, Length, 0, 0, NULL);
    if(!g_pmdl)
        return STATUS_UNSUCCESSFUL;
    MmBuildMdlForNonPagedPool(g_pmdl);
    unsigned int* Mapped = (unsigned int*)MmMapLockedPages(g_pmdl, KernelMode);
    if(!Mapped)
    {
        IoFreeMdl(g_pmdl);
        return STATUS_UNSUCCESSFUL;
    }
    KIRQL kirql = KeRaiseIrqlToDpcLevel();
    RtlCopyMemory(Mapped, Source, Length);
    KeLowerIrql(kirql);
    //Restore memory properties.
    MmUnmapLockedPages((PVOID)Mapped, g_pmdl);
    IoFreeMdl(g_pmdl);
    return STATUS_SUCCESS;
}
```

Also, we can modify the "**WP"** flags of the `cr0`register to **enable** or **disable** kernel memory write protection

**To disable WP:**

```c
KIRQL DisableWP(){
    KIRQL irql=KeRaiseIrqlToDpcLevel();
    ULONG_PTR cr0=__readcr0();
#ifdef _AMD64_        
    cr0 &= 0xfffffffffffeffff;
#else
    cr0 &= 0xfffeffff;
#endif
    __writecr0(cr0);
    _disable();    // Disable interrupts
    return irql;
}
```

there we can modify the kernel memory without access violation. 🧐

**To enable WP after writing:**

```c
void EnableWP(KIRQL irql){
	ULONG_PTR cr0=__readcr0();
	cr0 |= 0x10000;
	_enable();		// Enable interrupts
	__writecr0(cr0);
	KeLowerIrql(irql);
}
```

### x86 Hook

Just replace the SSDT index with your own function

```c
PVOID realNtFunction = ntTable[FunctionIndex];

hHook = (HOOK)RtlAllocateMemory(true, sizeof(HOOK));
hHook.SSDTold = realNtFunction;
hHook.SSDTnew = YourFunction;
hHook.SSDTindex = FunctionIndex;
hHook.SSDTaddress = realNtFunction;

SuperCopyMemory(&realNtFunction, YourFunction);
```

### x64 Hook

In x64 platform, because `realFunction = ntTable[FunctionIndex] >> 4 + ntTable`, the only thing you can modify is the value of `ntTable[FunctionIndex]`, and it's **long** type\(32 bit\), which means you can only jump within the range **-2GB + ntTable ~ 2GB + ntTable**, Obviously the address is inside the **ntoskrnl.exe module address space**. So to jump to the function located in our own module, we have to make a **"indirect jump"**.

Jump from SSDT to our stub function, then jump to our real function. Below is the stub shellcode

```text
movabs rax, xxxxxxxxxxxxxxxx ; 48B8 XXXXXXXXXXXXXXXX (move our real function to rax)
push rax ; 50
ret ; c3 (return to rax, which is our function)
```

As you can see, the shellcode is only 12 bytes, so we only need to find a executable memory block which is larger than 12 bytes. Well in every code page\(4KB size\) there are many **nop** instruction where we can insert our shellcode, So we try to insert the shellcode into the code page where the real kernel function to be hooked is.

Get the ntoskrnl.exe's **section header** first

```c
ULONG dwRva = (ULONG)((unsigned char*)realNtFunction - (unsigned char*)ntBase);
IMAGE_DOS_HEADER* pdh = (IMAGE_DOS_HEADER*)ntBase;
if(pdh->e_magic != IMAGE_DOS_SIGNATURE)
    return 0;
IMAGE_NT_HEADERS* pnth = (IMAGE_NT_HEADERS*)((unsigned char*)ntBase + pdh->e_lfanew);
if(pnth->Signature != IMAGE_NT_SIGNATURE)
    return 0;
IMAGE_SECTION_HEADER* psh = IMAGE_FIRST_SECTION(pnth);
```

find out in which section the real function is

```c
static ULONG RvaToSection(IMAGE_NT_HEADERS* pNtHdr, ULONG dwRVA)
{
    USHORT wSections;
    PIMAGE_SECTION_HEADER pSectionHdr;
    pSectionHdr = IMAGE_FIRST_SECTION(pNtHdr);
    wSections = pNtHdr->FileHeader.NumberOfSections;
    for(int i = 0; i < wSections; i++)
    {
        if(pSectionHdr[i].VirtualAddress <= dwRVA &&
           (pSectionHdr[i].VirtualAddress + pSectionHdr[i].Misc.VirtualSize) > dwRVA)
                return i;
    }
    return (ULONG) - 1;
}

int section = RvaToSection(pnth, dwRva);
if(section == -1)
    return 0;

CodeSize = psh[section].SizeOfRawData;
CodeStart = (PVOID)((ULONG_PTR)ntBase + psh[section].VirtualAddress);
```

then try to find a at least 12 bytes large nop block in the code page

```c
unsigned char* Code = (unsigned char*)CodeStart;
unsigned int CaveSize = sizeof(HOOKOPCODES);
for(unsigned int i = 0, j = 0; i < CodeSize; i++)
{
    if(Code[i] == 0x90 || Code[i] == 0xCC)  //NOP or INT3
        j++;
    else
        j = 0;
    if(j == CaveSize)
        ShellcodeCave = (PVOID)((ULONG_PTR)CodeStart + i - CaveSize + 1);
}
```

Next we need to write the shellcode to the cave address we just got, and insert our own function address

initialize the hook struct

```c
//allocate structure
PHOOK hook = (PHOOK)RtlAllocateMemory(true, sizeof(HOOK));
//set hooking address
hook->addr = ShellcodeCave;        // Store the cave address
//set hooking opcode
#ifdef _WIN64
hook->hook.mov = 0xB848;
#else
hook->hook.mov = 0xB8;
#endif
hook->hook.addr = (ULONG_PTR)YourFunction;    // Insert our own function
hook->hook.push = 0x50;
hook->hook.ret = 0xc3;
//set original data
RtlCopyMemory(&hook->orig, (const void*)addr, sizeof(HOOKOPCODES));
```

write to the cave address

```c
SuperCopyMemory((void*)ShellcodeCave, &hook->hook, sizeof(HOOK));
```

After that, we can change the corresponding SSDT offset to point to shellcode cave.

**You have to convert cave address to the relative offset to SSDT first !!**

```c
oldOffset = ntTable[FunctionIndex];
newOffset = (LONG)((ULONG_PTR)ShellcodeCave - ntTable);
newOffset = (newOffset << 4) | oldOffset & 0xF;
```

update SSDT

```c
hook.SSDTold = oldOffset;
hook.SSDTnew = newOffset;
hook.SSDTIndex = FunctionIndex;
hook.SSDTaddress = realNtFunction;

SuperCopyMemory(&ntTable[FunctionIndex], newOffset);
```

**DONE!**

## Hook Shadow SSDT

The steps of hooking Shadow is similar with SSDT hook, except that you have to attach to a user-mode GUI process before getting the base address of the Shadow. 

```c
// Get the process id of the "winlogon.exe" process
GetProcessIdByName(ProcessId, "winlogon.exe");
PsLookupProcessByProcessId( ProcessId, &Process );
APC_STATE oldApc;
KeStackAttachProcess(Process, &oldApc);

// Find Shadows SSDT and do dirty things

KeUnstackDetachProcess(&oldApc);
```

