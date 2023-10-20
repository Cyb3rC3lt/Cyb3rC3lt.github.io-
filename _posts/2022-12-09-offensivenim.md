---
layout: post
title: "Offensive Nim"
permalink: "offensivenim"
date: 19-10-2023
categories: Blog
---

During some downtime in my pentesting recently, I have  tried to increase my knowledge of the tooling required to avoid AV. I had built a basic runner in C++ that incorporated syscalls to bypass Defender, but I was seeing some [videos](https://www.youtube.com/watch?v=vq6wNGYzdDE) by John Hammond explaining the powers of the programming language called Nim, so I wanted to give it a try next.

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

*Windows API*       *Native API*
VirtualAllocEx      NtAllocateVirtualMemory
WriteProcessMemory  NtWriteVirtualMemory
VirtualProtectEx    NtProtectVirtualMemory
CreateRemoteThread  NtCreateThreadEx


### Coding





Before I go any further, I want to say that I am standing on the shoulders of giants here and that I didn't build this from scratch but just refactored the Cobalt Strike BOF created by OutflankNL to now work with Sliver.

To get BOF's to work with Sliver you ideally want 3 files:

- An x86 object file for 32 bit systems.
- An x64 object file for 64 bit systems.
- An extension.json file shown below to tell Sliver where the files are located and other config information.

**extension.json:**
{% highlight powershell %}
{
    "name": "klist",
    "version": "1.0.0",
    "command_name": "klist",
    "extension_author": "Cyb3rC3lt",
    "original_author": "OutflankNl",
    "help": "Displays a list of currently cached Kerberos tickets.",
    "long_help": "",
    "depends_on": "coff-loader",
    "entrypoint": "go",
    "files": [
        {
            "os": "windows",
            "arch": "amd64",
            "path": "klist.x64.o"
        },
        {
            "os": "windows",
            "arch": "386",
            "path": "klist.x86.o"
        }
    ],
    "arguments": [
        {
            "name": "purge",
            "desc": "Purge the cached Kerberos tickets.",
            "type": "wstring",
            "optional": true
        }
    ]
}
{% endhighlight %}

### Install The Release

The quickest way to add these 3 files to Sliver is as follows.

1. Download the zip file from my releases [here](https://github.com/Cyb3rC3lt/SliveryArmory/releases/tag/v1.0.0)
2. Extract it to a folder on your machine named klist for argument sake.
3. Within Sliver load the folder you extracted with this command:
{% highlight powershell %}
extensions install /home/david/klist
{% endhighlight %}
4. Then load the extension into Sliver as follows:
{% highlight powershell %}
extensions load /home/david/.sliver-client/extensions/klist
{% endhighlight %}

### Install From Source

1. Make sure that Mingw-w64 (including mingw-w64-binutils) has been installed.
2. Download the source folder above.
3. Within that folder execute "make" to compile the object files.
4. Now you have the object files like they appear in the release zip folder so continue from Step 1 of the release method.

### Usage

To display all the cached Kerberos tickets issue the command:
{% highlight powershell %}
klist
{% endhighlight %}
To purge all the cached Kerberos tickets issue the command:
{% highlight powershell %}
klist purge
{% endhighlight %}
### Screenshots

<img src="https://user-images.githubusercontent.com/33097451/274965338-4ac8bf58-9134-4c1d-9d00-efe0bee11b75.png"/>

<img src="https://user-images.githubusercontent.com/33097451/274966113-146cafe6-f3c8-43c6-ad8c-2ad417bfd129.png"/>

After speaking to the Sliver devs over at the BloodHoundGang Slack channel, they have said they would like to include it in the Armory. They advised me to open an issue on their Github so it can be verified for inclusion. This is now located here:

[https://github.com/BishopFox/sliver/issues/1433](https://github.com/BishopFox/sliver/issues/1433)

This process has been lots of fun and a great learning experience. It gave me an opportunity to work with such an interesting C2 framework as Sliver and to improve my knowledge on how it's BOF framework operates. Hope it proved to be informative and happy hacking!





