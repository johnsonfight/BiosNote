
![[Pasted image 20241109151424.png]]


The output you provided is from the `lspci -vvv` command, showing the _Advanced Error Reporting_ (AER) capabilities and status registers for a PCIe device. Let’s break down each section:

### Overview of Fields

1. **UESta** (Uncorrectable Error Status): Shows the current status of uncorrectable errors.
2. **UEMsk** (Uncorrectable Error Mask): Masks (enables/disables) specific uncorrectable errors.
3. **UESvrt** (Uncorrectable Error Severity): Indicates which uncorrectable errors are considered "fatal."
4. **CESta** (Correctable Error Status): Shows the current status of correctable errors.
5. **CEMsk** (Correctable Error Mask): Masks specific correctable errors.
6. **AERCap** (Advanced Error Reporting Capabilities): Shows additional capabilities of the AER feature.
7. **HeaderLog**: Logs details about the most recent error.

### Detailed Explanation of Each Field

#### Uncorrectable Errors (UESta, UEMsk, UESvrt)

- **Uncorrectable errors** are errors that cannot be automatically corrected by the hardware. Let’s go through each uncorrectable error bit in detail:
    
    - **DLP**: Data Link Protocol error (not detected, `DLP-`)
    - **SDES**: Surprise Down Error (not detected, `SDES-`)
    - **TLP**: Transaction Layer Packet error (not detected, `TLP-`)
    - **FCP**: Flow Control Protocol error (not detected, `FCP-`)
    - **CmpltTO**: Completion Timeout error (not detected, `CmpltTO-`)
    - **CmpltAbrt**: Completion Aborted error (not detected, `CmpltAbrt-`)
    - **UnxCmplt**: Unexpected Completion (not detected, `UnxCmplt-`)
    - **RxOF**: Receiver Overflow error (not detected, `RxOF-`)
    - **MalfTLP**: Malformed Transaction Layer Packet (not detected, `MalfTLP-`)
    - **ECRC**: End-to-End CRC Error (not detected, `ECRC-`)
    - **UnsupReq**: Unsupported Request error (not detected, `UnsupReq-`)
    - **ACSViol**: Access Control Services Violation (not detected, `ACSViol-`)

Each error type has three registers:

- **UESta**: Indicates if the error is currently flagged.
- **UEMsk**: Masks (disables) specific errors. If an error is masked, it won’t be reported.
- **UESvrt**: Determines the severity of each error (`+` means the error is considered fatal or critical if it occurs).

In this example:

- `DLP+`, `SDES+`, `FCP+`, `RxOF+`, and `MalfTLP+` indicate that these errors are considered severe if they occur.
- The absence of a `+` in `UEMsk` means no uncorrectable errors are currently masked, meaning they are all enabled for reporting.

#### Correctable Errors (CESta, CEMsk)

- **Correctable errors** can be automatically corrected by the hardware and typically do not affect system stability. Here’s a breakdown of correctable errors shown:
    
    - **RxErr**: Receive Error (not detected, `RxErr-`)
    - **BadTLP**: Bad Transaction Layer Packet (not detected, `BadTLP-`)
    - **BadDLLP**: Bad Data Link Layer Packet (not detected, `BadDLLP-`)
    - **Rollover**: Error count rollover (not detected, `Rollover-`)
    - **Timeout**: Indicates a correctable timeout (not detected, `Timeout-`)
    - **AdvNonFatalErr**: Advisory Non-Fatal Error (detected, `AdvNonFatalErr+`)

Each correctable error type has:

- **CESta**: Status register for each correctable error type.
- **CEMsk**: Mask register to enable or disable specific correctable errors.

In this example:

- The only correctable error flagged as present is `AdvNonFatalErr+`, indicating an advisory (non-critical) error was detected.

#### Advanced Error Reporting Capabilities (AERCap)

- **First Error Pointer**: Points to the first uncorrectable error that occurred.
    - `First Error Pointer: 00` means no specific error has been recorded as the "first error."
- **ECRCGenCap**: End-to-End CRC Generation Capability (`ECRCGenCap-` means this device doesn’t support generating CRC).
- **ECRCGenEn**: End-to-End CRC Generation Enable (`ECRCGenEn-` means CRC generation isn’t enabled).
- **ECRCChkCap**: End-to-End CRC Checking Capability (`ECRCChkCap-` means CRC checking isn’t supported).
- **ECRCChkEn**: End-to-End CRC Checking Enable (`ECRCChkEn-` means CRC checking isn’t enabled).
- **MultHdrRecCap**: Multiple Header Recording Capability (`MultHdrRecCap-` means it doesn’t support recording multiple headers).
- **MultHdrRecEn**: Multiple Header Recording Enable (`MultHdrRecEn-` means multiple header recording is not enabled).
- **TLPPfxPres**: Transaction Layer Packet Prefix Present (`TLPPfxPres-` means there is no TLP prefix).
- **HdrLogCap**: Header Logging Capability (`HdrLogCap-` indicates the device doesn’t support additional header logging).

#### HeaderLog

- This field logs information about the most recent error. If an error occurs, the AER Header Log captures details about the transaction that caused the error.
    - In this example, `HeaderLog: 00000000 00000000 00000000 00000000` means no errors were logged recently.
