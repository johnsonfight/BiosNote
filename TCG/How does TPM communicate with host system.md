How does TPM communicate with host system? For example for reading and sending cmd above shown, what protocol do they use?

### TPM Communication Interfaces

The physical layer or bus over which the TPM communicates with the host system can vary depending on the system architecture. Below are the most commonly used interfaces:

1. **Low Pin Count (LPC) Interface:**
    
    - **Discrete TPMs** on many systems (especially older ones) communicate with the host over the **LPC bus**.
    - LPC is a legacy interface used for low-bandwidth communications between peripheral devices and the motherboard.
    - In the case of the TPM, LPC is often used because it is simple and does not require high-speed communication for TPM functions.
2. **SPI (Serial Peripheral Interface):**
    
    - Newer systems may use the **SPI bus** for TPM communication, especially in embedded systems and modern platforms.
    - SPI is faster and more efficient than LPC, allowing for quicker data exchanges between the host and the TPM.
    - Discrete TPMs like the TPM 2.0 modules use SPI for more modern implementations.
3. **I2C (Inter-Integrated Circuit):**
    
    - In embedded systems, especially for firmware-based TPMs (fTPM), the **I2C** bus might be used.
    - This is commonly found in ARM-based or mobile platforms that have a TPM for security functions.
4. **Memory-Mapped I/O (MMIO) Interface:**
    
    - Firmware TPMs (fTPMs) that are implemented as part of the system’s firmware (often inside the CPU) can use **MMIO** for communication.
    - The TPM registers are memory-mapped, and the host communicates with the TPM by reading from and writing to specific memory addresses.
    - This is common in systems that implement TPM functionality in firmware, such as AMD's fTPM or Intel's Platform Trust Technology (PTT).
5. **PCIe (Peripheral Component Interconnect Express):**
    
    - In some newer systems, the TPM may communicate via the **PCIe bus**. This can be particularly true for integrated TPM modules on modern chipsets.
6. **USB Interface:**
    
    - In some special cases, external TPM modules or tokens (often used in secure environments) may use **USB** as a communication channel. This is less common for standard PC platforms but may appear in specific high-security applications.

### Communication Stack Overview

The communication between the host system and TPM follows a layered stack:

- **Physical Layer (LPC, SPI, MMIO, etc.)**: This defines the actual bus or connection between the host and the TPM hardware.
- **Transport Layer (TPM TIS or PTP)**:
    - **TPM Interface Specification (TIS)**: This is an older standard that defines the protocol for TPM 1.2 and early TPM 2.0 communication.
    - **Platform TPM Profile (PTP)**: PTP is an updated specification that replaces TIS, used for TPM 2.0, and improves the communication protocol.
    - These layers define how the host packages commands and how responses are sent back from the TPM.
- **Command/Response Protocol (TCG TPM Protocol)**: This is the logical communication protocol defined by TCG, which specifies the structure of commands and responses. Each command sent to the TPM includes fields such as `commandCode`, `tag`, and `commandSize`, followed by any specific parameters required for that command.
  
  
  

[[Any reason,benifit,limitation that why does fTPM use I2C instead of SPI]]
