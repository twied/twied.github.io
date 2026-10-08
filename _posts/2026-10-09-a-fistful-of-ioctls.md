---
title: "A fistful of ioctls"
date: 2026-10-09
excerpt: >
  Twenty lines of C, a handful of system calls, and you have code running in a 
  hardware-isolated VM on Linux. Useful for plugin sandboxing, untrusted code, 
  bare-metal testing. Also a great way to accidentally spend five years writing 
  a worse QEMU from scratch.
tags:
  - terrarium
  - kvm
  - virtualization
---

Linux's hypervisor is called KVM. It's the foundation of an entire ecosystem of
tools to create, run, and manage virtual machines: QEMU, Firecracker, Cloud
Hypervisor, crosvm, libvirt, KubeVirt; millions of lines of code and
configuration. But using KVM directly is stupidly simple: you can execute code
in a hardware-isolated VM with about twenty lines of C and half a dozen system
calls, and zero dependencies. Isolate plugin code. Run something you don't
trust. Test bare-metal code without rebooting. Here's how.

## Adding two numbers

In my day job I work on [libvirt](https://libvirt.org/). If you have ever used 
`virsh`, `virt-manager`, or anything else that manages virtual machines on 
Linux, you have used libvirt. It talks to different hypervisors like QEMU and 
KVM, Xen, Hyper-V, LXC, VirtualBox, VMware, Bhyve etc. so you don't have to 
deal with the specifics. One day I needed to look something up in the KVM 
documentation, and somewhere between the description of `KVM_CREATE_VM` and 
`KVM_RUN`, I had a thought that would cost me a lot of evenings over the coming 
years: "This is really just a few ioctls."

My first "guest program" was as humble as it can get: add two numbers. `int
add(int a, int b) { return a + b; }`, compiled on
[Godbolt](https://godbolt.org/) ... only to be replaced immediately by a single
`add ax, cx`, as I had no idea yet how to properly set up the stack for a
proper function call. Then a `hlt` instruction to rather unceremoniously signal
the termination of my "virtual machine".

```c
#include <assert.h>
#include <fcntl.h>
#include <linux/kvm.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>

int main() {
    /* open kvm device */
    int kvm = open("/dev/kvm", O_RDWR);
    assert(kvm > 0);
    assert(ioctl(kvm, KVM_GET_API_VERSION, 0) == KVM_API_VERSION);
    assert(ioctl(kvm, KVM_CHECK_EXTENSION, KVM_CAP_CHECK_EXTENSION_VM) != 0);

    /* create vm */
    int vm = ioctl(kvm, KVM_CREATE_VM, 0);
    assert(vm > 0);
    assert(ioctl(vm, KVM_CHECK_EXTENSION, KVM_CAP_USER_MEMORY) != 0);

    /* create and setup memory */
    void* ptr = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    assert(ptr != NULL);
    memcpy(ptr, "\x01\xC8\xF4", 3); /* add ax, cx; hlt; */
    struct kvm_userspace_memory_region region = { 0, 0, 0, 4096, (__u64) ptr };
    assert(ioctl(vm, KVM_SET_USER_MEMORY_REGION, &region) == 0);

    /* create and setup cpu */
    int cpu = ioctl(vm, KVM_CREATE_VCPU, 0);
    assert(cpu > 0);
    struct kvm_sregs sregs;
    assert(ioctl(cpu, KVM_GET_SREGS, &sregs) == 0);
    sregs.cs.base = 0; /* default: 0xFFFF0000:FFF0 like a real CPU */
    sregs.cs.selector = 0;
    assert(ioctl(cpu, KVM_SET_SREGS, &sregs) == 0);
    struct kvm_regs regs = { 0 };
    regs.rax = 17;
    regs.rcx = 4;
    assert(ioctl(cpu, KVM_SET_REGS, &regs) == 0);

    /* run the code */
    assert(ioctl(cpu, KVM_RUN, 0) == 0);

    assert(ioctl(cpu, KVM_GET_REGS, &regs) == 0);
    assert(regs.rax == 21);

    return 0;
}
```

There you have it: Open `/dev/kvm`, create a VM, a CPU, attach some memory and
set up initial values for the registers, and off it goes. Granted, this example
doesn't do anything earth shattering, but I was hooked and my mind immediately
ran towards all the possible things I could do with this. But first things
first:

## Learning days

My employer has a program called "Day of Learning" where you can spend work
time exploring topics related to your field. Care to guess what topic I
explored for the next several Days of Learning?

The first thing I did was stop repeating myself. Every time I set up a VM, it
was the same sequence: open KVM, assert on the result. Create the VM, assert on
the result. Allocate memory, assert. Create the CPU, assert. Configure
registers, assert. Run the VM, and, you guessed it, more boilerplate asserts.

I wrapped the calls to the KVM API in C++ classes with RAII semantics:
destructors close file descriptors, constructors validate handles, API calls
are wrapped so that failures raise exceptions. The code turned into a small
library, which I called
[kvmmm](https://gitlab.com/twied/kvmdemo/-/tree/main/libkvmmm) ("KVM minus
minus", a nod to gtkmm and sdlmm) because apparently I still haven't brought my
delusions of grandeur under control.

Here is an example of kvmmm, demonstrating how to use `out` instructions in the
guest to do IO on the host:

```cpp
#include "kvmmm.hpp"

#include <cstdio>

constexpr unsigned char code[] = {
    0xB8, 0x21, 0x00,   /* mov $0x21, %ax */
    0xE7, 0x2A,         /* out %ax, $42 */
    0xF4                /* hlt */
};

int main() {
    kvmmm::KVM kvm {};

    kvmmm::Mem ram = kvmmm::Mem::memory(1 * 1024 * 1024);
    ram.insert(0, code, sizeof(code));

    kvmmm::VM vm = kvm.create_vm();
    vm.set_memory(0, 0, ram);

    kvmmm::CPU cpu = kvm.create_cpu(vm);
    kvmmm::initialize_real_mode(cpu, ram, 0);

    for (;;) {
        auto* run = cpu.run();

        if (run->exit_reason == KVM_EXIT_IO && run->io.port == 42) {
            putchar(cpu.state().get<char>(run->io.data_offset));
            continue;
        }

        if (run->exit_reason == KVM_EXIT_HLT) {
            break;
        }

        return 1;
    }

    return 0;
}
```

## More modes, more bits

I hinted at it earlier: By default, the virtual CPU is set up to start
executing not at address zero, but rather `0xFFFF0000:FFF0`. This is the [reset
vector](https://en.wikipedia.org/wiki/Reset_vector) of an x86 CPU. KVM emulates
the starting state of the VM faithfully to that of a PC directly after power
up, where it would normally jump into BIOS (or EFI) code and begin
initialization. And if you know your x86 boot sequence, you know what that
means: We are in 16-bit real mode. And if you have ever written a boot loader,
you (a) have my deepest condolences and (b) know that there are quite a few
steps to take to switch to 32-bit protected mode, or even 64-bit long mode.

If you have a "The Matrix"-like control over the environment though, as one
does with a Virtual Machine, setting this up turns out to not be so hard after
all. And you get the benefit of turning the setup code into functions in your
Virtual Machine Manager, instead of having to write it in your guest. And so,
for your convenience and mine, I added three functions, one of which you
already saw in the previous example:

```
void initialize_real_mode(CPU& cpu, Mem& ram, __u64 entry, __u64 descriptor_address = 0);

void initialize_protected_mode(CPU& cpu, Mem& ram, __u64 entry, __u64 descriptor_address);

void initialize_long_mode(CPU& cpu, Mem& ram, __u64 entry, __u64 descriptor_address);
```

`entry` is the (guest logical) address where execution will begin, and
`descriptor_address` (unused in the case of real mode, but included for
symmetry) defines the place where things like the [Global Descriptor
Table](https://wiki.osdev.org/Global_Descriptor_Table) or [Page
Tables](https://wiki.osdev.org/X86_Page_Tables) are set up.

## Multiboot

Having functions ready to effortlessly set up a protected mode or long mode
environment was a huge quality of life improvement that made experimenting and
toying around a lot more convenient. The next problem was loading code.
Hand-assembling machine code bytes into a `constexpr` array gets old fast. And
recompiling every time I changed the demo program wasn't particularly
comfortable either.

Back in university I wrote a toy operating system. I got as far as rudimentary
hard disk IO, text mode EGA display, keyboard input, multitasking, enumerating
PCI devices, and, most importantly: Loading ELF files. It couldn't do much with
the PCI devices it found, and the multitasking code had a bug I never figured
out that made the whole thing crash after a couple of minutes, but my knowledge
about the ELF file format sprang to mind: Why don't I try to write an ELF
loader for my VM? That would allow me to write proper freestanding C programs
and compile them with `gcc -ffreestanding -nostdlib`. No more extracting code
from object files with `objdump` and fiddling with padding myself.

And while we're at it: Wouldn't it be cool if I could support
[Multiboot](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)?
Multiboot is a standard that GRUB and other bootloaders support: A kernel is a
regular ELF file with a magic header in the first 8 KB. The bootloader finds
it, loads the kernel into the right address, sets up 32-bit protected mode, and
jumps to the entry point. The kernel gets passed the address of a nice struct
with the memory map, the command line, and whatever modules the bootloader was
told to pass along. QEMU by the way understands Multiboot natively, meaning you
can load Multiboot kernels directly, without needing to write the files to a
virtual disk that QEMU then has to emulate.

That was the moment `loadvm` was born: I had a tool now that could load and
execute arbitrary programs in hardware isolation. I had learned a lot on the
way, and I think you can see how satisfied my tinkerer's heart was with my
creation, when you look at the
[examples](https://gitlab.com/twied/kvmdemo/-/tree/main/examples) I wrote for
the project.

This is where the project could have ended, and I would have been happy with
it. But there was another thought, at the back of my mind, one that wouldn't
quiet down:

Isn't Linux a Multiboot kernel too? Would it be possible...?

## Linux

Some months, and actually years passed before I found the time to continue this
project in earnest, but eventually I. JUST. HAD. TO. KNOW. Is it possible to
boot a Linux kernel with this laughably simple VM?

Firstly, I bolted on an SDL window that displayed whatever the guest wrote to
the VGA text buffer at `0xB8000`. Crude, unsynchronized, wasting CPU cycles,
but it worked.

Next, output anything the guest `outb`s to port `0x3F8`, the serial port.
Ignore anything else, handshakes, status, just show the bytes.

Then, learning how KVM lets you configure interrupts (or rather: the irqchip(s)
of your VM), and adding that to libkvmmm. Becoming really, really frustrated
because suddenly my VM wouldn't let me create CPUs anymore. Figuring out that
there is an undocumented requirement to leave a 4 KB memory hole for the lapic,
and that requirement is not checked immediately but only when the VM is
attempted to be instantiated, hence only reported at the "create CPU" step with
a very unhelpful error code ("device exists"). Write a [pair of
](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d82d77c6430ccd4482b57fc1d22b80649ef71ad8)
[patches](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=99607da19b66ca6649088176ecaaa83eeae780b7)
to the linux kernel mailing list. Get them accepted.

And finally, FINALLY... realizing that I had misremembered. Yes, both QEMU and
GRUB can boot a Linux kernel directly, but Linux' "bzImage" kernels are not
multiboot.

I was disappointed at first, and honestly: ready to give up. It was just an
experiment after all, just that funny question in the back of my head that I
could now answer with "no". If I hadn't stumbled across the documentation for
the [linux boot
protocol](https://www.kernel.org/doc/html/latest/arch/x86/boot.html). A lot
more fields in the info structure that has to be passed to the kernel, but
overall... the requirements weren't actually that hard: Calculate where the
"code" content of the kernel file begins, and copy that part of the file to a
specific ram address. Minimally different environment than what I set up as
"default" protected mode. Tell the kernel how much RAM there is. Set the
instruction pointer and go for it.

That year's Day of Learning had come and gone, and it was actually in the
middle of the night somewhen on the following weekend, when I saw the first
kernel panic on my makeshift VM display. That must have been the happiest
kernel panic of my life. It meant the kernel had made it through its
decompression stub, through the early setup, through the page table
initialization, and all the way to the point where it realized it had no root
filesystem, no console, no anything, and called it quits. It *booted*. It
booted enough to panic. FUCK YEAH!

The project was officially back from its hiatus, and no longer bound to any
schedule. I worked on it basically whenever I had some spare time at night. I
refactored, separated this new thing into its own executable which I began
calling "terrarium" (because "sandbox" is so hackneyed), improved serial port
emulation, and added initramfs loading. And then, one afternoon, the kernel
printed its boot messages to my terminal, ran my init, and printed "Hello
World."

It was exciting. I had done it: My "VM" could "boot" "Linux". It was also, at
this point, fairly useless, as every set of quotation marks in that sentence
came with a huge footnote of limitations. There was no storage, no graphical
display, no input, the kernel log was chock-full with warnings about devices
not existing or failing to respond properly, ACPI tables completely absent, not
even a PCI bus, and the interrupt controller gave me trouble as well.

## BIOS boot

When I had found out that my multiboot support wouldn't buy me anything with
loading Linux kernels, I had contemplated other directions I could take with
this project. And, as it happens way too often with me, another silly idea took
root in the back of my head: How much BIOS do I need to emulate for GRUB to be
able to boot Linux?

This side-loading a Linux kernel business is all fun and games, but... being
able to just load a disk image, now THAT would be impressive. "Just" load the
boot sector at `0x7C00` and start the CPU in 16-bit real mode. Let GRUB figure
out the rest, that's its job after all.

The answer is: Yes and No. Following the BIOS boot specification really is that
trivial: Load the boot sector to ram, initialize a few registers, and off it
goes. Intercepting interrupts isn't too complicated either: Write a "BIOS" that
executes an `outb` instruction that triggers an IO VM_EXIT. If the instruction
pointer is in your "BIOS" segment, you know that the guest called an interrupt.

GRUB is actually quite modest in which interrupts it requires: interrupt 10h
(video: put character, set cursor), interrupt 13h (disk: read sectors),
interrupt 15h (memory map), and interrupt 16h (keyboard: read key). GRUB drew
its menu, accepted keyboard input, and tried to boot whatever was on the disk
image.

Linux on the other hand, is not so frugal. It calls *looots* of interrupts, and
it needs them to work *correctly*. There is virtually no room to cheat or take
shortcuts. The disk access related interrupts alone were way more work than I
had bargained for.

## Good old DOS

You might slowly recognize the pattern: Just as I was ready to archive the
project again, another question quietly popped up in my head: If Linux was too
demanding, how would an OS from less sophisticated times fare? A time when
reading sectors and putting characters on the screen (functions that I now
supported) was about the extent of what personal computing could do? Coming by
a legal copy of an early MS-DOS version is not trivial, so I tried my luck with
[FreeDOS](https://freedos.org/). There were still a few interrupts I had to
implement or stub out, but very soon, I was presented with this:

![FreeDOS 1.4 booting in terrarium](https://gitlab.com/twied/kvmdemo/-/raw/main/terrarium/kernels/freedos/screenshot.png)

FreeDOS 1.4, booting from a floppy disk image. "FreeDOS runs on
Terrarium! :-)" was typed by me, just to make sure keyboard input
worked. And it did.

---

At this point, the project had a name: Terrarium. A small, enclosed environment
for running things. It had three boot modes (raw binary, multiboot, and Linux
boot protocol), a C++ wrapper library, a floppy disk emulation, a VGA text and
graphics display, and a PS/2 keyboard. All of its device emulation was what I
would generously call "legacy": incomplete shims for real hardware, handling
just enough I/O to not crash the guest. It could run test kernels, it could
panic a Linux, and it could boot FreeDOS. Not bad for a stack of learning days
and late evenings, and its humble beginnings of running a single instruction in
a 50 lines "virtual machine".

The code is on [GitLab](https://gitlab.com/twied/kvmdemo).

PS: Well, erm, I just had another idea: How about using VirtIO...?
