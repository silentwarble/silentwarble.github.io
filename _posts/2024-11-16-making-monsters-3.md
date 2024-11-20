---
title: Making Monsters - Part 3
date: 2024-11-16 09:09:09 +/-TTTT
categories: [SECURITY, REDTEAM, MALDEV]
tags: [mythic, hannibal, maldev, c2, c, pic, position independent]     # TAG names should always be lowercase
description: Lessons learned and troubleshooting tips.
---


![hannibal_logo](/assets/img/hannibal_logo_dark_red.png)


## Introduction

Lets roundup this series and discuss a few items that illuminated themselves during this process.


## Lessons Learned

These are dependent on your development experience, resourcing, and so on. Always choose the right tooling for your particular requirements. These notes are specific to me at this time.

### PIC

While PIC coding provides several advantages, I do not think it is always the right choice for every agent. The limitations made for a very slow and sometimes painful development process. For example, I couldn't just call memcpy and move on. I have to write my own everything. Once you've gotten accustomed to it and have a base of functionality built, it's not as bad. Hannibal's design objectives made PIC C the correct tool for the job and I don't regret using it. 

If I didn't have those requirements, and development speed was essential, I would write non-PIC code in something like C#/C++. Then I would use tooling similar to [donut](https://github.com/TheWover/donut) or [srdi.py](https://github.com/monoxgas/sRDI/blob/master/Python/ConvertToShellcode.py).

### C

Additionally, C can also be a slow language to develop in. I spent a lot of time chasing down memory management issues. This may not have been as big of an issue if my C experience was more refined. If you are newer to the world of C and not having RAII protections, you will run into this.

### Mythic

While Mythic does significantly reduce development workload, it also introduces its own complexities. I've used Mythic in the past so this was less of an issue, but the translation containers were new for me. It took a good bit of time to figure out how all the pieces fit together and what to parse.

Another example was hooking into the various features such as file upload/download. Mythic handles this in its own way which had more of a design effect on my agent than I wanted.

Additionally, it is much more resource intensive than the teamservers for most of the other C2s I've used.

All that said, it was still the correct choice for this project and I'll likely continue using it. This is not a dig on Mythic, but I would be remiss not to mention difficulties I experienced with it.

Cheers to @its-a-feature!

## Troubleshooting

### PIC Limitations

As you develop, remember these limitations:

- No global variables
- Everything must be in the .text section
- Stack size is limited to 5k 
- No libc functions (such as malloc)

If something works in the debug exe but not the PIC .bin, then you violated one of these rules. 

The stack size one was particularly tricky. You can't exceed 5kb of data on the stack as it ends up clobbering return addresses. I am assuming this is due to us defining chkstk as just a return. See **stub_wrapper.asm** for more details. I am not 100% sure this is why, but it seems plausible. If that is not correct, please DM me so I can fix.

```asm
;; If not in this section it points to a bunch of zeros and crashes.
;; Also works in .text$CODE section. 
;; https://www.metricpanda.com/rival-fortress-update-45-dealing-with-__chkstk-__chkstk_ms-when-cross-compiling-for-windows/
;; https://nullprogram.com/blog/2024/02/05/
;; https://skanthak.hier-im-netz.de/msvcrt.html
___chkstk_ms:
    ret
```

Review slides 36 and on for more PIC information in this [talk](https://files.brucon.org/2021/PIC-Your-Malware.pdf).

### Memory Management

I found that simply watching the process in [Process Hacker](https://processhacker.sourceforge.io/downloads.php) helped me spot when there was a leak.

### PIC Debugging

I used [x64dbg](https://x64dbg.com/) and __debugbreak() extensively when debugging Hannibal in memory when executed as PIC shellcode.

### Mythic

I'm sure there's a better way, but for debugging any of the Python scripts invoked by Mythic I used exceptions. 

For example, when I needed to figure out what the json looked like at a certain point in translator.py, I would place a ```raise Exception(json_object)``` at that point. Then use ```./mythic-cli logs python_services``` to see what the output was. It was very clunky, but it got me through it. You have to restart the container after changing the Python scripts.

Also, all the normal Docker commands work for entering a container for internal troubleshooting.

```
docker ps -a
docker exec -it xxx /bin/bash
```

## Credits

I always take extra effort to ensure I give proper credit where it is due. If I missed any please DM me so I can get them added.

- https://github.com/MythicAgents/Apollo
- https://github.com/MythicAgents/Athena
- https://github.com/Cracked5pider/Stardust
- https://github.com/HavocFramework/Havoc/tree/main/payloads/Demon
- https://github.com/Cracked5pider/Ekko
- https://github.com/kokke/tiny-AES-c
- https://github.com/robertdavidgraham/whats-dec/
- https://github.com/zhicheng/base64

<br>
<div style="text-align: center; font-size: smaller; font-style: italic; font-family: 'Arial', sans-serif; 
            text-shadow: 0 0 10px rgba(0, 255, 255, 0.7), 0 0 20px rgba(0, 255, 255, 0.7);">
  It puts the stolen code on its skin or it gets detected again.
</div>

## References

- https://bruteratel.com/research/feature-update/2021/01/30/OBJEXEC/
- https://files.brucon.org/2021/PIC-Your-Malware.pdf
- https://web.archive.org/web/20201202085848/http://www.exploit-monday.com/2013/08/writing-optimized-windows-shellcode-in-c.html

## Errata

If you notice anything that is not accurate, or perhaps you have suggestions for improved methods, please contact me at any of my links. I am most active at [x.com](https://x.com/silentwarble).

## End Part 3, End Article