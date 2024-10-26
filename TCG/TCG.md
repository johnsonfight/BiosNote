

Trusted Computing Ecosystem
├── Trusted Computing Group (TCG)
│   ├── TPM (Trusted Platform Module)
│   │   ├── Hardware TPM (Discrete TPM Chip)
│   │   └── Software-based TPMs
│   │       ├── Intel PTT (Platform Trust Technology)
│   │       └── fTPM (Firmware TPM)
│   │   ├── PCR (Platform Configuration Registers)
│   │   │   ├── Used for integrity measurement
│   │   │   ├── Store hash values of software and configurations
│   │   │   └── Integral to secure boot and attestation processes
│   └── TCG Standards & Specifications
│       ├── Platform Attestation
│       ├── Secure Boot
│       └── Root of Trust
├── Intel TXT (Trusted Execution Technology)
│   ├── Utilizes TPM for secure boot and attestation
│   └── Works with Intel PTT (as TPM replacement)
├── AMD SVM (Secure Virtual Machine)
│   └── Works with fTPM (AMD's firmware-based TPM)
├── ARM TrustZone (Hardware-based security)
│   ├── Provides a secure environment for trusted code execution
│   └── Can work with fTPM in ARM-based platforms
└── Microsoft Windows
    ├── Uses TPM (hardware or firmware) for BitLocker, Secure Boot, etc.
    └── Works with fTPM (on devices without discrete TPM chips)
