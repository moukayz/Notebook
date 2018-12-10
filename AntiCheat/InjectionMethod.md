# R3 Dll 注入方式总结

## 1. SetWindowsHookEx 方法

```c
HHOOK WINAPI SetWindowsHookEx(
  _In_ int       idHook,
  _In_ HOOKPROC  lpfn,
  _In_ HINSTANCE hMod,
  _In_ DWORD     dwThreadId
);
```

> 该方法主要通过注册 指定线程（dwThreadId） 的 Windows 消息事件（idHook）的回调函数（lpfn）来进行注入
> 当 dwThreadId 设为 0 或者不是由当前进程创建的线程时，该 hook 函数（lpfn）必须存在于 指定的 dll （hMod）中
> eg. ` SetWindowsHookEx(WH_CBT, MyHookProc, MyDll, 0) ` ，当系统中有线程触发 WH_CBT事件后，系统自动将 MyDll载入该线程中，并执行 MyHookProc，以此实现  Dll 注入 



## 2. CreateRemoteThread 方法

```c
HANDLE CreateRemoteThread(
  HANDLE                 hProcess,
  LPSECURITY_ATTRIBUTES  lpThreadAttributes,
  SIZE_T                 dwStackSize,
  LPTHREAD_START_ROUTINE lpStartAddress,
  LPVOID                 lpParameter,
  DWORD                  dwCreationFlags,
  LPDWORD                lpThreadId
);
```

> 该方法在CreateRemoteThread 在目标进程中创建一个线程，再通过 新创建的线程进行dll注入
> 1. ` loadLibraryAddr = GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA") // 获取函数 LoadLibrary 在内存中的地址 `
>
> 2. ` remoteDllAddr = VirtualAllcoEx (targetProcess, NULL, str(myDllPath)+1, MEM_COMMIT | MEM_READWRITE) // 在目标进程空间给将要注入的dll路径字符串（形如“c:\my.dll”）分配空间 `
>
> 3. ` WriteProcesMemory(targetProcess, remoteDllAddr, (LPVOID)myDllPath, strlen(myDllPath)+1, NULL) // 将dll路径 写入目标进程空间 `  
>
> 4. ```c
>   CreateRemoteThread(
>   	targetProcess,	// 目标进程
>   	NULL,
>   	0,
>   	// 线程的执行函数（此处是LoadLibrary)
>   	(LPTHREAD_START_ROUTINE)loadLibraryAddr, 
>   	// 执行函数的参数（此处是 需要载入的dll路径）
>   	remoteDllPath,
>   	NULL,NULL
>   )
>   ```
>   **通过该方法创建的线程，其创建进程和父进程均为 目标进程 ！**

**另：还可以通过RtlCreateUserThread 和 NtCreateThread 函数进行注入，但这两个函数没有从ntdll.dll导出，因此需要手动寻找这两个函数的地址。注入原理和 CreateRemoteThread非常相似，只是替换了CreateRemoteThread 来在目标进程中创建线程，因此将它们归为一类**

## 2.5  Reflective Dll Injection

> 该方法依然使用线程创建函数在目标进程中创建线程， 但不是通过使线程执行 LoadLibrary 来载入线程
>
> 具体过程比较复杂，简述如下：
>
> 1. 将需要载入的dll的文件内容读入内存中（读入其二进制代码）
> 2. 搜索其PE头，查找该dll 的入口地址（EntryPoint：DllMain）  **（关键一步）**
> 3. 将其入口地址（DllMain）直接作为 线程创建函数的线程起始地址（lpStartAddress)
> 4. 创建远程线程



## 3. QueueUserApc 方法

```c
DWORD QueueUserAPC(
  PAPCFUNC  pfnAPC,
  HANDLE    hThread,
  ULONG_PTR dwData
);
```

> - 该方法和 CreateRemoteThread 类似，将实现注入功能的函数（pfnAPC）加入目标线程的 APC 队列中，当目标线程（hThread）通过 SleepEx、WaitForSingleObject 、WaitForMultipleObject 等过程 **变为 Alertable 状态** 时，该线程将依次调用其APC队列中的函数，并以dwData作为传入参数，以此实现dll注入
>
> - 和 CreateRemoteThread相同，pfnAPC 指向 LoadLibrary 的内存地址，dwData则为需要注入的dll的 路径
>
>   **（目标线程的状态必须为 Alertable，否则不会执行 APC ）**



## 4. SetThreadContext 方法

> - 该方法原理是 通过 SetThreadContext 函数更改 目标进程上下文中的指令指针（EIP for x86），跳转到目标进程中我们写入的内存区域，并开始执行，以此实现dll注入或任意代码执行。
>
> - 由于该方法无法传递参数，因此需要在目标进程中写入一段 汇编代码 用于处理参数并调用 LoadLibrary，然后将EIP指向该段代码起始地址，汇编码如下所示：
>
>   **（写入内容实际是汇编代码对应的机器码）**
>
>   ```c
>   // 0xAAAAAAAA 为占位符，以后会被替换
>   unsigned char shellcode[] =
>   {
>   	0x68, 0xef, 0xbe, 0xad, 0xde,		// push 0xAAAAAAAA, 将原EIP值压栈
>   	0x9c,							  // pushfd，通用寄存器压栈
>   	0x60,							  // pushad，标志寄存器压栈
>   	0x68, 0xef, 0xbe, 0xad, 0xde,	    // push 0xAAAAAAAA， 将dll路径压栈（传参）
>   	0xb8, 0xef, 0xbe, 0xad, 0xde,	    // mov eax, 0xAAAAAAAA 
>   	0xff, 0xd0,						  // call eax， 调用 LoadLibrary
>   	0x61,							 // popad
>   	0x9d,							 //popfd
>   	0xc3							 //ret
>   };
>   ```
>
> - 注入流程简述如下：
>
>   1. 获取 LoadLibrary 内存地址 -  loadLibraryAddr
>
>   2. 在目标进程中申请内存，将dll路径写入 - remoteDllAddr
>
>   3. 在目标进程中申请内存，用于保存shellcode  - remoteShellcodeAddr
>
>   4. 使用 ` SuspendThread(htargetThread) ` 挂起目标线程，并使用` GetThreadContext(targetThread, &context)` 获取线程上下文 context
>
>   5. 存储 context中的原EIP并更新 EIP 
>
>      ```c
>      oldEip = context.Eip;
>      context = (DWORD)remoteShellCode;
>      context.ContextFlags = CONTEXT_CONTROL;
>      ```
>
>   6. 替换 shellcode中的相关地址参数
>
>      ```c
>      // 修改内存页为可写可执行属性
>      VirtualProtect(shellcode, sizeof(shellcode), PAGE_EXECUTE_READWRITE, &oldProtect);
>      memcpy((void*)(shellcode + 1), &oldEip);	
>      memcpy((void*)(shellcode + 8), &remoteDllAddr);
>      memcpy((void*)(shellcode + 13, &loadLibraryAddr));
>      ```
>
>   7. 将shellcode写入目标进程并开始执行
>
>      ```c
>      // 写入shellcode
>      WriteProcessMemory(targetProcess, remoteShellcodeAddr, shellcode, NULL);
>      // 更改线程上下文
>      SetThreadContext(targetThread, &context);
>      // 恢复执行
>      ResumeThread(targetThread);
>      ```

# 总结

当前 Windows 平台用户层的注入方法主要为以上4种，其共同点为均需要在目标进程的内存中申请空间来保存相关参数，再通过创建远程线程或注入shellcode的方式直接或间接调用 LoadLibrary 来载入dll （Reflective Injection 除外）

因此涉及到的函数主要有` VirtualAlloc(Ex)、WriteProcessMemory、VirtualProtect(Ex) `， 为保证这些函数成功执行，其传入的目标进程句柄（hProcess）必须拥有相应的权限，通常进程句柄通过 OpenProcess 获取

```c
HANDLE OpenProcess(
  DWORD dwDesiredAccess,	// 句柄权限， PROCESS_ALL_PROCESS表示请求所有权限
  BOOL  bInheritHandle,
  DWORD dwProcessId			// 进程id
);
```

PROCESS_ALL_ACCESS包括以下 **等**  特定权限：

- PROCESS_TERMINATE			

	 PROCESS_CREATE_THREAD		
	 PROCESS_DUP_HANDLE			
	 PROCESS_SET_INFORMATION				
	 PROCESS_VM_OPERATION		
	 PROCESS_VM_READ				
	 PROCESS_VM_WRITE			

而以上函数要求进程句柄至少具有 PROCESS_CREATE_THREAD,  PROCESS_VM_OPERATION,  PROCESS_VM_WRITE 权限

### 防御 
在我们的驱动中，对打开进程或线程句柄的操作注册了回调函数，当某一个进程句柄被请求时，回调函数首先查看` DesiredAccess`参数，如果相应权限位被置1，则将该位置为0（即拒绝该权限的请求行为），因此通过OpenProcess函数返回的句柄将不具有 读写内存或进行内存操作的权限，以此来防御 dll 注入

目前为止，驱动可以防御住所有的用户层 针对特定进程的 dll 注入行为