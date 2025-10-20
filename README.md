# 🛡️ hARMless

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Platform](https://img.shields.io/badge/platform-ARM64%20Linux-green.svg)
![Build](https://img.shields.io/badge/build-passing-brightgreen.svg)
![Stars](https://img.shields.io/github/stars/litemars/hARMless?style=social)



**An ARM64 ELF Packer/Loader for AArch64 Linux Binaries**

A comprehensive security research tool that encrypts ARM64 ELF executables using multi-layer encryption and provides runtime in-memory execution without writing the original binary to disk.

---

## 📋 Table of Contents

- [Features](#-features)
- [Quick Start](#-quick-start)
- [Installation](#-installation)
- [Usage](#-usage)
- [Technical Details](#-technical-details)
- [Security Features](#-security-features)
- [Architecture](#-architecture)
- [Contributing](#-contributing)
- [License](#-license)

---

## ✨ Features

- **🎯 ARM64 ELF Support**: Specifically designed for AArch64 Linux binaries
- **🔐 Multi-Layer Encryption**: Triple encryption using AES-256, ChaCha20, and RC4
- **💾 Memory Execution**: Runtime decryption and execution entirely in memory using `memfd_create`
- **🔒 Code Obfuscation**: Advanced obfuscation techniques for anti-analysis
- **✅ CRC32 Verification**: Integrity checking to detect tampering
- **📦 Self-Contained**: Packed binaries are completely standalone
- **🛡️ Core Dump Prevention**: Prevents memory dumps using `setrlimit`
- **🧹 Secure Memory Wiping**: Multi-pass memory erasure for sensitive data
- **🔧 Direct Syscalls**: Bypasses userland hooks for enhanced stealth

---

## 🚀 Quick Start

```bash
# Clone the repository
git clone https://github.com/litemars/hARMless.git
cd hARMless

# Build everything
make all

# Pack a binary
make pack INPUT=/bin/ls OUTPUT=packed_ls

# Run the packed binary
./packed_ls
```

---

## 📦 Installation

### Prerequisites

- **ARM64/AArch64 Linux system** or cross-compilation toolchain
- **GCC** for ARM64 (`aarch64-linux-gnu-gcc` or native)
- **Make**
- **Standard development tools** (`git`, `build-essential`)

### Build Steps

```bash
# 1. Clone the repository
git clone https://github.com/litemars/hARMless.git
cd hARMless

# 2. Build all components
make all

# This creates:
# - build/packer    : Binary packer
# - build/loader    : Stub loader
# - build/stubgen   : Stub generator
```

### Cross-Compilation (x86_64 → ARM64)

```bash
# Install ARM64 cross-compiler
sudo apt-get install gcc-aarch64-linux-gnu

# Build with cross-compiler
make CC=aarch64-linux-gnu-gcc all
```

---

## 📖 Usage

### Basic Packing

```bash
# Pack an ARM64 binary
make pack INPUT=your_arm64_binary OUTPUT=packed_binary

# Alternative: Use tools directly
./build/packer your_arm64_binary packed_data
./build/stubgen ./build/loader packed_data packed_binary
```

### Running Packed Binaries

```bash
# Simply execute the packed binary
./packed_binary

# The packed binary will:
# 1. Read its own embedded encrypted data
# 2. Decrypt the original ELF in memory
# 3. Verify integrity with CRC32
# 4. Execute directly from memory using memfd_create
```

### Test

```bash
# Testing using /bin/ls

make test
# Output: packed_binary: packed_ls

```

---

## 🔬 Technical Details

### Encryption Pipeline

The packer uses a **triple-layer encryption** approach:

1. **RC4 Stream Cipher**: Initial obfuscation layer
2. **AES-256-CTR**: Industry-standard symmetric encryption
3. **ChaCha20**: Modern stream cipher for additional security

```
Original Binary → RC4 → AES-256 → ChaCha20 → Packed Data
```

**Key Generation**: Cryptographically secure random keys from `/dev/urandom` (256 bits per layer)

### ARM64 Direct Syscalls

The loader uses direct syscalls to bypass userland hooks:

| Syscall | Number | Purpose |
|---------|--------|---------|
| `memfd_create` | 279 | Create anonymous file descriptor |
| `execve` | 221 | Execute decrypted binary |
| `mmap` | 222 | Memory mapping |
| `write` | 64 | Output operations |
| `fexecve` | 281 | Execute from file descriptor |

**Syscall Convention (ARM64)**:
```c
// x8 = syscall number
// x0-x5 = arguments
// svc #0 = invoke
```

### Memory Safety

- **Secure Wiping**: 3-pass overwrite (zeros, ones, random)
- **No Disk Writes**: Original binary never touches filesystem
- **Stack Protection**: Non-executable stack
- **ASLR Compatible**: Position-independent code

---

## 🛡️ Security Features

### Core Dump Prevention

```c
setrlimit(RLIMIT_CORE, &(struct rlimit){0, 0});
```

Ensures sensitive memory is never written to disk, even during crashes.

### Integrity Verification

CRC32 checksums detect any tampering with:
- Encrypted payload
- Decryption keys
- Loader code

### Anti-Analysis

- **No debug symbols**: Stripped binaries
- **Obfuscated control flow**: Reduces reverse engineering surface
- **Direct syscalls**: Evades LD_PRELOAD and EDR hooks
- **In-memory execution**: No `/tmp` artifacts

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Original Binary                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Packer (packer.c)   │
         │  - Read ELF           │
         │  - Generate keys      │
         │  - Triple encrypt     │
         │  - Compute CRC32      │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Packed Data File     │
         │  [encrypted payload]  │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │ Stub Generator        │
         │ (stubgen.c)           │
         │  - Embed loader       │
         │  - Append data        │
         └───────────┬───────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────┐
│              Packed Binary (Output)                     │
│  ┌──────────────────────────────────────────────┐     │
│  │ Loader Stub (loader.c)                       │     │
│  │  - Read embedded data                        │     │
│  │  - Decrypt (ChaCha20 → AES → RC4)           │     │
│  │  - Verify CRC32                              │     │
│  │  - Create memfd                              │     │
│  │  - Execute via fexecve                       │     │
│  └──────────────────────────────────────────────┘     │
│  ┌──────────────────────────────────────────────┐     │
│  │ Encrypted Payload + Metadata                 │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Runtime Execution   │
         │  (in-memory only)     │
         └───────────────────────┘
```


---

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.


---


**⚠️ Legal Notice**: This tool is intended for:
- Authorized penetration testing
- Security research and education
- Red team operations
- Malware analysis

**Unauthorized use is prohibited and may be illegal.**

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

