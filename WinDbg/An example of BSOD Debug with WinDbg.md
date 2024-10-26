
![[Pasted image 20241026113634.png]]



![[Pasted image 20241026113846.png]]



![[Pasted image 20241026113852.png]]


Step 0: See the bug check code we're going to investigate. Here we check 0xD1 first.


Step 1: **!analyze -v**
And we see the address in **Arg4** called/referenced **Arg1**.
BSOD occurred on **Arg1 ffffde00a8fa2050**
![[Pasted image 20241026113858.png]]



Step 2: Tracing to the Arg4 address,
we see "**storport!RaidpAdapterDpcRoutine+0xd2**", above/after parts are BSOD showing stuff that we can ignore.
We can see some clue that in idle state, DPC routine triggered and failed, causing BSOD.
![[Pasted image 20241026120823.png]]



Step3: We check the disassembly of "**RaidpAdapterDpcRoutine**"
(L35: List 35 lines)

It seems it wanted to move content in **r15** to **rax**, but then error occurred.
![[Pasted image 20241026122721.png]]



Step4: So we check r15(Arg1) **!pool**, memory check.
The command checks that memory is not accessible.

![[Pasted image 20241026131738.png]]




Step5: We move back to memory layout overview with **!address** (like UEFI memmap).

Also !pte
![[Pasted image 20241026131957.png]]



Step6: Back to **!analyze -v**, we found the inconsistency of **Arg1(xxxx2050)** and **r15(00000000)**.
Which causes the problem?
![[Pasted image 20241026132225.png]]

At least for now we know, memory crashed, with certain register.
But who is the murderer, that's what we need to investigate further.


Step7: Our direction goes to, what (something) is/are related to these memory?
The memory should be allocated by "something".
And the murder write/modify the memory causing error.

So we use disassembly to check this,
and found "**r15**" is from the content of "**rsi+100h**",
and **rsi** is from the content of "**rdx+40h**"

Noted that, according to the X64 calling convention,
"**rdx**" is the 2nd parameter of the function.

![[Pasted image 20241026132426.png]]



Step8: Now our purpose is to trace **rdx**, and try to understand what is this memory used for. We need trying to restore **rdx** value, and check whether **rdx** has been modified at any moment.
If we're lucky, **rdx** may keep the same value as it passed in.

We check trap_frame.
Is the **rdx=xxx8e050** correct? That's what we need to make sure.

We use db cmd to check the content, but, meh, no any useful clue. 

![[Pasted image 20241026133207.png]]




Step9: So we check "**rdx+40h**", which is "**rsi**"
and print 120 lines of "**rsi**" content.
We found the "RTDA" signature at "**rsi(e1a0)**"! though we don't know what it is.

![[Pasted image 20241026133926.png]]



Step10: At here we see "**e1a0+0x100h**" which is **r15**, and the value of **r15** is the same as **Arg1**.
But in trap_frame the value is 0.

![[Pasted image 20241026134409.png]]




Step11: Let's guess what the hell is the **rdx**.
Since this is a DPC OS routine. There must be something register this callback and call it.
We check DPC implementation from MSDN.
Just like BIOS driver, the second parameter is usually "context".
But the below one is special, the 2nd parameter is DeviceObject.
Let check is it device object.

![[Pasted image 20241026135000.png]]


Step12: We use cmd "**dt**" following with NT Device_Object data structure name.
Well, at "+0x040" seems no problem.

btw, we can assume the source C code would be like,
DeviceObject->DeviceExtension.

Anyways, we need more evidence.
![[Pasted image 20241026135437.png]]

We click the "blue links"


Step13: Info from those links seems fine.
But at least we make sure at "+0x018", the parameter it passed in, IS a device object, and it uses it fine.

![[Pasted image 20241026135908.png]]



Step14: Let's move on to **rdx+40h**, as we remember there's a RTDA signature.
We try to dump all types. Print all type symbols.
Finally, **RaidAdapterObject = 0x41445452**, is RTDA.
It means, it is an device object, generated by a certain driver.
This data is similar to the private data that when EFI driver connecting.
![[Pasted image 20241026140248.png]]



Step15: We also dump RAID_ADAPTER_EXTENSION. 
The **r15(rsi+100h)** is called **SaveXrbList**, which is a list entry.
And the list entry is sabotaged. That's why the error occurred.

We also use **!poaction** (power action) and **!powertriage**,
we found at this moment, the system already set S3.
Lots of issues happened when system goes to S3.
Because entering and exiting S3, devices need to be disconnected or connected,
or device RTD3 (runtime D3) on/off.
If any sequence is wrong, 

![[Pasted image 20241026141258.png]]



Step16: Besides, once we know the object, we can use 3 cmd, **!devstack**, **!devobj**, **!drvobj**, to check what is/are related to this driver.
We found it's related to mpi3drv.
And **!drvobj mpi3** shows it generated 3 objects (f1ba2050, f1b8b050, f1b8e050).
![[Pasted image 20241026142302.png]]


Here's the wrap up:
We know the corrupted memory and related data, are generated by Broadcom driver.
The error it generated is in the data structure, +100h offset field of the device extension, the RAID card extension.

![[Pasted image 20241026143235.png]]

Broadcom RAID driver might be under the Microsoft RAID arch, that might use Microsoft's data structure.



Some recaps, notes and takeaways:
![[Pasted image 20241026143759.png]]


- For report from **!analyze -v**, we need to double verified with other info. 
- "Who" actually generates the crash memory is important.
- BSOD problem usually comes from the parameter of the environment. So, again, "who" generates the parameter is important.
![[Pasted image 20241026143846.png]]



![[Pasted image 20241026144324.png]]
