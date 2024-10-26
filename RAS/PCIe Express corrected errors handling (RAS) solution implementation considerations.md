
[# PCIe Express corrected errors handling (RAS) solution implementation considerations](https://www.youtube.com/watch?v=2kxbPCf1lgg&t=322s&ab_channel=OpenComputeProject)

![[Pasted image 20241015105038.png]]
* 2 CPU with multiple PCIe root ports, communicating with PCIe gen 5 switches, 
  -> that break out the PCIe bus to many devices(NIC, drives, accelerator)
* The system has tons of receivers which have AER capability, 
  -> reporting error back to root port


![[Pasted image 20241015105805.png]]
A slice of one CPU, showing many devices (NIC, SSD, GPU) transferring tons of data at gen 5 speed. 
-> have to ensure RAS of these devices


![[Pasted image 20241015110810.png]]
Anytime a correctable error happens (hardware), 
-> SMI triggered (handled through firmware)
-> notify to the OS through SCI System Control Interrupt

Both correctable and uncorrectable error use SMI.
-> becomes a problem.
-> need a solution that correctable error do not trigger SMI


![[Pasted image 20241015111609.png]]
Proposed solution to overcome SMI limitation
-> MSI

Solution:
-> Uncorrectable, still SMI flow
-> Correctable, using MSI (Message Signaled Interrupts)
-> MSI goes to OS directly

Next challenge: goes to BMC(SEL) to log error.
How does OS writes to the BMC?
-> Ras daemon (open source software)

