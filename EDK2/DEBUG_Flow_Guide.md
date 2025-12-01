# EDK2 DEBUG() Macro: Complete Flow from Input to Serial Output

## Table of Contents
1. [Overview](#overview)
2. [Layer Architecture](#layer-architecture)
3. [Complete Flow Diagram](#complete-flow-diagram)
4. [Layer 1: DEBUG Macro (DebugLib.h)](#layer-1-debug-macro-debuglibh)
5. [Layer 2: DebugPrint Implementation (BaseDebugLibSerialPort)](#layer-2-debugprint-implementation-basedebuglib)
6. [Layer 3: Serial Port Library (SerialPortLib)](#layer-3-serial-port-library-serialportlib)
7. [Layer 4: Hardware I/O (IoLib/MmioLib)](#layer-4-hardware-io-iolib)
8. [Configuration via PCDs](#configuration-via-pcds)
9. [Example Trace](#example-trace)
10. [Alternative Implementations](#alternative-implementations)

---

## Overview

The DEBUG() macro in EDK2 provides a flexible debug logging system that outputs messages to various destinations. This guide traces the **most common path**: DEBUG messages going to a serial port (UART) for hardware output.

**Key Design Principle**: The system uses a layered abstraction where:
- **Interface** is defined once in headers
- **Implementation** is selected at build time
- **Hardware access** is abstracted through libraries

---

## Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: User Code                                          │
│ DEBUG((DEBUG_INFO, "Hello World\n"));                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: DEBUG Macro (MdePkg/Include/Library/DebugLib.h)    │
│ - Compile-time filtering (MDEPKG_NDEBUG)                    │
│ - Runtime filtering (DebugPrintEnabled)                     │
│ - Error level checking (DebugPrintLevelEnabled)             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: DebugPrint() Implementation                        │
│ (MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c)          │
│ - Format string processing (PrintLib)                       │
│ - Buffer management                                         │
│ - Calls SerialPortWrite()                                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 4: SerialPortLib Implementation                       │
│ (MdeModulePkg/Library/BaseSerialPortLib16550/)              │
│ - UART register access                                      │
│ - Hardware flow control                                     │
│ - FIFO management                                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 5: Hardware I/O (IoLib or MmioLib)                    │
│ - IoWrite8() for I/O port access (x86)                      │
│ - MmioWrite8() for memory-mapped I/O (ARM, x64)             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 6: Physical Hardware                                  │
│ - UART controller (16550, PL011, etc.)                      │
│ - RS-232 serial port output                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Complete Flow Diagram

```
User Code: DEBUG((DEBUG_INFO, "Value = %d", x));
    │
    ├─── Compile Time Check: Is MDEPKG_NDEBUG defined?
    │    ├─── YES → DEBUG expands to do { if (FALSE) { ... } } while (FALSE)
    │    │           Compiler optimizes away (dead code elimination)
    │    │           *** FLOW ENDS HERE ***
    │    │
    │    └─── NO → DEBUG expands to:
    │             do {
    │               if (DebugPrintEnabled()) {
    │                 _DEBUGLIB_DEBUG(Expression);
    │               }
    │             } while (FALSE)
    │
    ├─── Runtime Check 1: DebugPrintEnabled()
    │    └─── Checks: PcdDebugPropertyMask & DEBUG_PROPERTY_DEBUG_PRINT_ENABLED
    │         ├─── FALSE → *** FLOW ENDS HERE ***
    │         └─── TRUE → Continue
    │
    ├─── Macro Expansion: _DEBUGLIB_DEBUG((DEBUG_INFO, "Value = %d", x))
    │    └─── Expands to: _DEBUG_PRINT(DEBUG_INFO, "Value = %d", x)
    │
    ├─── Runtime Check 2: DebugPrintLevelEnabled(DEBUG_INFO)
    │    └─── Checks: DEBUG_INFO & PcdFixedDebugPrintErrorLevel
    │         ├─── FALSE → *** FLOW ENDS HERE ***
    │         └─── TRUE → Continue
    │
    ├─── Function Call: DebugPrint(DEBUG_INFO, "Value = %d", x)
    │    Location: BaseDebugLibSerialPort/DebugLib.c:67
    │    │
    │    ├─── Convert varargs to VA_LIST
    │    └─── Call DebugVPrint(ErrorLevel, Format, Marker)
    │
    ├─── Function: DebugVPrint()
    │    Location: BaseDebugLibSerialPort/DebugLib.c:151
    │    └─── Call DebugPrintMarker(ErrorLevel, Format, VaListMarker, NULL)
    │
    ├─── Function: DebugPrintMarker()
    │    Location: BaseDebugLibSerialPort/DebugLib.c:98
    │    │
    │    ├─── ASSERT(Format != NULL)
    │    │
    │    ├─── Runtime Check 3: ErrorLevel & GetDebugPrintErrorLevel()
    │    │    └─── If zero → return (*** FLOW ENDS HERE ***)
    │    │
    │    ├─── Allocate Buffer[MAX_DEBUG_MESSAGE_LENGTH] (256 bytes)
    │    │
    │    ├─── Format String: AsciiVSPrint(Buffer, sizeof(Buffer), Format, VaListMarker)
    │    │    └─── "Value = 123" is written to Buffer
    │    │
    │    └─── Call SerialPortWrite((UINT8 *)Buffer, AsciiStrLen(Buffer))
    │         Location: BaseDebugLibSerialPort/DebugLib.c:131
    │
    ├─── Function: SerialPortWrite()
    │    Location: BaseSerialPortLib16550/BaseSerialPortLib16550.c:606
    │    │
    │    ├─── Get Hardware Base Address: GetSerialRegisterBase()
    │    │    Location: Line 196
    │    │    │
    │    │    ├─── Check PcdSerialPciDeviceInfo
    │    │    │    ├─── If 0xff → Use fixed address PcdSerialRegisterBase
    │    │    │    └─── Else → Walk PCI bus to find UART device
    │    │    │         ├─── Walk PCI bridge chain (lines 240-310)
    │    │    │         ├─── Find UART device (lines 315-355)
    │    │    │         ├─── Enable I/O or MMIO in PCI command register
    │    │    │         └─── Return BAR address
    │    │    │
    │    │    └─── Return SerialRegisterBase (e.g., 0x3F8 for COM1)
    │    │
    │    ├─── Determine FIFO Size (lines 646-655)
    │    │    └─── Based on PcdSerialFifoControl (1, 16, or 64 bytes)
    │    │
    │    └─── For each byte in buffer (lines 658-681):
    │         │
    │         ├─── Wait for UART ready (LSR.TEMT & LSR.TXRDY both set)
    │         │    Location: Line 663
    │         │    └─── Read R_UART_LSR (offset 5) via SerialPortReadRegister()
    │         │
    │         ├─── Fill FIFO (up to FifoSize bytes) (lines 669-680)
    │         │    │
    │         │    └─── For each byte:
    │         │         │
    │         │         ├─── Check hardware flow control (lines 673-674)
    │         │         │    └─── SerialPortWritable() checks MSR.DSR & MSR.CTS
    │         │         │
    │         │         └─── Write byte to UART (line 679)
    │         │              └─── SerialPortWriteRegister(Base, R_UART_TXBUF, *Buffer)
    │         │
    │         └─── Repeat until all bytes written
    │
    ├─── Function: SerialPortWriteRegister()
    │    Location: BaseSerialPortLib16550/BaseSerialPortLib16550.c:107
    │    │
    │    ├─── Check PcdSerialUseMmio
    │    │    │
    │    │    ├─── TRUE (Memory-Mapped I/O - ARM, x64)
    │    │    │    │
    │    │    │    ├─── Calculate Address:
    │    │    │    │    Address = Base + Offset * PcdSerialRegisterStride
    │    │    │    │    Example: 0x9000000 + 0 * 4 = 0x9000000
    │    │    │    │
    │    │    │    ├─── Check PcdSerialRegisterAccessWidth
    │    │    │    │    ├─── 32-bit → MmioWrite32(Address, Value)
    │    │    │    │    └─── 8-bit → MmioWrite8(Address, Value)
    │    │    │    │
    │    │    │    └─── MmioWrite8/32() performs:
    │    │    │         └─── *(volatile UINT8/32 *)Address = Value
    │    │    │
    │    │    └─── FALSE (I/O Port - x86)
    │    │         │
    │    │         ├─── Calculate Port:
    │    │         │    Port = Base + Offset * PcdSerialRegisterStride
    │    │         │    Example: 0x3F8 + 0 * 1 = 0x3F8 (COM1 TX)
    │    │         │
    │    │         └─── IoWrite8(Port, Value)
    │    │              └─── Assembly instruction: OUT DX, AL
    │    │
    │    └─── Return Value
    │
    └─── Hardware Level:
         │
         ├─── UART Controller (16550 Compatible)
         │    │
         │    ├─── Receive byte in Transmit Holding Register (THR)
         │    ├─── Transfer to Transmit Shift Register (TSR)
         │    ├─── Serialize data with start bit, data bits, parity, stop bit
         │    └─── Output serial data on TX pin
         │
         └─── RS-232 Level Converter
              └─── Converts TTL to RS-232 voltage levels (-12V to +12V)
                   └─── Serial output visible on terminal (COM port)
```

---

## Layer 1: DEBUG Macro (DebugLib.h)

**Location**: `MdePkg/Include/Library/DebugLib.h:434-448`

### Macro Definition

```c
#if !defined (MDEPKG_NDEBUG)
  #define DEBUG(Expression)        \
    do {                           \
      if (DebugPrintEnabled ()) {  \
        _DEBUGLIB_DEBUG (Expression);       \
      }                            \
    } while (FALSE)
#else
  #define DEBUG(Expression)        \
    do {                           \
      if (FALSE) {                 \
        _DEBUGLIB_DEBUG (Expression);       \
      }                            \
    } while (FALSE)
#endif
```

### How It Works

1. **Compile-Time Filtering**:
   - If `MDEPKG_NDEBUG` is **defined**: DEBUG becomes dead code, compiler removes it
   - If `MDEPKG_NDEBUG` is **not defined**: DEBUG code is included

2. **Runtime Filtering**:
   - `DebugPrintEnabled()` checks `PcdDebugPropertyMask & DEBUG_PROPERTY_DEBUG_PRINT_ENABLED`
   - Returns TRUE if debug printing is enabled at runtime

3. **Expression Processing**:
   - `_DEBUGLIB_DEBUG(Expression)` expands to `_DEBUG_PRINT(Expression)`
   - Which further expands to check error level and call `DebugPrint()`

### Helper Macros

```c
#define _DEBUG_PRINT(PrintLevel, ...)              \
    do {                                             \
      if (DebugPrintLevelEnabled (PrintLevel)) {     \
        DebugPrint (PrintLevel, ##__VA_ARGS__);      \
      }                                              \
    } while (FALSE)

#define _DEBUGLIB_DEBUG(Expression)  _DEBUG_PRINT Expression
```

### Error Levels

**Location**: `MdePkg/Include/Library/DebugLib.h:39-61`

```c
#define DEBUG_INIT      0x00000001  // Initialization
#define DEBUG_WARN      0x00000002  // Warnings
#define DEBUG_LOAD      0x00000004  // Load events
#define DEBUG_FS        0x00000008  // File system
#define DEBUG_POOL      0x00000010  // Pool allocations
#define DEBUG_PAGE      0x00000020  // Page allocations
#define DEBUG_INFO      0x00000040  // Informational messages
#define DEBUG_DISPATCH  0x00000080  // PEI/DXE/SMM Dispatchers
#define DEBUG_VARIABLE  0x00000100  // Variable
#define DEBUG_BM        0x00000400  // Boot Manager
#define DEBUG_BLKIO     0x00001000  // Block I/O
#define DEBUG_NET       0x00004000  // Network I/O
#define DEBUG_UNDI      0x00010000  // UNDI Driver
#define DEBUG_LOADFILE  0x00020000  // LoadFile
#define DEBUG_EVENT     0x00080000  // Events
#define DEBUG_GCD       0x00100000  // Global Coherency Database
#define DEBUG_CACHE     0x00200000  // Cache changes
#define DEBUG_VERBOSE   0x00400000  // Verbose messages
#define DEBUG_MANAGEABILITY 0x00800000  // Manageability (Redfish, IPMI, MCTP)
#define DEBUG_SECURITY  0x01000000  // Security (TPM)
#define DEBUG_ERROR     0x80000000  // Errors
```

### Example Usage

```c
DEBUG((DEBUG_INFO, "Initializing driver\n"));
DEBUG((DEBUG_ERROR, "Failed to allocate memory, Status = %r\n", Status));
DEBUG((DEBUG_VERBOSE, "Register value = 0x%08x\n", RegValue));
```

---

## Layer 2: DebugPrint Implementation (BaseDebugLibSerialPort)

**Location**: `MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c`

### Function Call Chain

```
DebugPrint() [Line 67]
    │
    ├─── Creates VA_LIST from varargs
    └─── Calls DebugVPrint()
         │
         └─── DebugVPrint() [Line 151]
              │
              └─── Calls DebugPrintMarker()
                   │
                   └─── DebugPrintMarker() [Line 98]
                        ├─── Checks error level
                        ├─── Formats string with AsciiVSPrint()
                        └─── Calls SerialPortWrite()
```

### Key Code: DebugPrintMarker()

**Location**: `MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c:98-132`

```c
VOID
DebugPrintMarker (
  IN  UINTN        ErrorLevel,
  IN  CONST CHAR8  *Format,
  IN  VA_LIST      VaListMarker,
  IN  BASE_LIST    BaseListMarker
  )
{
  CHAR8  Buffer[MAX_DEBUG_MESSAGE_LENGTH];  // 256 bytes

  // Validate format string
  ASSERT (Format != NULL);

  // Check driver debug mask value and global mask
  if ((ErrorLevel & GetDebugPrintErrorLevel ()) == 0) {
    return;
  }

  // Convert the DEBUG() message to an ASCII String
  if (BaseListMarker == NULL) {
    AsciiVSPrint (Buffer, sizeof (Buffer), Format, VaListMarker);
  } else {
    AsciiBSPrint (Buffer, sizeof (Buffer), Format, BaseListMarker);
  }

  // Send the print string to a Serial Port
  SerialPortWrite ((UINT8 *)Buffer, AsciiStrLen (Buffer));
}
```

### Configuration Functions

**Location**: `MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c:269-364`

```c
// Returns TRUE if DEBUG() macros are enabled
BOOLEAN DebugPrintEnabled (VOID)
{
  return (BOOLEAN)((PcdGet8(PcdDebugPropertyMask) &
                    DEBUG_PROPERTY_DEBUG_PRINT_ENABLED) != 0);
}

// Returns TRUE if specific error level is enabled
BOOLEAN DebugPrintLevelEnabled (IN CONST UINTN ErrorLevel)
{
  return (BOOLEAN)((ErrorLevel &
                    PcdGet32(PcdFixedDebugPrintErrorLevel)) != 0);
}
```

### Constructor

**Location**: `MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c:43-48`

```c
RETURN_STATUS
EFIAPI
BaseDebugLibSerialPortConstructor (VOID)
{
  return SerialPortInitialize();  // Initialize UART hardware
}
```

---

## Layer 3: Serial Port Library (SerialPortLib)

### Interface Definition

**Location**: `MdePkg/Include/Library/SerialPortLib.h`

```c
RETURN_STATUS EFIAPI SerialPortInitialize (VOID);
UINTN EFIAPI SerialPortWrite (IN UINT8 *Buffer, IN UINTN NumberOfBytes);
UINTN EFIAPI SerialPortRead (OUT UINT8 *Buffer, IN UINTN NumberOfBytes);
BOOLEAN EFIAPI SerialPortPoll (VOID);
RETURN_STATUS EFIAPI SerialPortSetControl (IN UINT32 Control);
RETURN_STATUS EFIAPI SerialPortGetControl (OUT UINT32 *Control);
RETURN_STATUS EFIAPI SerialPortSetAttributes (...);
```

### 16550 UART Implementation

**Location**: `MdeModulePkg/Library/BaseSerialPortLib16550/BaseSerialPortLib16550.c`

#### UART Register Definitions

**Lines 30-51**

```c
// 16550 UART register offsets
#define R_UART_RXBUF         0    // Receive Buffer (Read, DLAB=0)
#define R_UART_TXBUF         0    // Transmit Buffer (Write, DLAB=0)
#define R_UART_BAUD_LOW      0    // Baud Rate Divisor LSB (DLAB=1)
#define R_UART_BAUD_HIGH     1    // Baud Rate Divisor MSB (DLAB=1)
#define R_UART_IER           1    // Interrupt Enable Register
#define R_UART_FCR           2    // FIFO Control Register
#define   B_UART_FCR_FIFOE   BIT0 // FIFO Enable
#define   B_UART_FCR_FIFO64  BIT5 // 64-byte FIFO Enable
#define R_UART_LCR           3    // Line Control Register
#define   B_UART_LCR_DLAB    BIT7 // Divisor Latch Access Bit
#define R_UART_MCR           4    // Modem Control Register
#define   B_UART_MCR_DTRC    BIT0 // Data Terminal Ready Control
#define   B_UART_MCR_RTS     BIT1 // Request To Send
#define R_UART_LSR           5    // Line Status Register
#define   B_UART_LSR_RXRDY   BIT0 // Receiver Data Ready
#define   B_UART_LSR_TXRDY   BIT5 // Transmitter Holding Register Empty
#define   B_UART_LSR_TEMT    BIT6 // Transmitter Empty
#define R_UART_MSR           6    // Modem Status Register
#define   B_UART_MSR_CTS     BIT4 // Clear To Send
#define   B_UART_MSR_DSR     BIT5 // Data Set Ready
#define   B_UART_MSR_RI      BIT6 // Ring Indicator
#define   B_UART_MSR_DCD     BIT7 // Data Carrier Detect
```

#### SerialPortWrite() Function

**Lines 606-684**

```c
UINTN EFIAPI
SerialPortWrite (
  IN UINT8  *Buffer,
  IN UINTN  NumberOfBytes
  )
{
  UINTN  SerialRegisterBase;
  UINTN  Result;
  UINTN  Index;
  UINTN  FifoSize;

  if (Buffer == NULL) {
    return 0;
  }

  // Get the base address (0x3F8 for COM1, or from PCI)
  SerialRegisterBase = GetSerialRegisterBase();
  if (SerialRegisterBase == 0) {
    return 0;
  }

  if (NumberOfBytes == 0) {
    // Flush hardware - wait for TX complete
    while ((SerialPortReadRegister(SerialRegisterBase, R_UART_LSR) &
            (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) !=
            (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) {
    }
    while (!SerialPortWritable(SerialRegisterBase)) {
    }
    return 0;
  }

  // Compute FIFO size (1, 16, or 64 bytes)
  FifoSize = 1;
  if ((PcdGet8(PcdSerialFifoControl) & B_UART_FCR_FIFOE) != 0) {
    if ((PcdGet8(PcdSerialFifoControl) & B_UART_FCR_FIFO64) == 0) {
      FifoSize = 16;
    } else {
      FifoSize = PcdGet32(PcdSerialExtendedTxFifoSize);
    }
  }

  Result = NumberOfBytes;
  while (NumberOfBytes != 0) {
    // Wait for UART ready (TX FIFO and shift register empty)
    while ((SerialPortReadRegister(SerialRegisterBase, R_UART_LSR) &
            (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) !=
            (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) {
    }

    // Fill entire TX FIFO
    for (Index = 0; Index < FifoSize && NumberOfBytes != 0;
         Index++, NumberOfBytes--, Buffer++) {
      // Wait for hardware flow control signal
      while (!SerialPortWritable(SerialRegisterBase)) {
      }

      // Write byte to transmit buffer
      SerialPortWriteRegister(SerialRegisterBase, R_UART_TXBUF, *Buffer);
    }
  }

  return Result;
}
```

#### SerialPortInitialize() Function

**Lines 489-583**

```c
RETURN_STATUS EFIAPI
SerialPortInitialize (VOID)
{
  RETURN_STATUS  Status;
  UINTN          SerialRegisterBase;
  UINT32         Divisor;
  UINT32         CurrentDivisor;
  BOOLEAN        Initialized;

  // Platform-specific initialization hook
  Status = PlatformHookSerialPortInitialize();
  if (RETURN_ERROR(Status)) {
    return Status;
  }

  // Calculate baud rate divisor: Ref_Clk_Rate / Baud_Rate / 16
  Divisor = PcdGet32(PcdSerialClockRate) /
            (PcdGet32(PcdSerialBaudRate) * 16);
  if ((PcdGet32(PcdSerialClockRate) % (PcdGet32(PcdSerialBaudRate) * 16)) >=
       PcdGet32(PcdSerialBaudRate) * 8) {
    Divisor++;
  }

  // Get serial port base address
  SerialRegisterBase = GetSerialRegisterBase();
  if (SerialRegisterBase == 0) {
    return RETURN_DEVICE_ERROR;
  }

  // Check if already initialized
  Initialized = TRUE;
  if ((SerialPortReadRegister(SerialRegisterBase, R_UART_LCR) & 0x3F) !=
      (PcdGet8(PcdSerialLineControl) & 0x3F)) {
    Initialized = FALSE;
  }

  // Check current divisor
  SerialPortWriteRegister(SerialRegisterBase, R_UART_LCR,
    (UINT8)(SerialPortReadRegister(SerialRegisterBase, R_UART_LCR) |
            B_UART_LCR_DLAB));
  CurrentDivisor = SerialPortReadRegister(SerialRegisterBase,
                                          R_UART_BAUD_HIGH) << 8;
  CurrentDivisor |= (UINT32)SerialPortReadRegister(SerialRegisterBase,
                                                    R_UART_BAUD_LOW);
  SerialPortWriteRegister(SerialRegisterBase, R_UART_LCR,
    (UINT8)(SerialPortReadRegister(SerialRegisterBase, R_UART_LCR) &
            ~B_UART_LCR_DLAB));
  if (CurrentDivisor != Divisor) {
    Initialized = FALSE;
  }

  if (Initialized) {
    return RETURN_SUCCESS;
  }

  // Wait for UART to be ready
  while ((SerialPortReadRegister(SerialRegisterBase, R_UART_LSR) &
          (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) !=
          (B_UART_LSR_TEMT | B_UART_LSR_TXRDY)) {
  }

  // Configure baud rate (set DLAB=1, write divisor, clear DLAB)
  SerialPortWriteRegister(SerialRegisterBase, R_UART_LCR, B_UART_LCR_DLAB);
  SerialPortWriteRegister(SerialRegisterBase, R_UART_BAUD_HIGH,
                          (UINT8)(Divisor >> 8));
  SerialPortWriteRegister(SerialRegisterBase, R_UART_BAUD_LOW,
                          (UINT8)(Divisor & 0xff));

  // Configure data bits, parity, stop bits
  SerialPortWriteRegister(SerialRegisterBase, R_UART_LCR,
                          (UINT8)(PcdGet8(PcdSerialLineControl) & 0x3F));

  // Enable and reset FIFOs
  SerialPortWriteRegister(SerialRegisterBase, R_UART_FCR, 0x00);
  SerialPortWriteRegister(SerialRegisterBase, R_UART_FCR,
    (UINT8)(PcdGet8(PcdSerialFifoControl) &
            (B_UART_FCR_FIFOE | B_UART_FCR_FIFO64)));

  // Set FIFO polled mode (disable interrupts)
  SerialPortWriteRegister(SerialRegisterBase, R_UART_IER, 0x00);

  // Reset Modem Control Register
  SerialPortWriteRegister(SerialRegisterBase, R_UART_MCR, 0x00);

  return RETURN_SUCCESS;
}
```

---

## Layer 4: Hardware I/O (IoLib)

### Register Access Functions

**Location**: `MdeModulePkg/Library/BaseSerialPortLib16550/BaseSerialPortLib16550.c:76-122`

```c
UINT8
SerialPortReadRegister (
  UINTN  Base,
  UINTN  Offset
  )
{
  if (PcdGetBool(PcdSerialUseMmio)) {
    // Memory-Mapped I/O (ARM, x64)
    if (PcdGet8(PcdSerialRegisterAccessWidth) == 32) {
      return (UINT8)MmioRead32(Base + Offset *
                               PcdGet32(PcdSerialRegisterStride));
    }
    return MmioRead8(Base + Offset * PcdGet32(PcdSerialRegisterStride));
  } else {
    // I/O Port (x86)
    return IoRead8(Base + Offset * PcdGet32(PcdSerialRegisterStride));
  }
}

UINT8
SerialPortWriteRegister (
  UINTN  Base,
  UINTN  Offset,
  UINT8  Value
  )
{
  if (PcdGetBool(PcdSerialUseMmio)) {
    // Memory-Mapped I/O (ARM, x64)
    if (PcdGet8(PcdSerialRegisterAccessWidth) == 32) {
      return (UINT8)MmioWrite32(Base + Offset *
                                PcdGet32(PcdSerialRegisterStride),
                                (UINT8)Value);
    }
    return MmioWrite8(Base + Offset * PcdGet32(PcdSerialRegisterStride),
                      Value);
  } else {
    // I/O Port (x86)
    return IoWrite8(Base + Offset * PcdGet32(PcdSerialRegisterStride),
                    Value);
  }
}
```

### IoLib Functions

**Location**: `MdePkg/Include/Library/IoLib.h`

#### I/O Port Access (x86/x64)

```c
UINT8 EFIAPI IoRead8 (IN UINTN Port);
UINT8 EFIAPI IoWrite8 (IN UINTN Port, IN UINT8 Value);
```

**Implementation** (architecture-specific):
- **x86/x64**: Uses `IN` and `OUT` assembly instructions
- Example: `OUT 0x3F8, AL` writes byte in AL register to I/O port 0x3F8

#### Memory-Mapped I/O (ARM, x64)

```c
UINT8 EFIAPI MmioRead8 (IN UINTN Address);
UINT8 EFIAPI MmioWrite8 (IN UINTN Address, IN UINT8 Value);
UINT32 EFIAPI MmioRead32 (IN UINTN Address);
UINT32 EFIAPI MmioWrite32 (IN UINTN Address, IN UINT32 Value);
```

**Implementation**:
- Direct memory access: `*(volatile UINT8 *)Address = Value;`
- Compiler ensures proper memory barriers for volatile access

---

## Configuration via PCDs

PCDs (Platform Configuration Database) control debug behavior at build time.

### Debug Control PCDs

| PCD Name | Type | Description | Default |
|----------|------|-------------|---------|
| `PcdDebugPropertyMask` | UINT8 | Controls which debug features are enabled | 0xFF |
| `PcdFixedDebugPrintErrorLevel` | UINT32 | Bitmask of enabled error levels | 0xFFFFFFFF |
| `PcdDebugClearMemoryValue` | UINT8 | Value for clearing memory | 0xAF |

**PcdDebugPropertyMask Bits**:
```c
#define DEBUG_PROPERTY_DEBUG_ASSERT_ENABLED       0x01
#define DEBUG_PROPERTY_DEBUG_PRINT_ENABLED        0x02
#define DEBUG_PROPERTY_DEBUG_CODE_ENABLED         0x04
#define DEBUG_PROPERTY_CLEAR_MEMORY_ENABLED       0x08
#define DEBUG_PROPERTY_ASSERT_BREAKPOINT_ENABLED  0x10
#define DEBUG_PROPERTY_ASSERT_DEADLOOP_ENABLED    0x20
```

### Serial Port PCDs

| PCD Name | Type | Description | Example Value |
|----------|------|-------------|---------------|
| `PcdSerialUseMmio` | BOOLEAN | Use MMIO (TRUE) or I/O ports (FALSE) | FALSE (x86) / TRUE (ARM) |
| `PcdSerialRegisterBase` | UINT64 | Base address of UART | 0x3F8 (COM1) |
| `PcdSerialBaudRate` | UINT32 | Baud rate | 115200 |
| `PcdSerialClockRate` | UINT32 | UART reference clock | 1843200 Hz |
| `PcdSerialLineControl` | UINT8 | Data bits, parity, stop bits | 0x03 (8N1) |
| `PcdSerialFifoControl` | UINT8 | FIFO enable and depth | 0x07 (16-byte) |
| `PcdSerialRegisterStride` | UINT32 | Register spacing | 1 or 4 |
| `PcdSerialRegisterAccessWidth` | UINT8 | MMIO access width (8 or 32) | 8 |
| `PcdSerialUseHardwareFlowControl` | BOOLEAN | Enable RTS/CTS | FALSE |
| `PcdSerialDetectCable` | BOOLEAN | Detect cable via DSR | FALSE |
| `PcdSerialPciDeviceInfo` | VOID* | PCI UART device info | {0xFF, 0, 0} |
| `PcdSerialExtendedTxFifoSize` | UINT32 | Extended FIFO size | 64 |

### Common Serial Port Addresses

| Platform | Port | Base Address (I/O) | Base Address (MMIO) |
|----------|------|-------------------|---------------------|
| x86 PC | COM1 | 0x3F8 | N/A |
| x86 PC | COM2 | 0x2F8 | N/A |
| x86 PC | COM3 | 0x3E8 | N/A |
| x86 PC | COM4 | 0x2E8 | N/A |
| ARM QEMU | UART0 | N/A | 0x09000000 |
| ARM PL011 | UART | N/A | Platform-specific |

---

## Example Trace

Let's trace a complete DEBUG call through the system.

### Source Code

```c
// File: MyDriver/MyDriver.c
#include <Library/DebugLib.h>

EFI_STATUS
EFIAPI
MyDriverEntryPoint (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  UINT32 Value = 0x12345678;

  DEBUG((DEBUG_INFO, "MyDriver: Value = 0x%08x\n", Value));

  return EFI_SUCCESS;
}
```

### Step-by-Step Execution

#### Step 1: Macro Expansion

**Original**:
```c
DEBUG((DEBUG_INFO, "MyDriver: Value = 0x%08x\n", Value));
```

**After DEBUG macro expansion** (MDEPKG_NDEBUG not defined):
```c
do {
  if (DebugPrintEnabled()) {
    _DEBUGLIB_DEBUG((DEBUG_INFO, "MyDriver: Value = 0x%08x\n", Value));
  }
} while (FALSE)
```

**After _DEBUGLIB_DEBUG expansion**:
```c
do {
  if (DebugPrintEnabled()) {
    _DEBUG_PRINT(DEBUG_INFO, "MyDriver: Value = 0x%08x\n", Value);
  }
} while (FALSE)
```

**After _DEBUG_PRINT expansion**:
```c
do {
  if (DebugPrintEnabled()) {
    do {
      if (DebugPrintLevelEnabled(DEBUG_INFO)) {
        DebugPrint(DEBUG_INFO, "MyDriver: Value = 0x%08x\n", Value);
      }
    } while (FALSE)
  }
} while (FALSE)
```

#### Step 2: Runtime Checks

1. **DebugPrintEnabled()** returns TRUE
   - Checks: `PcdDebugPropertyMask & 0x02` = TRUE

2. **DebugPrintLevelEnabled(DEBUG_INFO)** returns TRUE
   - Checks: `DEBUG_INFO (0x40) & PcdFixedDebugPrintErrorLevel` = TRUE

#### Step 3: DebugPrint() Call

**Location**: `BaseDebugLibSerialPort/DebugLib.c:67`

```c
DebugPrint(0x00000040, "MyDriver: Value = 0x%08x\n", 0x12345678)
  │
  └─→ VA_START(Marker, Format)
  └─→ DebugVPrint(0x00000040, "MyDriver: Value = 0x%08x\n", Marker)
      │
      └─→ DebugPrintMarker(0x00000040, "MyDriver: Value = 0x%08x\n",
                           Marker, NULL)
```

#### Step 4: DebugPrintMarker()

**Location**: `BaseDebugLibSerialPort/DebugLib.c:98`

```c
CHAR8 Buffer[256];

// Error level check
if ((0x40 & GetDebugPrintErrorLevel()) == 0) {
  return;  // Would skip if level disabled
}

// Format the string
AsciiVSPrint(Buffer, 256, "MyDriver: Value = 0x%08x\n", Marker);
// Buffer now contains: "MyDriver: Value = 0x12345678\n"

// Send to serial port
SerialPortWrite((UINT8 *)Buffer, 29);  // 29 = string length
```

#### Step 5: SerialPortWrite()

**Location**: `BaseSerialPortLib16550/BaseSerialPortLib16550.c:606`

**Assumptions**:
- Platform: x86 PC
- Port: COM1 (0x3F8)
- FIFO: 16550A (16-byte FIFO enabled)

```c
SerialRegisterBase = GetSerialRegisterBase();
// Returns: 0x3F8

FifoSize = 16;  // 16-byte FIFO

// Write "MyDriver: Value = 0x12345678\n" (29 bytes)

Iteration 1:
  // Wait for LSR.TEMT & LSR.TXRDY
  while ((IoRead8(0x3F8 + 5) & 0x60) != 0x60) { }

  // Fill FIFO with 16 bytes: "MyDriver: Value "
  for (i = 0; i < 16; i++) {
    IoWrite8(0x3F8, Buffer[i]);
  }

Iteration 2:
  // Wait for LSR.TEMT & LSR.TXRDY
  while ((IoRead8(0x3F8 + 5) & 0x60) != 0x60) { }

  // Fill FIFO with 13 bytes: "= 0x12345678\n"
  for (i = 16; i < 29; i++) {
    IoWrite8(0x3F8, Buffer[i]);
  }
```

#### Step 6: IoWrite8()

**For each character**:
```
Assembly (x86):
  MOV DX, 0x3F8      ; Port address
  MOV AL, [character] ; Character to write
  OUT DX, AL         ; Write to I/O port
```

**Hardware**:
1. CPU executes `OUT` instruction
2. Data appears on I/O bus at address 0x3F8
3. UART chip (16550) detects write to Transmit Holding Register
4. Character transferred to Transmit Shift Register
5. UART serializes data: Start bit + 8 data bits + 1 stop bit
6. Serial data output on TX pin at 115200 baud
7. RS-232 level converter outputs signal
8. Terminal receives characters on COM1

#### Step 7: Terminal Output

Terminal (PuTTY, minicom, etc.) displays:
```
MyDriver: Value = 0x12345678
```

---

## Alternative Implementations

EDK2 provides multiple DebugLib implementations for different environments:

### 1. BaseDebugLibNull

**Location**: `MdePkg/Library/BaseDebugLibNull/`

**Purpose**: Null implementation (no output)

**Use Case**: Production builds, size optimization

**Implementation**: All functions return immediately without doing anything

```c
VOID DebugPrint (IN UINTN ErrorLevel, IN CONST CHAR8 *Format, ...)
{
  // Do nothing
}
```

### 2. UefiDebugLibConOut

**Location**: `MdePkg/Library/UefiDebugLibConOut/`

**Purpose**: Output to UEFI console (screen)

**Use Case**: DXE/UEFI phase when console is available

**Implementation**: Uses `gST->ConOut->OutputString()`

### 3. UefiDebugLibStdErr

**Location**: `MdePkg/Library/UefiDebugLibStdErr/`

**Purpose**: Output to UEFI Standard Error device

**Use Case**: DXE/UEFI phase

**Implementation**: Uses `gST->StdErr->OutputString()`

### 4. PeiDxeDebugLibReportStatusCode

**Location**: `MdeModulePkg/Library/PeiDxeDebugLibReportStatusCode/`

**Purpose**: Output via Report Status Code Protocol

**Use Case**: When debug output needs to be captured by platform code

**Implementation**: Converts DEBUG to status code reports

### 5. SemiHostingDebugLib

**Location**: `ArmPkg/Library/SemiHostingDebugLib/`

**Purpose**: Output via ARM semihosting

**Use Case**: ARM development with debugger attached

**Implementation**: Uses semihosting SYS_WRITEC calls

### 6. DebugLibFdtPL011Uart

**Location**: `ArmVirtPkg/Library/DebugLibFdtPL011Uart/`

**Purpose**: ARM PL011 UART with Device Tree discovery

**Use Case**: ARM virtual machines (QEMU)

**Implementation**: Parses FDT to find UART base address

### 7. PlatformDebugLibIoPort

**Location**: `OvmfPkg/Library/PlatformDebugLibIoPort/`

**Purpose**: QEMU/OVMF debug port (I/O port 0x402)

**Use Case**: QEMU x86 virtual machines

**Implementation**: Writes to special QEMU debug I/O port

---

## Summary

The EDK2 DEBUG() system is a multi-layered architecture:

1. **DEBUG Macro** → Compile-time and runtime filtering
2. **DebugPrint()** → String formatting and buffering
3. **SerialPortWrite()** → Hardware abstraction
4. **IoWrite8()/MmioWrite8()** → Low-level hardware access
5. **UART Hardware** → Physical serial output

This design provides:
- **Flexibility**: Multiple output destinations (serial, console, status codes)
- **Performance**: Compile-time removal of debug code when disabled
- **Portability**: Same code works across x86, ARM, RISC-V
- **Configurability**: PCDs control all aspects of debug behavior

The key insight is that **the interface is fixed, but implementations are swappable at build time**, allowing the same driver code to work across all platforms.

---

## File References

### Key Headers
- `MdePkg/Include/Library/DebugLib.h` - DEBUG macro definitions
- `MdePkg/Include/Library/SerialPortLib.h` - Serial port interface
- `MdePkg/Include/Library/IoLib.h` - I/O access functions

### Key Implementations
- `MdePkg/Library/BaseDebugLibSerialPort/DebugLib.c` - Serial debug implementation
- `MdeModulePkg/Library/BaseSerialPortLib16550/BaseSerialPortLib16550.c` - 16550 UART driver

### Configuration
- Platform `.dsc` file - PCD values
- Module `.inf` file - Library dependencies

---

**Document Version**: 1.0
**EDK2 Codebase**: Master branch (as of commit 7410754041)
**Author**: Generated from EDK2 source code analysis
**Date**: 2025-12-01
