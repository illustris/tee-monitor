TEE monitor
===
Repository for TEE monitor code, derived from OPTEE-OS

# About the TEE monitor project
## Need for stealthy on-device dynamic analysis
- Sophisticated malware is able to detect that it is being run in a virtualized environment through side channels
- If virtualization is detected, malware behaves differently, thereby preventing analysis
- Thus, a method to run programs on physical devices without a hypervisor and trace its behavior is needed

## ARM TEE
- ARM processors have a “secure mode” of execution that is functionally very similar to SMM on intel devices
- The “secure world” has its own private memory regions inaccessible from the “normal world”
- The secure world can access and modify normal world memory
- Similar to SWI instructions to trap to interrupt handler, the “SMC” (Secure Monitor Call) instruction can be used to trap to the secure world

## The TEE-Monitor project
- With this project, we aimed to create a secure world monitor that is able to 
    - Observe and check the integrity of the normal world kernel
    - Dump snapshots of the normal world kernel for offline analysis
    - Dump snapshots of programs running on the host
    - Trace syscalls made by programs

## OP-TEE
- The “Open Portable Trusted Execution Environment” is an open source TEE maintained by the linario foundation
- It provides an interface for secure world code to be triggered from the normal world
- The TEE-Monitor project is built on top of OP-TEE so as to utilize the memory mapping features it provides

## Accessing normal world memory from secure world
There are two mechanisms provided by OP-TEE for accessing normal world memory.
1. Static mapping of physical pages to the secure world VA space
    - Needs to be done at compile time
    - Can’t be modified during runtime
    - Used for mapping kernel pages to the secure world VA space, as the physical base address of the kernel is constant (0x4000000) for ARM.
2. Dynamic 
    - Physical address ranges can be mapped to secure world VA ranges during runtime
    - Suitable for mapping dynamically allocated pages belonging to programs

# Current functionality and future scope
## Kernel snapshots
### Current status:
- Given the base address of the kernel is known, it is statically mapped to a VA range using the “register_phys_mem” function that OPTEE provides
- This allows the kernel .text segment to be read from the secure world
- The secure world OS and trusted agents can then read the kernel memory
- I have added code to the secure world monitor that reads the .text segment of the kernel and dumps it out in the intel HEX format
- These hex dumps can be disassembled using objdump for further analysis

### Future work in kernel snapshots:
- Take periodic snapshots
- Compare with previous snapshots to detect any modifications to the kernel
- Parse data structures in kernel to get information on running processes, memory mapping etc.

## Trace system calls
- Achieved by inserting SMC instruction at the start of the SWI trap handler
- This traps system calls to the secure world, allowing the secure world monitor to log the system call. Execution is then returned to the normal world SWI handler

### Current status:
- Insertion point for SMC to trap esoteric system calls ( numbered 0x9f0000 - 0x9fffff ) has been identified
- This traps only some non-standard syscalls

### Future work in tracing syscalls:
- Requires kernel data structure parsing to implement
- Once pages belonging to a process are identified, they can be dynamically mapped, read and dumped to an intel hex format file
- This hexdump can be used for further offline analysis

# Get and build

## Prerequisites
Install the following packages (or their equivalent for your distribution):
```bash
sudo apt-get install android-tools-adb android-tools-fastboot autoconf \
	automake bc bison build-essential cscope curl device-tree-compiler \
	expect flex ftp-upload gdisk iasl libattr1-dev libc6:i386 libcap-dev \
	libfdt-dev libftdi-dev libglib2.0-dev libhidapi-dev libncurses5-dev \
	libpixman-1-dev libssl-dev libstdc++6:i386 libtool libz1:i386 make \
	mtools netcat python-crypto python-serial python-wand unzip uuid-dev \
	xdg-utils xterm xz-utils zlib1g-dev
```

## Install Android repo
OPTEE-OS uses Google repo for managing a multi-repository codebase. Install repo by following the instructions [here](https://source.android.com/setup/build/downloading).

## Get the source code
```bash
$ mkdir -p $HOME/devel/optee
$ cd $HOME/devel/optee
$ repo init -u https://github.com/illustris/manifest.git -m default.xml
$ repo sync
```

## Get the toolchains
```
$ cd build
$ make toolchains
```

## Build the solution
```bash
$ make
```

## Run
```bash
$ make run-only
```

