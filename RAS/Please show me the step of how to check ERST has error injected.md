To check if the **Error Record Serialization Table (ERST)** has logged an error after an injection, we need to follow specific steps involving the reading of error records from the table. ERST is designed to store error records in a non-volatile storage area (like NVRAM), and the steps to verify if an error has been injected and stored involve interacting with ERST’s functionality.

Here’s a breakdown of the steps to check whether ERST has an injected error:
### Step-by-Step Process to Check ERST for an Injected Error

#### 1. **Access ERST and Prepare for Error Record Handling**
- The system firmware (BIOS/UEFI) sets up the ERST during boot, making it available for the operating system.
- You can view ERST-related information using tools like **RWEverything** or through ACPI dump utilities that allow reading the contents of ACPI tables, including ERST.

#### 2. **Locate ERST Table Using RWEverything or Other Tools**
- Use **RWEverything** or a similar utility to dump the ERST table. You should see output similar to the following:
    
    Example ERST Dump:
    ![[Pasted image 20241015160324.png]]
This dump shows the **ERST table** that includes the serialization actions for reading and writing error records.

#### 3. **Check for Existing Error Records in Non-Volatile Storage**
- To check if an error record has been injected or logged in the ERST, we can issue the **ERST read actions** to retrieve the error record. The steps are handled by the firmware or the operating system.
    
    ERST read actions typically follow this sequence:
    
    1. **BEGIN_READ_OPERATION**: Initiates the read operation from the non-volatile storage (like NVRAM or SPI flash).
    2. **READ_ERROR_RECORD**: Reads the specific error record stored in ERST. If an error has been injected, this action retrieves it.
    3. **END_READ_OPERATION**: Completes the read operation.
    
    Example pseudo-code to check the ERST for an error:
![[Pasted image 20241015160346.png]]
The above pseudo-code illustrates how the system might interact with ERST to retrieve error records.

#### 4. **Verify the Error Details**
- After retrieving the error record from the ERST table, the details of the injected error can be reviewed. These details will include:
    
    - **Error Type**: Indicates the type of error injected (e.g., memory error, PCIe error).
    - **Error Severity**: Tells if the error is recoverable or uncorrectable.
    - **Error Address**: Identifies where the error occurred (e.g., memory address, PCIe device address).
    - **Error Source ID**: Points to the hardware source where the error originated.
    
    Example error details:
    ![[Pasted image 20241015160404.png]]
- The record shows that a PCIe error was logged with an uncorrected severity, and the error occurred at the specified address.

#### 5. **Check System Logs (Optional)**
- Depending on the system configuration, injected errors might also be logged into system logs (e.g., dmesg in Linux or Event Viewer in Windows).
- You can check these logs to see if any error injection events are logged. On Linux, for instance, running `dmesg | grep -i erst` or `dmesg | grep -i pci` might show error-related logs.

#### 6. **Clear Error Records (Optional)**

- If errors were found, you can clear the error record after reading it using ERST actions:
    1. **BEGIN_CLEAR_OPERATION**: Starts the process of clearing an error record.
    2. **CLEAR_ERROR_RECORD**: Deletes the error record from NVRAM.
    3. **END_CLEAR_OPERATION**: Completes the clear operation.

---
### Scenario: Checking for Injected PCIe Error in ERST

Here’s a practical scenario of checking for an injected PCIe error in the ERST:

1. **PCIe Error Injection**: Suppose a test environment injects a PCIe error using the APEI EINJ mechanism.
2. **Error Logging to ERST**: After injection, the firmware writes the error record to ERST. This might include details like error severity (e.g., uncorrected), the source ID (e.g., PCIe device ID), and the address where the error occurred.
3. **Read Error Record from ERST**: Using the ERST read actions, the OS or a diagnostic tool retrieves the error record from non-volatile storage.
4. **Analyze the Error**: The error details (PCIe address, severity, type) are reviewed by system administrators or diagnostics tools to understand the nature of the error.

---
### Conclusion

To check whether ERST has an error injected:

1. Dump the ERST table using tools like RWEverything to verify that the table is available and functioning.
2. Execute the sequence of actions defined in ERST to read error records (begin read operation, read error record, end read operation).
3. Analyze the retrieved error record to confirm if an error was injected (e.g., PCIe error, memory error).
4. Optionally, clear the error records if necessary.

This process ensures that the system can track injected errors and handle them accordingly for diagnostics and error handling.