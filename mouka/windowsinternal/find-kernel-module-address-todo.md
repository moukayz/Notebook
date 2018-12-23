# Find Kernel Module Address

Sometimes we need to get the base address of some kernel modules, such as ntoskrnl, win32k and other device driver modules. So far as I know there are three methods we can use to make it 🤓 

## Method 1: Query system information

MS has an undocumented function named `ZwQuerySystemInformation` which can obtain many information about the system. Its declaration is as below\( [see msdn docs page](https://docs.microsoft.com/en-us/windows/desktop/sysinfo/zwquerysysteminformation) \):

```c
NTSYSAPI
NTSTATUS
NTAPI
ZwQuerySystemInformation(
    IN SYSTEM_INFORMATION_CLASS SystemInformationClass,
    OUT PVOID SystemInformation,
    IN ULONG SystemInformationLength,
    OUT PULONG ReturnLength OPTIONAL 
    );
```

The first parameter `SystemInformationClass` indicates what information the function should query. Here we pass `SystemModuleInformation` as argument to get information about all kernel modules.

First we pass 0 to `SystemInformation` buffer to get accurate size of this information, allocate memory respectively and query system information.

```c
status = ZwQuerySystemInformation( SystemModuleInformation, 0, bytes, &bytes );

pMods = (PSYSTEM_MODULE_INFORMATION)ExAllocatePoolWithTag( NonPagedPool, bytes, 'tag');
RtlZeroMemory( pMods, bytes );

status = ZwQuerySystemInformation( SystemModuleInformation, pMods, bytes, &bytes );
```

there are two structure we need to use to extract the specific information:

```c
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

typedef struct _SYSTEM_MODULE_INFORMATION
{
    ULONG Count;
    SYSTEM_MODULE_ENTRY Module[1];
} SYSTEM_MODULE_INFORMATION, *PSYSTEM_MODULE_INFORMATION;
```

as you can see, `Module` field in `_SYSTEM_MODULE_INFORMATION` is a list of `_SYSTEM_MODULE_ENTRY` in which each element represents a information block of corresponding module.

Generally the **ntoskrnl** module is always the first entry whose base address is `pMods->Module[0].ImageBase`. For other modules we can loop through the list and compare each entry's **FullPathName**, with our target module's path to get target module's base address. 

 \( **Note** that the path is "_**\Systemroot\system32\module-name**_ "\).

```c
PSYSTEM_MODULE_ENTRY pMod = pMods->Modules;
STRING targetModuleName = RTL_CONSTANT_STRING("\\systemroot\\system32\\win32k.sys");
STRING current;
for (ULONG i = 0; i < pMods->NumberOfModules; i++)
{
    rRtlInitAnsiString(&current, (PCSZ)pMod[i].FullPathName);
    if (0 == RtlCompareString(&targetModuleName, &current, TRUE))
    {
        g_ModuleBase = pMod[i].ImageBase;
        g_ModuleSize = pMod[i].ImageSize;
        break;
    }
}
```

Alternatively, if you have some data or function address belonging to a certain module, we can use it to get the module base address.

```c
for (ULONG i = 0; i < pMods->NumberOfModules; i++)
{
    // System routine is inside module
    // checkPtr is an address within a module
    if (checkPtr >= pMod[i].ImageBase &&
        checkPtr < (PVOID)((PUCHAR)pMod[i].ImageBase + pMod[i].ImageSize))
    {
        g_KernelBase = pMod[i].ImageBase;
        g_KernelSize = pMod[i].ImageSize;
        break;
    }
}
```

## Method 2: Traverse system module list

There is a `LIST_ENTRY` data structure in kernel named `PsLoadedModuleList`, which is the head of a list of information blocks about all kernel modules. Each block has a structure defined as below:

```c
typedef struct _KLDR_DATA_TABLE_ENTRY
{
    LIST_ENTRY InLoadOrderLinks;
    PVOID ExceptionTable;
    ULONG ExceptionTableSize;
    PVOID GpValue;
    PNON_PAGED_DEBUG_INFO NonPagedDebugInfo;
    PVOID DllBase;
    PVOID EntryPoint;
    ULONG SizeOfImage;
    UNICODE_STRING FullDllName;
    UNICODE_STRING BaseDllName;
    ULONG Flags;
    USHORT LoadCount;
    USHORT __Unused5;
    PVOID SectionPointer;
    ULONG CheckSum;
    PVOID LoadedImports;
    PVOID PatchInformation;
} KLDR_DATA_TABLE_ENTRY, *PKLDR_DATA_TABLE_ENTRY;
```

The first field `InLoadOrderLinks` of every block is linked to the list, so we can traverse the list and for a block retrieve some information like `FullDllName` or `BaseDllName` , then compare them with our target module and get its base address.

### Find PsLoadedModuleList

This structure is the head of the system module list, and the **ntoskrnl** module is always the first entry in the list, so we can find **ntoskrnl** in the list first and trace back to find **PsLoadedModuleList**. 

First we need to find **ntoskrnl** 's base address \( well we can use method 1 showed above, and something else will be added here in future\)

Then we need to find a breakthrough point into the system modules list. Actually we can use our own driver module to do this \( because it's in the list too\). 

In every driver's entry function `DriverEntry` , there is a parameter named `DriverObject` which has type of `PDRIVER_OBJECT` \( [see msdn pages for detailed information of this structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/ns-wdm-_driver_object) \). The field  `DriverSection` with `PVOID` type is actually a pointer to the `KLDR_LOAD_TABLE_ENTRY` structure stored information of the driver module, which is exactly the block inserted into the system modules list. 

So the basic idea is to start from the list block of our driver, then traverse the list to find **ntoskrnl** module and get the list head **PsLoadedMouldList** at last.

```c
// Get kernel base address already

// Get PsLoadedModuleList address
for (PLIST_ENTRY pListEntry = pThisModule->InLoadOrderLinks.Flink; pListEntry != &pThisModule->InLoadOrderLinks; pListEntry = pListEntry->Flink)
{
    // Search for Ntoskrnl entry
    PKLDR_DATA_TABLE_ENTRY pEntry = CONTAINING_RECORD( pListEntry, KLDR_DATA_TABLE_ENTRY, InLoadOrderLinks );
    if (kernelBase == pEntry->DllBase)
    {
        // Ntoskrnl is always first entry in the list
        // So the previous entry is the PsLoadedModuleList
        // Check if found pointer belongs to Ntoskrnl module
        if ((PVOID)pListEntry->Blink >= pEntry->DllBase && (PUCHAR)pListEntry->Blink < (PUCHAR)pEntry->DllBase + pEntry->SizeOfImage)
        {
            PsLoadedModuleList = pListEntry->Blink;
            break;
        }
    }
}
```

### Find specified module

After we find the **PsLoadedModuleList**, To find any given kernel module we just need to traverse the list at the beginning. Like method 1, we can use either the module name or a address within the module.

```c
// No images
if (IsListEmpty( PsLoadedModuleList ))
    return NULL;

// Search in PsLoadedModuleList
for (PLIST_ENTRY pListEntry = PsLoadedModuleList->Flink; pListEntry != PsLoadedModuleList; pListEntry = pListEntry->Flink)
{
    PKLDR_DATA_TABLE_ENTRY pEntry = CONTAINING_RECORD( pListEntry, KLDR_DATA_TABLE_ENTRY, InLoadOrderLinks );

    // Check by name or by address
    if ((pName && RtlCompareUnicodeString( &pEntry->BaseDllName, pName, TRUE ) == 0) ||
         (pAddress && pAddress >= pEntry->DllBase && (PUCHAR)pAddress < (PUCHAR)pEntry->DllBase + pEntry->SizeOfImage))
    {
        return pEntry;
    }
}
```

## Method 3: Through Driver Name

Every driver has a name like `\Driver\driver-name`, and we can get it's driver object pointer with its name by an undocumented function `ObReferenceObjectByName`, declared below:

```c
 NTSTATUS
 NTSYSAPI
 NTAPI
 ObReferenceObjectByName(
         IN PUNICODE_STRING ObjectPath,
         IN ULONG Attributes,
         IN PACCESS_STATE PassedAccessState,
         IN ACCESS_MASK DesiredAccess,
         IN POBJECT_TYPE ObjectType,
         IN KPROCESSOR_MODE AccessMode,
         IN OUT PVOID ParseContext,
         OUT PVOID* ObjectPtr);
```

Well, the parameter `ObjectPath` is the driver name we need to specify, `AccessMode` needs to be `KernelMode`, and `ObjectPtr` is used to receive the pointer to `_DRIVER_OBJECT` structure of specified driver module. Moreover the parameter `ObjectType` indicates which type the object belongs to, here the driver is `IoDriverObjectType` \( **Note** this type is undocumented so we need declare it explicitly as below\).

Here we use keyboard class driver \( **kbdclass.sys**\) as a example:

```c
// Declare the driver type explicitly
extern POBJECT_TYPE IoDriverObjectType;

UNICODE_STRING kbdName = RTL_CONSTANT_STRING(L"\\Driver\\kbdclass");
PDRIVER_OBJECT pKbdDriverObject;

ObReferenceObjectByName(
    &kbdName,
    OBJ_CASE_INSENSITIVE,
    NULL,
    0,
    IoDriverObjectType,
    KernelMode,
    NULL,
    &pKbdDriverObject);
```

After we get the driver object pointer of specified driver, we can retrieve the `KLDR_LOAD_TABLE_ENTRY` structure through the `DriverSection` field and get some useful information including its base address.

