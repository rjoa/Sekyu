---
layout: post
title: Analysis of Konni APT Campaign Targeting South Korea
date: '2025-04-05 15:35:03 +0800'
categories: [Cyber Security, Malware Analysis]
---

As I was scrolling through Twitter/X yesterday, I saw a post about an ongoing campaign by the Konni APT group. Since I dont have anything better to do and my curiousity got the best of me, I decided to take a look :D.

Upon searching the hash from the post into VirusTotal, the initial file is a Microsoft shortcut (.LNK). To trick users into thinking that the it is Hanword document file, it used the double extension **".hwp.lnk"**. The usage of *".hwp"* strongly suggests that this is targeted towards those in South Korea. If you're not familiar with HWP, it's a document format for Hangul Word Processor – think of it like South Korea's Microsoft Word's .doc or .docx files. Taking a quick look at the LNK revealed that it contnains two scripts. One javascript and the other one is a PowerShell script. When the the LNK is executed, it will execute the javascript first.

![img-description](/images/2025-04-05/1.png)
_figure 1. Content of LNK file when viewed opened on a hex viewer_

The purpose of the javascript is to form a PowerShell script whose purpose is to look for the original LNK file. It does so by iterating through files and checking if the file size is **6699** (**0x1a2b**) bytes. Once it finds the target file, it start to read the embedded PowerShell script at **0x942** and execute it aferwards.

![img-description](/images/2025-04-05/2.png)
_figure 2. Beautified PowerShell Script_

![img-description](/images/2025-04-05/3.png)
_figure 3. Embedded PowerShell Script at 0x942_

The PowerShell script contains multiple base64 encoded strings and when decoded, produces another PowerShell script that download **Sm.dat** from the Dropbox link and writes it to `C:\programData\gs.zip`{: .filepath}. This archive contains two files: 26535.tmp and **AN9385.tmp**. Since **26535.tmp** acts as a loader, the malware creates persistence for it by creating both a scheduled task and an entry in *HKCU\Software\Microsoft\Windows\CurrentVersion\Run* registry key. After that, it will execute **AN93585.tmp** as this where the backdoor routine resides.

![img-description](/images/2025-04-05/4.png)
_figure 4. PowerShell Downloader_


# Analysis of 26535.tmp

By the look if it, the purpose of this javascript is to form another script. To see what it's trying do, we can write the content of *res* variable to a file or use the built-in debugger of our browsers (I chose the latter since it's much simpler). We can skip all of the other instructions by setting a breakpoint on the line where *res* is executed using *eval()*. Dumping the variable will show that it is another Javascript-powershell combo and is used to execute **AN93585.tmp**

![img-description](/images/2025-04-05/5.png)
_figure 5. Content of 26535.tmp_

![img-description](/images/2025-04-05/6.png)
_figure 6. Content of Res Variable_

# Analysis of AN93585.tmp

The backdoor code is obfuscated using Base64 and is only assembled and decoded by the PowerShell script at runtime. To bypass this, we can paste the script into a PowerShell console. After it runs, simply typing the variable name that holds the concatenated strings (*$ik68*) at the PowerShell terminal will display its content.

![img-description](/images/2025-04-05/7.png)
_figure 7. Content of AN93585.tmp_

The decoded string is the backdoor, written in PowerShell. It uses Google Drive as its C&C server. It does this by listing files in a specific Drive folder and searching for any with the "text/plain" mime type. If it finds one, it downloads that file and saves it to `C:\programdata\tmps4.ps1`{: .filepath}. This downloaded script is then executed using the line `powershell -ep bypass -f $tmpz 2>&1`, where $tmpz contains the path to the script, and it uses 2>&1 to redirect any errors from stderr to stdout. The output is then saved to a temporary file stored in **C:\programdata\** and uploaded to the C&C server (Google Drive).

![img-description](/images/2025-04-05/9.png)
_figure 8. Backdoor Routine_

It's getting more common to see malware using popular sites like Dropbox and Google Drive to hide its tracks. By mixing malicious communication in with everyday traffic to these services, the malware becomes harder to spot and avoids raising immediate red flags.
