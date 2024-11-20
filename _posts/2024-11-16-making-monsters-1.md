---
title: Making Monsters - Part 1
date: 2024-11-16 09:09:09 +/-TTTT
categories: [SECURITY, REDTEAM, MALDEV]
tags: [mythic, hannibal, maldev, c2, c, pic, position independent]     # TAG names should always be lowercase
description: Introduction, development setup, and building.
---


![hannibal_logo](/assets/img/hannibal_logo.png)


## Preface

This is the companion development journal for [Hannibal](https://github.com/silentwarble/Hannibal). 

This article will attempt to stay concise, provide reasoning for decisions, and highlight pro/cons for each. It will be split into several parts of digestible length.

It is possible that the agent will continue to evolve and introduce breaking changes. This article may or may not be updated as well. For now, assume these details are accurate for the 1.0.0 version.

## Introduction

In part one we will explore the overall use-case for Hannibal, how to setup a development environment for it, and build it.

### What

Hannibal is a C2 agent currently designed to be used with Mythic. It is written in position independent (PIC) C and built off of the [Stardust](https://github.com/Cracked5pider/Stardust) template. It communicates over HTTP with a custom protocol leveraging a Mythic translator service.

### Why
Throughout the years doing Red Team engagements, I've several times needed a very small in-memory agent with the ability to add/remove functionality. I've also seen other Operators in the industry requesting "mini" versions of COTS C2 agents in vendor support channels.

Ideally, this agent would have these characteristics:

- Tiny memory footprint
- Minimal dependencies
- Small shellcode to enable stashing in size restrictive places. 
- The ability to swap communication profiles to change network messaging and protocols.
- Loosely coupled design for simple way to change core systems and behavior.

Being able to replace core parts of the agent would assist with evasion should they be signatured. And the small code surface would lower detection opportunities. This is ideal for an initial foothold, especially if conducting phishing exercises. 

Many of the commercial C2 solutions are focused on building more obfuscation on top of their agents. What I wanted however, was the ability to remove/replace what was being picked out of memory entirely.

### How

#### The Agent

After coming across the Stardust template I was attracted to the simplicity of not needing any kind of reflective loading or parsing relocations etc. It gave a high degree of control and in a small size which I was looking for. 

#### The C2 Controller


  [Mythic](https://github.com/its-a-feature/Mythic) was selected as I was already familiar with it having built private agents in the past. The ability to implement a translation layer made it an obvious choice as I wanted the ability to change up messaging protocols quickly. Additionally, this saved me from having to build and maintain a backend.
  
### Advantages and Drawbacks

Now that we have selected the base we're going to build the agent on, here is what we will have to navigate:

Advantages
<blockquote class="prompt-tip">
  <p>
    <li>Minimal size due to C and not linking external libs.</li>
    <li>High degree of control with C.</li>
    <li>Performant due to C. (Dependent on implementation.)</li>
    <li>Mythic saves a significant amount of development cycles for maintaining a backend.</li>
    <li>Arguably lower complexity due to not needing reflective loading.</li>
    <li>Many maldev resources written in C/C++, making it easy to integrate them into the agent.</li>
</p>
</blockquote>

Drawbacks
<blockquote class="prompt-danger">
  <p>
    <li>Depending on the developer, C can be significantly slower to develop in vs C++/C#/Python/Rust etc.</li>
    <li>Depending on the developer, memory issues are easy to introduce.</li>
    <li>No built-in exception handling due to PIC. (This may be fixable, but needs research).</li>
    <li>If exception, we lose agent.</li>
    <li>PIC restrictions severely limit what libraries can be used in the project. Increases dev load.</li>
    <li>Requires learning Mythic. Can be overwhelming at first. Resource heavy. Fairly complex.</li>
</p>
</blockquote>

## Sidenote

Originally, Hannibal was going to include LLVM obfuscation. However, this was decided against due to the added complexities with PIC and increasing .bin size. [This](https://trustedsec.com/blog/behind-the-code-assessing-public-compile-time-obfuscators-for-enhanced-opsec) article is worth a read. However, it may still be worth implementing, but it is not supported at this time. 

## Project Layout

### Mythic

I've found the easiest way to get started developing a Mythic agent is to simply clone [Apollo](https://github.com/MythicAgents/Apollo). Walk through the files to see how it works and change/remove what you want.

A few important locations within it starting from the root of the repo:

- **./apollo/Dockerfile** - This is used to create the environment that your agent will be built in. Hannibal uses a custom Dockerfile vs the one supplied by Mythic as it needs a newer MingW version.
- **./apollo/apollo/mythic/agent_functions/builder.py** - This is the script that is called to actually build your agent. Hannibal uses it to execute make.
- **./apollo/apollo/mythic/agent_functions/*.py** - These scripts define each command that you can load into the agent on the Mythic side. You can parse arguments with them to be sent to your agent.
- **./apollo/apollo/agent_code/** - This directory is where you store your actual agent codebase.

There's one other thing to note with Hannibal. It uses a [translation container](https://docs.mythic-c2.net/customizing/payload-type-development/translation-containers) as well. We will discuss this a bit more in part 2.


### Hannibal 


There are four main types of files you'll find in the project.

#### src/hannibal_* 

Core functionality for the agent.

#### src/cmd_* 

These are commands such as ls, cd, etc. They accept Task structs and place their responses into the task response queue.

#### src/profile_* 

This is the functionality that handles comms with the controller. What language will Hannibal speak? Serialization, deserialization, message formatting, etc.

#### src/utility_* 

We try to keep various components in their own utilities to assist with modular design. Hannibal attempts to make it easy to switch various systems in and out.


## Setup

My personal preferences for dev work are using Linux+NVIM or similar. Especially for developing private in-house tooling due to the telemetry in Microsoft products. However, this particular project had these objectives:

- Open-Source
- Accessible
- Easy debugging, (test direct in IDE, no need for test vm, x64dbg, etc)

With those in mind, Windows 11 was chosen as the OS and VSCode as the IDE. I used [Tiny11](https://github.com/ntdevlabs/tiny11builder) core builder to create a stripped down Windows 11 ISO. This used significantly less disk space, ram, and reduced telemetry. 

MingW was selected as the compiler due to it already being what was used in the Stardust makefile. Additionally, I've found it easier to work with and customize vs Microsoft's compilers.

### Installation

[Chocolatey](https://chocolatey.org/install) was the easiest way to get what I needed installed. 

```
C:\>choco list
Chocolatey v2.3.0
chocolatey 2.3.0
chocolatey-compatibility.extension 1.0.0
chocolatey-core.extension 1.4.0
chocolatey-windowsupdate.extension 1.0.5
git 2.45.2
git.install 2.45.2
git-credential-manager-for-windows 1.20.0
make 4.4.1
mingw 13.2.0
nasm 2.16.3
python 3.12.4
python3 3.12.4
python312 3.12.4
```

### IDE Setup

Here is the launch.json from my VSCode. This is used to enable just pressing F5 to run the debug binary in GDB. This way we can set breakpoints and visually step through the code, inspect memory, and otherwise. I found this saved me a large amount of time when chasing down bizarre bugs.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "C:\\code\\Hannibal\\Payload_Type\\hannibal\\hannibal\\agent_code\\Hannibal\\bin\\hannibal.exe",
            "args": [                
            ],
            "stopAtEntry": false,
            "cwd": "C:\\code\\Hannibal\\Payload_Type\\hannibal\\hannibal\\agent_code\\Hannibal",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "C:\\ProgramData\\mingw64\\mingw64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "Use GDB",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}

```

### Building Hannibal

You will notice there are three makefiles available. 

- windows_makefile - Creates a PIC .bin
- linux_makefile   - Creates a PIC .bin
- debug_makefile   - Creates a .exe with symbols/debug info (step this with GDB)

To build Hannibal on Windows, cd into the directory with the makefiles and issue ```make -f <makefile_name>```. If you want incremental compilation to work you will need to remove the clean and the del commands.

```makefile
$(BIN_DIR)/$(PROJECT).exe: clean $(ASM_OBJ_FILES) $(OBJ_FILES)
	@ echo "[+] Linking x64 Executable"
	@ $(CC_X64) bin/obj/*.o -o $(BIN_DIR)/$(PROJECT).exe $(CFLAGS) $(LDFLAGS)
	@python scripts/build.py -f $(BIN_DIR)/$(PROJECT).exe -o $(BIN_DIR)/$(PROJECT).bin
	@ del /q bin\obj\*.o 2>nul
	@ del /q bin\*.exe 2>nul
```

Hannibal reads configuration from **include/config.h**. If you change that header you will need to delete the **bin/obj/hannibal.o** file. If you change up the ifdefs to add/remove functionality, you will need to make sure the old .o files are not cached otherwise it won't rebuild with the new config. 

For ease of use I simply have it wipe all the .o files each time. But this forces a full rebuild and is slow. If you are just making changes to .c files then incremental recompilation will work, but headers require more effort.


#### PIC vs Debug

There are two ways to build Hannibal. Some of these details will be discussed later, but briefly this is enabled via:

In **include/hannibal.h**:

```c
#ifdef PIC_BUILD
#define HANNIBAL_INSTANCE_PTR PINSTANCE hannibal_instance_ptr = (PINSTANCE)*(PVOID*)((PVOID)((UINT_PTR)StRipStart() +  (UINT_PTR)&__Instance_offset));    
#else
#define HANNIBAL_INSTANCE_PTR extern PINSTANCE hannibal_instance_ptr;
#endif
```

In **src/hannibal.c**:

```c
////////////////////////////////////////////////// PIC BUILD
#ifdef PIC_BUILD

SECTION_CODE VOID Hannibal(
    _In_ PVOID Param
) {
    PINSTANCE hannibal_instance_ptr = (PINSTANCE)*(PVOID*)((PVOID)((UINT_PTR)StRipStart() +  (UINT_PTR)&__Instance_offset));

#else 

////////////////////////////////////////////////// DEBUG BUILD

// Global Variable Instance
INSTANCE hannibal_instance;
PINSTANCE hannibal_instance_ptr = &hannibal_instance;

int main(
    _In_ PVOID Param
) {
    
#endif

```

There are a few other places this is used, but effectively by using the power of [Preprocessor Macros](https://gcc.gnu.org/onlinedocs/cpp/Ifdef.html) we can choose what code gets included at build time.

The flag is passed in via the makefile: ```CFLAGS += -D PIC_BUILD -D PROFILE_MYTHIC_HTTP```.

Hannibal does require a newer MingW which is why the Dockerfile uses ```FROM python:3.11.10-bookworm``` as that is a newer Debian with an updated MingW.

Here is a screenshot of debugging Hannibal after built with debug_makefile. You can step the code and also issue commands to GDB in the **Debug Console**.

![hannibal_logo](/assets/img/hannibal_debug.png)

For testing, just generate a test payload in Mythic, copy the UUID, and any other param into Config.h, and recompile. The encryption key will need to be converted from base64 to a c-style byte array. Look in the Hannibal scripts folder for utilities to help with this.

## End Part 1

In the next part we will discuss agent architecture.

[Part 2](/posts/making-monsters-2)