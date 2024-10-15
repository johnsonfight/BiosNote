In **Advanced Error Reporting (AER)**, errors are classified into three main categories, with specific error types under each category. Here’s a list of all the error types for AER:

### 1. **Correctable Errors**:

These are errors that the system can automatically recover from without interrupting normal operations. They do not require software intervention beyond logging.

- **Receiver Error**
- **Bad TLP (Transaction Layer Packet)**: Malformed or corrupted transaction layer packets.
- **Bad DLLP (Data Link Layer Packet)**: Malformed or corrupted data link layer packets.
- **Replay Num Rollover**: Error related to replaying data packets.
- **Replay Timer Timeout**: Time exceeded for retrying a transmission.
- **Advisory Non-Fatal Error**: Non-fatal error that does not prevent successful transaction completion.

### 2. **Uncorrectable Non-Fatal Errors**:

These errors cannot be corrected, but they do not immediately cause a system-wide failure. They might require driver or OS intervention to handle.

- **Undefined Non-Fatal Error**
- **Data Link Protocol Error**: Errors in the data link protocol layer.
- **Poisoned TLP**: A corrupted transaction layer packet marked as poisoned.
- **Flow Control Protocol Error**: Issues with flow control, such as credit limit violations.
- **Completion Timeout**: Timeout waiting for a completion response.
- **Completer Abort**: The completer device could not handle the requested transaction and aborted it.
- **Unexpected Completion**: A completion packet was received without a corresponding request.
- **Receiver Overflow**: The receiving device’s buffer was overloaded.
- **Malformed TLP**: The transaction layer packet is malformed or corrupted.
- **ECRC (End-to-End CRC) Error**: Failure in the End-to-End Cyclic Redundancy Check, indicating packet corruption.
- **Unsupported Request Error**: The device received a request it doesn’t support.

### 3. **Uncorrectable Fatal Errors**:

These errors are severe and may cause the system or PCIe device to stop functioning, potentially requiring a system reboot or reset.

- **Undefined Fatal Error**
- **Surprise Down Error**: Unexpected loss of connection, typically due to device removal or failure.
- **Poisoned TLP** (if marked as fatal in this case)
- **Malformed TLP** (if marked as fatal in this case)
- **Flow Control Protocol Error** (if marked as fatal)
- **Completer Abort** (if marked as fatal)

Each of these error types provides information on the nature and severity of errors in the PCIe system, helping in error diagnosis and recovery through AER.