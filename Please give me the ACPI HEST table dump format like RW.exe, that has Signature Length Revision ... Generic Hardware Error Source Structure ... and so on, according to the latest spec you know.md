The **ACPI Hardware Error Source Table (HEST)** provides detailed information about the hardware error sources in a system. Below is an example of the format of an ACPI **HEST** table dump, such as what might be seen using a tool like **RWEverything (RW.exe)**. The output includes key fields from the HEST header, followed by entries that describe individual hardware error sources, based on the latest ACPI specification.

This example demonstrates how the **HEST** table dump might appear and is simplified for illustrative purposes:

HEST Table Dump Format:
![[Pasted image 20241015114632.png]]
![[Pasted image 20241015114828.png]]
![[Pasted image 20241015114849.png]]
![[Pasted image 20241015114906.png]]
![[Pasted image 20241015114923.png]]
![[Pasted image 20241015114938.png]]
![[Pasted image 20241015114957.png]]

### **Field Breakdown**:

1. **Error Source Type** (`0x06` for PCIe AER): Each PCIe device error source (GPU, SSD, NIC, chipset) uses the **PCIe AER (Advanced Error Reporting)** mechanism, represented by `Error Source Type` 0x06.
    
2. **Error Source ID**: Unique ID assigned to each error source:
    
    - **Error Source ID: 0x0003** (PCIe GPU, NVIDIA A100)
    - **Error Source ID: 0x0004** (PCIe SSD, NVMe)
    - **Error Source ID: 0x0005** (NIC, 10GbE Dual-Port)
    - **Error Source ID: 0x0006** (Chipset)
3. **Flags**:
    
    - **Firmware First** (`0x01`) indicates that the firmware (UEFI/BIOS) will handle the error before passing control to the OS.
4. **Error Status Address**: Each device has a **status address** that points to where error information is stored. These are mapped to system memory (with 64-bit register width) for each PCIe device:
    
    - **PCIe GPU (NVIDIA A100)** at `0xFED90000`.
    - **PCIe SSD (NVMe)** at `0xFED80000`.
    - **NIC (10GbE)** at `0xFED70000`.
    - **Chipset** at `0xFED60000`.
5. **Notification Structure**:
    
    - **Notification Type**: SCI (System Control Interrupt) (`0x04`) is used to notify the OS about hardware errors via interrupt, ensuring the errors are handled promptly.
6. **Max Error Data Length**: This defines the size of the error record data (128 bytes in this case), which holds the details of the PCIe errors for each device.
    

### **Error Scenario Description**:

- **PCIe GPU (NVIDIA A100)**: An uncorrectable non-fatal error occurs during computation, triggering the **AER** mechanism to log the error at `0xFED90000` and notify the OS using an SCI interrupt.
- **PCIe SSD (NVMe)**: The SSD encounters a PCIe transaction issue (e.g., completion timeout). The error is logged in memory at `0xFED80000` and reported to the OS.
- **NIC (10GbE Dual-Port)**: A network card error (e.g., a PCIe parity error or device malfunction) is detected, and the error is logged at `0xFED70000`.
- **Chipset**: The chipset generates its own PCIe AER error, possibly due to a system-level











