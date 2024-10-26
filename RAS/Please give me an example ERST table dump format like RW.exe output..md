
Here is an example **ERST (Error Record Serialization Table)** dump format that resembles the style of output you would see from a tool like **RW Everything (RW.exe)**. This format shows the major fields of an ERST table as per the ACPI specification, along with a sample of error serialization actions. The table format may vary depending on the system and implementation, but this example follows the general layout.

### Example ERST Table Dump Format
![[Pasted image 20241015154342.png]]
![[Pasted image 20241015154355.png]]

### Breakdown of the ERST Table Dump

- **Signature**:
    - Identifies the table as an ERST table with the signature "ERST."
- **Length**:
    - The total size of the ERST table in bytes (168 bytes in this example).
- **Revision**:
    - Indicates the revision of the ERST specification being used.
- **Checksum**:
    - A checksum of the table used for validation, ensuring that the table's content is accurate and complete.
- **OEM ID, OEM Table ID, OEM Revision**:
    - Identifies the OEM (Original Equipment Manufacturer) and the version of the table.
- **Creator ID, Creator Revision**:
    - Identifies the entity that created the table (in this case, ACPI) and the revision of the creator.
---

### **Serialization Header Section**

This section contains metadata about the error record serialization instructions.
- **Serialization Header Length**:
    - The size of the serialization header, 48 bytes in this example.
- **Instruction Entry Count**:
    - The number of **Serialization Instruction Entries** that follow. In this case, 3 entries are listed.
---

### **Serialization Instruction Entries**

These define the steps that the system takes when performing error record serialization tasks, such as writing, reading, or managing error records. Each instruction has an associated action and register operation.

- **Action**:
    - Describes what kind of action is being performed, such as "Begin Write," "Write to Persistent Storage," or "Complete Write."
- **Instruction**:
    - Specifies the type of operation to be performed. In the example, the **Write Register** operation is used to perform various actions like starting or completing a write operation.
- **Flags**:
    - Flags control how the instruction behaves, such as whether the current register value should be preserved.
- **Register Region**:
    - Describes the memory-mapped address space where error records are being stored or manipulated. This is represented using the **Generic Address Structure (GAS)**, which defines:
        - **Space ID**: Specifies the address space (e.g., 0x00 for system memory, 0x01 for NVRAM).
        - **Register Bit Width**: The size of the data being accessed (64 bits in this case).
        - **Register Bit Offset**: Offset in the register for the data access (0 in this example).
        - **Access Size**: Access mode, such as **Qword** (64-bit access).
        - **Address**: The physical address where the action occurs (e.g., `0xFED9F000`, `0xFED9F100`).
- **Value to Write**:
    - The value that is written to the register when executing the instruction.
- **Mask**:
    - A bitmask to apply when writing values to the register. `0xFFFFFFFFFFFFFFFF` indicates all bits are written.

---

### **Error Log Area Range**

This section defines the memory region where error records are stored.

- **Error Log Address Range**:
    - The start and end addresses of the error log storage area. This is where error records are written and retrieved.
- **Maximum Error Data Length**:
    - Defines the maximum size (in bytes) of each error data entry. In this case, 512 bytes is the limit.
---

### **How ERST Works**

In this example, the ERST table outlines the process by which the platform handles and stores hardware error records. The steps typically include:

1. **Begin Write Operation**:  
    The first instruction begins the error record write process by writing to a specific memory address (`0xFED9F000`).
    
2. **Write to Persistent Storage**:  
    The second instruction writes the error record into non-volatile storage (NVRAM) at address `0xFED9F100`, ensuring the record is retained even after system restarts.
    
3. **Complete Write Operation**:  
    The final instruction completes the write process by writing a final value to `0xFED9F200`, marking the error record as fully serialized.
---

### Conclusion

The **ERST table** provides a robust mechanism for persisting hardware error records, which can be crucial for debugging and analyzing system failures. The example above illustrates the basic structure and how various fields are utilized to manage the serialization of error records. The **Error Status Address** fields are crucial for accessing error logs, and instructions outline the exact steps taken during the error handling process.
