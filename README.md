A bootloader is a piece of software located on a special 4KB sector of the drive known as the Master Boot Record (MBR). The MBR is the first sector of the drive by convention, and the operating system will write the bootloader to the sector as part of it's installation. When your computer powers on, it loads its first instruction of the bootloader from an address that is in fact right at the top of the 32-bit address space, FFFFFFF0, and begins executing in 32bit unreal mode. This is the demarcation point in the boot process between the BIOS and bootloader, and triggers the portion of the boot process that is controlled by the bootloader and operating system.

EFI firmwares in general switch to protected mode within a few instructions of exiting processor reset. Switching to protected mode is done early on in the so-called "SEC Phase" of EFI firmware initialization. Technically, 32-bit and greater x86 processors don't even start in real mode proper, but in what is colloquially known as unreal mode. (The initial segment descriptor for the CS register does not describe the conventional real mode mapping and is what makes this "unreal".) The EFI doesn't load a bootstrap program from sector #0 of a disc at all. They bootstrap by means of the EFI Boot Manager loading and running an EFI boot loader application. Such programs are run in protected mode.

As such, it could be said that those EFI systems never enter real mode proper at all. When bootstrapping natively to an EFI bootloader and not employing a compatibility support module, they switch from unreal mode directly to protected mode and stay in protected mode from then on.

The bootloader can be written in most languages that compile into X86 assembly, usually Assembler, C, and C++. Recently Rust and C# with .NET Native have entered the fray as new options. This project is written in C++ with the MSVC toolchain, with some assembly to create the entry point and interupt calls. In the spirit of running bare metal, this project avoids using stdlib or any other libraries.

We'll be using Developer Command Prompt for VS 2017 in administrator mode as the shell environment. we'll need to add NASM (Netwide Assembler) and Qemu to the path
%PATH%="%PATH%;C:\Users\username\AppData\Local\bin\NASM;C:\Program Files\qemu"

Creating the disk image for UEFI
The UEFI specification allows you to create applications that the firmware can run during the boot process. This is what we will use to load the operating system. These applications must be stored on a FAT12, FAT16 or FAT32 file system and located on a hard disk with a GPT partition. So our first step is to create a virtual hard disk that is ready to accept a UEFI application.

The following will create the disk image on the V:\ drive
diskpart
create vdisk file=C:\projectrepo\images\kernel.vhd maximum=512
select vdisk file=C:\projectrepo\images\kernel.vhd
attach vdisk
convert gpt
create partition efi size=200
create partition primary
format fs=fat32 quick
assign letter=V
exit

Let's build a simple efi script and run it to test everything done so far. (Thanks Brian Otto)

###kernel.asm
; Copyright 2018-2019 Brian Otto @ https://hackerpulp.com
; 
; Permission to use, copy, modify, and/or distribute this software for any 
; purpose with or without fee is hereby granted, provided that the above 
; copyright notice and this permission notice appear in all copies.
; 
; THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH 
; REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY 
; AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, 
; INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM 
; LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE 
; OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR 
; PERFORMANCE OF THIS SOFTWARE.

; generate 64-bit code
bits 64

; contains the code that will run
section .text

; allows the linker to see this symbol
global _start

; see http://www.uefi.org/sites/default/files/resources/UEFI Spec 2_7_A Sept 6.pdf#G8.1001729
struc EFI_TABLE_HEADER
    .Signature    RESQ 1
    .Revision     RESD 1
    .HeaderSize   RESD 1
    .CRC32        RESD 1
    .Reserved     RESD 1
endstruc

; see http://www.uefi.org/sites/default/files/resources/UEFI Spec 2_7_A Sept 6.pdf#G8.1001773
struc EFI_SYSTEM_TABLE
    .Hdr                  RESB EFI_TABLE_HEADER_size
    .FirmwareVendor       RESQ 1
    .FirmwareRevision     RESD 1
    .ConsoleInHandle      RESQ 1
    .ConIn                RESQ 1
    .ConsoleOutHandle     RESQ 1
    .ConOut               RESQ 1
    .StandardErrorHandle  RESQ 1
    .StdErr               RESQ 1
    .RuntimeServices      RESQ 1
    .BootServices         RESQ 1
    .NumberOfTableEntries RESQ 1
    .ConfigurationTable   RESQ 1
endstruc

; see http://www.uefi.org/sites/default/files/resources/UEFI Spec 2_7_A Sept 6.pdf#G16.1016807
struc EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL
    .Reset             RESQ 1
    .OutputString      RESQ 1
    .TestString	       RESQ 1
    .QueryMode	       RESQ 1
    .SetMode	       RESQ 1
    .SetAttribute      RESQ 1
    .ClearScreen       RESQ 1
    .SetCursorPosition RESQ 1
    .EnableCursor      RESQ 1
    .Mode              RESQ 1
endstruc

loopForever:
    jmp loopForever

_start:
    ; reserve space for 4 arguments
    sub rsp, 4 * 8

    ; rdx points to the EFI_SYSTEM_TABLE structure
    ; which is the 2nd argument passed to us by the UEFI firmware
    ; adding 64 causes rcx to point to EFI_SYSTEM_TABLE.ConOut
    mov rcx, [rdx + 64]

    ; load the address of our string into rdx
    lea rdx, [rel strHello]

    ; EFI_SYSTEM_TABLE.ConOut points to EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL
    ; call OutputString on the value in rdx
    call [rcx + EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL.OutputString]
    
    ; loop forever so we can see the string before the UEFI application exits
    jmp loopForever

codesize equ $ - $$

; contains nothing - but it is required by UEFI
section .reloc

; contains the data that will be displayed
section .data
    ; this must be a Unicode string
    strHello db __utf16__ `Hello World !\0`

datasize equ $ - $$

Assemble this with NASM to get kernel.obj. Add nasm.exe to the path first
nasm -f win64 kernel.asm

Link the object file as a UEFI application and set its entry point.
This will create a C:\kernel.efi file, which is a PE32+ DLL with a EFI subsystem.
link /subsystem:EFI_APPLICATION /entry:_start kernel.obj

Copy the UEFI app to the disk image. The firmware will search for this special path
mkdir V:\EFI
mkdir V:\EFI\BOOT
copy kernel.efi V:\EFI\BOOT\BOOTX64.EFI


Detach the virtual hard disk so that other applications can use it.
diskpart
select vdisk file=C:\kernel.vhd
detach vdisk
exit

Running the UEFI Application with Qemu
Install Qemu for Windows (64 bit), and add it's binaries to the path
    Download a EFI development kit firmware image for OVMF x64: https://www.kraxel.org/repos/jenkins/edk2/ https://www.kraxel.org/repos/jenkins/edk2/edk2.git-ovmf-x64-0-20190308.1011.ge0fd9ece26.noarch.rpm
        TianoCore provides an edk2.git-ovmf-x64-XXX.noarch.rpm image
        Open the RPM using 7zip and extract the following file …
        edk2.git-ovmf-x64-XXX.noarch.cpio\.\usr\share\edk2.git\ovmf-x64\OVMF-pure-efi.fd
        to C:\projectrepo\bios\OVMF-pure-efi.fd
    Run Qemu using the UEFI firmware and your virtual hard disk

qemu-system-x86_64 -cpu qemu64 -bios bios\OVMF-pure-efi.fd -drive file=images\kernel.vhd,format=raw
[readme\efi-hello-world.png]