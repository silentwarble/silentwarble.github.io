---
title: Making Mini Monsters
date: 2024-11-16 09:09:09 +/-TTTT
categories: [SECURITY, REDTEAM, MALDEV]
tags: [mythic, hannibal, maldev, c2, c, pic, position independent, bof, hbin]     # TAG names should always be lowercase
description: Discussing a COFF alternative.
---


![hannibal_logo](/assets/img/hannibal_logo_pink.png)


## Introduction

In this article we explore one of the post-ex capabilities of the [Hannibal](https://github.com/silentwarble/Hannibal) agent, the "HBIN".

## The HBIN

### What

Simply put, it is a stripped down version of the Hannibal agent which itself is built on the [Stardust](https://github.com/Cracked5pider/Stardust) template. See [Making Monsters](/posts/making-monsters-1).

### Why

All the same reasons for why operators make use of BOF files for post-ex apply here. This serves the same purpose but is a different format. It was built as it avoided me having to implement a COFF parser into Hannibal. I may still do so at a later date, but I like how these turned out.

### How

We will use a stripped down version of Hannibal and treat it like a BOF file. Mythic has the ability to show a modal to the user and accept a BOF file plus the args. So we'll just use that for the controller side to get it to our agent.

Mythic will serialize the arguments and send all of it to the agent. Look at **mythic/agent_functions/execute_hbins.py** for the Mythic side. Once within hannibal, it is sent to **src/cmd_execute_hbin.c**.

#### Passing Args Challenge

Ok we have the hbin and the args passed all the way to our agent. 

An initial challenge was how was I going to pass arguments into the hbin? We aren't parsing a function export [go](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/beacon-object-files_how-to-develop.htm) like Cobalt Strike BOFs use.

The Stardust template Hannibal is built on includes a stub written in asm. With a function within that I am able to retrieve the address of the start of the agent. That address is accessible and usable within the agent. In fact it is heavily used to locate offsets.

I quickly realized it would be easy to just get that address and then subtract the size of a pointer (8 bytes) to go backwards in memory. What if I just stored a pointer to a struct before the start of the hbin? I could get the start of the agent, go backwards 8 bytes, and then deref to get a struct containing any arg I want.

#### Prep Arg Passing

Now that we have an idea of how to pass args into the hbin, lets look at how it was implemented.

Taking a look at the beginning of the function we see us craft a struct that will contain:

- Buffer of the serialized args
- Size of the arg buffer
- Pointer to Hannibal instance
- UUID of the execute_hbin task for Mythic

```c
typedef struct _HBIN_IN {
    LPVOID args;
    int arg_size;
    LPVOID hannibal_instance;
    char *controller_uuid;
} HBIN_IN;

HBIN_IN *in = (HBIN_IN *)hannibal_instance_ptr->Win32.VirtualAlloc(
    NULL, 
    sizeof(HBIN_IN *), 
    MEM_COMMIT, 
    PAGE_READWRITE
);

in->args = exec_hbin->args;
in->arg_size = exec_hbin->arg_size;
in->hannibal_instance = hannibal_instance_ptr;
in->controller_uuid = t.task_uuid;
```

Next we allocate a buffer large enough to hold the size of a pointer address of that struct plus the hbin binary itself.

```c
size_t buffer_size = sizeof(HBIN_IN*) + exec_hbin->hbin_size;

UINT8 *hbin_buff = (UINT8 *)hannibal_instance_ptr->Win32.VirtualAlloc(
    NULL, 
    buffer_size, 
    MEM_COMMIT, 
    PAGE_READWRITE
);
```

We copy the address to the struct holding pointers to the args and instance into that buffer first. We then append the actual hbin after that address.

```c
if(hbin_buff != NULL){
    pic_memcpy(hbin_buff, &in, sizeof(HBIN_IN*));
    pic_memcpy(hbin_buff + sizeof(HBIN_IN*), exec_hbin->hbin, exec_hbin->hbin_size);
}
```

At this point, the buffer looks like this in memory:

```
| 8 Bytes | sizeof(hbin) Bytes
| Address of HBIN_IN Struct | HBIN Bytes
```

#### Execute

We can now mark that buffer executable, set the execution pointer past the struct address, and execute.
```
DWORD OldProtection  = 0;
hannibal_instance_ptr->Win32.VirtualProtect( hbin_buff, buffer_size, PAGE_EXECUTE_READ, &OldProtection );

UINT_PTR exec = (UINT_PTR)hbin_buff + sizeof(HBIN_IN*);

typedef void (*exec_func)(); 
exec_func hbin_exec = (exec_func)exec;
hbin_exec();
```

#### Inside the HBIN

In order to retrieve the address of the struct that contains all the pointers we need, I added an additional command to the stub.

```asm
GetInStruct:
    call StRipStart ;; Get start of agent       
    sub rax, 8      ;; Go backwards 8 bytes
    ret             ;; Return a ptr to a ptr to the input struct

```

Now at our main function I can simply call this function which gives me the address of 8 bytes before the start of our hbin. So it is a ptr > ptr > struct. After a cast and deref, we now have a pointer we can use.

```c
EXTERN_C SECTION_CODE VOID hbin_main(
    _In_ PVOID Param
) {

    PVOID base = StRipStart();
    PVOID in_struct = GetInStruct();

    HBIN_IN *in = *(HBIN_IN **)in_struct;
    LPVOID arg_buff = in->args;
    int arg_buff_size = in->arg_size;

    PINSTANCE hannibal_instance_ptr = in->hannibal_instance;
```

From this point on it's effectively the same as everything else with Hannibal. We can use anything in the instance struct. I need to test this but I think I could also store the address of functions compiled into the Hannibal agent. That would enable me to reduce even more what gets compiled into the hbins and reduce the size.

## The HBIN Template

Other than removing everything but the bare essentials, hbins only have a few differences from the primary Hannibal agent. They are:

**asm/x64/stub_wrapper.asm**

```c
EXTERN hbin_main

GLOBAL GetInStruct

GetInStruct:
    call StRipStart ;; Get start of agent       
    sub rax, 8      ;; Go backwards 8 bytes
    ret             ;; Return a ptr to a ptr to the input struct
```

**include/hannibal.h**

```c
// HBIN INPUT
typedef struct _HBIN_IN {
    LPVOID args;
    int arg_size;
    LPVOID hannibal_instance;
    char *controller_uuid;
} HBIN_IN;

EXTERN_C PVOID GetInStruct();

EXTERN_C VOID hbin_main(
    _In_ PVOID Param
);

#ifdef PIC_BUILD
#define HANNIBAL_INSTANCE_PTR PINSTANCE hannibal_instance_ptr = (*(HBIN_IN **)(GetInStruct()))->hannibal_instance;
#else
#define HANNIBAL_INSTANCE_PTR extern PINSTANCE hannibal_instance_ptr;
#endif
```

<blockquote class="prompt-warning">
    <p>
        Make sure the struct definitions match between what you compiled into the agent and the hbin. If HBIN_IN or the Instance struct don't match you will crash.
        Another caution is, for any win32 function you call, ensure both the dll was loaded and the function was resolved. Otherwise you'll access a null pointer and crash.
    </p>
</blockquote>



## Advantages and Drawbacks

Advantages
<blockquote class="prompt-tip">
  <p>
    <li>Full power of the agent. Can use anything stored in the instance struct.</li>
    <li>No Win32 function limits.</li>
    <li>No COFF parser needed.</li>
    <li>Porting open source BOFs fairly easy.</li>
</p>
</blockquote>

Drawbacks
<blockquote class="prompt-danger">
  <p>
    <li>Large amount of Open Source BOFs available for use, have to write these yourself.</li>
    <li>Easy to forget to propigate struct changes from Hannibal to your hbins.</li>
    <li>All the position independent limitations still exist.</li>
</p>
</blockquote>

## End Article

As always, should you have questions or have suggestions for a better approach, please submit a PR or DM me on socials.