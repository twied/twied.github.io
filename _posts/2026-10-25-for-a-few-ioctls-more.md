---
title: "For a few ioctls more"
date: 2026-10-25
excerpt: >
  My homegrown VM monitor could boot FreeDOS, but Linux needed more than legacy
  device shims. VirtIO promised clean, simple device interfaces designed for
  virtualization. How hard could it be?
tags:
  - terrarium
  - kvm
  - virtualization
  - virtio
---

Imagine for a second that you are an operating system. The user presses the
power button of the computer you live in and you wake up from your slumber. The
first thing you do is a roll call of the hardware. After all, you need to know
what's there to use it. Now imagine your report says that your computer has a
2026, state-of-the-art, latest-and-greatest shiny CPU from BigVendor™ with
all the features, plenty of cache, and Gigahertz upon Gigahertz of speed. And
look at the system memory! *Dozens* of Gigabytes, all for you to use.

Now the graphics card, but that is funny. It is a Cirrus Logic CL-GD5446,
produced in 1994, thirty years ago, requiring connectors that computers for the
last fifteen years or so didn't even have anymore. And yet, it's there!
The sound card is equally puzzling, a Creative Sound Blaster 16, from 1992.

The explanation of what's going on is simple: You live inside a virtual
machine, an (unusually configured) QEMU to be exact. Those are the graphics
card and sound card that QEMU emulates for the guest, feature-for-feature,
bug-for-bug, quirk-for-quirk, so we can run and enjoy old software on new
computers.

The exact emulation of legacy hardware comes at a price though: It is
incredibly complex to get right. Sometimes even the most minuscule of timing
issues and corner cases can make a difference. At least for equally legacy
software, if your goal is faithful recreation of an environment from that age.

There are other use cases for virtual machines though, where the software
running inside is not from yester-decade, and the VM is used for isolation, not
hardware emulation. In that case it is kind of wasteful for both guest and host
to go through the motions of pretending to either *be* ancient hardware or
*speak to* ancient hardware, with all its quirks, bugs, and idiosyncrasies.
Wouldn't it be much better for everyone involved in that case to just drop the
charade and say "*I* know this is a virtual environment, *you* know this is a
virtual environment, why don't we act like grown-ups and you just tell me what
you want to display, send over the network, or play on the speaker"?

## VirtIO

Doing away with the legacy devices and their unpleasantries is exactly what
[VirtIO](https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html) is
for. It is a standard for virtual device interfaces, based on the concepts of
shared memory and ring buffers, called "virtqueues", and relies on the fact
that the host can rather easily read and write guest memory. Add a notification
mechanism for both host and guest to let each other know that there's new data
to process, and you've got yourself a simple and performant way to do IO.

## Terrarium

I have written my own virtual machine manager, "terrarium" (see the last [blog
post]({% post_url 2026-10-09-a-fistful-of-ioctls %}) for how that came to be). Its
support for legacy devices is... "minimal", to say the least. It is enough to
boot FreeDOS, and I am very proud of that fact. But trying to run Linux in that
environment? Serial output that only handled writes, a VGA buffer that was just
a chunk of memory at `0xB8000`, a PS/2 keyboard that could barely deliver a
keystroke? Every real device Linux tried to probe hit a wall of unimplemented
I/O ports, and eventually Linux would just panic and -- rightfully -- give up.

There was this idea sitting in the back of my head: Can I teach terrarium how
to virt-io? And if I added VirtIO devices, would that be enough to *actually*
boot Linux?

## First attempt: MMIO

VirtIO devices can be exposed to the guest through two different methods (three
actually, but I'm not touching S/390): PCI and MMIO. As terrarium has no PCI
bus and MMIO is rather easy to deal with, I went with MMIO.

Now, what's MMIO? x86 has access to two different address spaces, "I/O" or
"port I/O" or "PIO" on one hand, and "memory address space" or "physical
address space" on the other. PIO is the one you access through `in` and `out`
instructions, memory is accessed through `mov` and other instructions.
Generally speaking, PIO is stateful, reading and writing is not
(necessarily) "storing data at a location" but rather "sending / receiving
messages to / from a port", hence the name *port* I/O. In contrast, memory is
just "storage cells". Values you write at an address stay there, can be
retrieved arbitrarily often and, most importantly, don't change unless you
write a new value there.

When a guest in KVM accesses PIO, a "vm exit" occurs, a fancy name for "the
function call to `ioctl(fd, KVM_RUN, ...)` returned". And it returns an exit
reason of `KVM_EXIT_IO`, letting the host know that it is to react in whatever
way it deems fit to the guest reading (`in`) or writing (`out`) data to some
I/O port. When done, the host simply calls `KVM_RUN` again, and the guest
continues executing.

When a guest in KVM accesses a memory address, normally nothing extraordinary
happens, at least not if the host instructed KVM to map that guest address to
some host memory with `KVM_SET_USER_MEMORY_REGION`. But accessing an unmapped
address does result in a similar vm exit as with PIO, but this time with exit
reason `KVM_EXIT_MMIO`. This is how one can map guest memory addresses to some
emulated device: Remove the memory backing for some address range, and handle
reading and writing to those addresses yourself. Note that this address range
loses its "memory" semantics and behaves like io ports: From the guest's
perspective, any side effect can happen when accessing such an address. Memory
mapped I/O, or "MMIO", plain and simple.

I believe you can already see how one could implement communication with the
host using MMIO. But the naive and direct solution of just reading and writing
data byte-by-byte to some magic address would be quite slow: Every access is a
vm exit and a context switch after all. Instead, VirtIO only maps the "device
registers" via MMIO. Think a C "struct" that is located at a specific address,
and that both guest and host can read and write.

The first 256 bytes of that "struct" are common for all VirtIO devices,
followed by zero or more device-specific values. Let's look at some of these
registers: Register 0, four bytes long: always contains the value `0x74726976`,
or the string "virt" in ASCII (or "triv" in
[Lilliput](https://en.wikipedia.org/wiki/Endianness)). Register 2, offset 0x08,
also four bytes: Specifies the type of VirtIO device, network interface, GPU,
mouse, or any other type. The register at offset 0x10 contains a bitfield of
features supported by the device. Offset 0x14: here the guest writes which of
those features it understands, thus negotiating the actually used set of
features with the device. There are also registers that the guest fills with
the addresses of ring buffer pairs containing chunks of memory: One for
messages to the host, one for messages from the host. And yet another register
serves as notification to the host: The guest writing to that register tells
the host that there is data available to process, using the "IO" part of
"MMIO". For the reverse direction, the host fires an interrupt if it has data
for the guest. Of course there are some more details with regard to the
configuration of VirtIO devices via MMIO, but by and large, that's it.

There is just one little blemish to the whole thing, and that is that the guest
cannot auto-detect VirtIO devices mapped via MMIO. The "solution" is to pass a
description of the device to the Linux kernel with a device tree file or via
command line. Add for example `virtio_mmio.device=0x200@0xd0000000:5` to the
command line for a device at address `0xd0000000` with `0x200` bytes of device
registers, using interrupt `5` for host-to-guest notifications. Not
particularly elegant, but serviceable, at least for now. It is easy enough to
silently append these lines to the command line, when someone is booting a
Linux kernel with terrarium.

## Transferring data

Now that the VirtIO device is configured and host and guest shook hands, how do
they actually *do* transfer data? The answer is "virtqueues". Each VirtIO
device has one or more virtqueues. A virtqueue is, in high-level terms, a
one-directional pipe, transferring data packages either from the guest to the
host ("driver to device"), or from the host to the guest ("device to driver").

Each virtqueue has a *size* that is negotiated at boot and doesn't change. That
is the maximum number of packages that can be "in flight" at any given time.
Next to the size, each virtqueue has three elements:

- A descriptor table, an array of size `size`. Each entry may describe a region
  of memory by address and length. The driver can change that mapping over time
  and not all entries need to be in use at all times, but there can only be up
  to `size` memory regions used in parallel.

- A ring buffer of available packages: This one confused me for quite a
  while, as a ring buffer needs *two* pointers (or, as is done here, *indices*
  into the descriptor table) to work: One for the next write operation, and one
  for the next read operation. But the VirtIO specification only contains *one*
  `index` field. The answer to that riddle: it's the write pointer. The read
  pointer is to be privately maintained in the device, i.e. the host code. Both
  read and write indices monotonically increment, but access the ring buffer
  modulo `size`.

- A ring buffer of used packages: Very similar to the available-ring
  buffer, but has an extra "length" field per package. Once the device is done
  with the data sent to it via the available-ring, it puts the packages in the
  used-ring to signal to the driver that the underlying memory is no longer
  used and can be repurposed.

This design is quite elegant and can help avoid copying data. Imagine the
situation that a program in the guest wants to print "Hello world!" to the
console. The guest program executes a `write` syscall to the guest kernel. The
guest kernel can then take an unused slot of the virtqueue descriptor table,
and point it directly to the (guest-physical address of the)
guest-userspace-program provided memory, and insert the slot number into the
available-ring. The host is notified, sees the new entry in the ring, and can
read the data from the original location in the guest-userspace-program
directly, without any intermediate duplication.

To create a "read" virtqueue, the driver creates a couple of "empty" packages.
"Empty" not in the sense that the package size is zero, but rather "some memory
without any meaningful content in it" and submits it to the available-ring.
Since it is a "read" virtqueue, the device knows that the data in those
packages is not some message to the host, but backing storage for it to put
*its* messages to the guest into. And if the guest is smart, it can do the same
trick as before again, directly point to e.g. the userspace buffer that some
program prepared and expects to retrieve its data in, eliding needless copies
again.

Neat, huh?

## My first VirtIO device

The first device that I implemented was VirtIO Console. It is beautifully
simple, and requires merely two queues: One for sending data from host to
guest, and one for guest to host. The hardest part about all of this was to
realize that the reason I saw no output, was not my VirtIO device not working
but rather a bug in my output code where I ended up double-locking a mutex.

Once I had fixed that (and then fixed it again, because the fix wasn't correct
and multithreaded programs are hard), I could pass a `console=hvc0` to the
kernel command line. Lo and behold, Linux obliged, and I got my first output
via my new, shiny, and homemade VirtIO Console device. And in contrast to the
botch job I called "UART interface" so far, it wasn't output only, I could send
input as well, proving the whole concept of VirtIO in terrarium feasible.

Time to celebrate!

## PCI

I am telling things a little out of order here: In reality, I continued the
MMIO path quite a bit further, but for the sake of the narrative, allow me to
cut to the chase: The celebration about VirtIO working was premature, as I
realized two things: One, I didn't want to work with device tree files and
injecting kernel command line arguments is incredibly brittle, and two,
adjusting the command line is impossible if I ever want to boot an OS from
disk.

So... PCI, huh.

PCI devices are discoverable. The operating system probes the PCI bus at boot,
reads configuration space headers, finds VirtIO devices by vendor and device
ID, and loads the right drivers. No magic command line arguments. No hardcoded
addresses. The guest just scans and finds what's there. No special-casing for
Linux and no command line manipulation.

I mentioned in the last post that I dabbled in OS development in Uni, but never
got PCI support working, didn't I? Well, it was time to face the foe that I
couldn't beat twenty years or so ago.

At this point I need to make a confession and admit to cheating somewhat: I
employed the help of some AI to ask it oh-so-formal and well-phrased questions
as "Why the \*\*\*\* is this \*\*\*\* not working, \*\*\*\*!". And I have to
say, 2026 AI is nothing like the early LLM models that I snidely dismissed back
then. I got actually helpful answers and explanations back, that helped me
understand what I got wrong, both now and back then.

Long story short: PCI itself is a "device" that handles two four-byte addresses
in the PIO address space, `0xCF8` and `0xCFC`. An operating system assembles a
PCI "address" (a tuple of bus, device, function and register number) into a 32-bit
value and writes it to address `0xCF8`. Reads and Writes to address `0xCFC`
then read or write four bytes from the configuration space of the addressed PCI
device. Classic register banking. If no device is there, you will read
`0xFFFFFFFF` from `0xCFC`, regardless of what you write to that address. This
is how an Operating System detects PCI devices: Iterate over all possible
addresses, and note down where something responds when you poke it.

Regular PCI devices (i.e. not bridges) have a common layout for the first 64
bytes (or 16 registers) of configuration space. The rest of the 256 bytes of
configuration space that each PCI device has, is filled with device/vendor
specific data. Notable common registers are:

- Device ID and Vendor ID: Uniquely identify the vendor and device type. Vendor
  ID "8086" for example, is of course "Intel Corporation". Device ID then
  differentiates different devices from that vendor.

- Class and subclass code: Identify the device class, e.g. class "0x01"
  identifies the device as a Mass Storage Controller with subclass "0x01" being
  IDE Controllers vs. subclass "0x02" Floppy Disk Controller.

- Base address ("BAR") 0 to 5: PCI devices, generally speaking, want to be
  mapped to some address space, so that the system can access the device's
  functionality. Imagine for example a graphics card, that maps its VRAM or
  framebuffer directly to system memory address space: Writing pixel data to
  that region directly appears on the screen. BARs can also map to PIO address
  space, but I won't use that in terrarium and ignore that possibility in this
  description. The OS interrogates these values by writing `0xFFFFFFFF`, and
  reads it back to get the size of the BAR. Afterwards it sets some memory
  address of its choosing, and in theory the device is ready to go.

- Interrupt pin and interrupt line: Four physical wires go from the PCI bus to
  the interrupt controller chip (or whatever assumes its responsibilities) on
  the motherboard, named "INTA#", "INTB#", "INTC#", and "INTD#". The "interrupt
  pin" register tells the operating system which line the device will activate
  ("assert") when it wants to raise an interrupt. The mapping from A, B, C, or
  D to actual interrupt numbers is the job of the interrupt controller, as
  programmed by the operating system, typically somewhere in the range of
  interrupts 9 to 12. Note that this means that the device does not and cannot
  know the number of the interrupt it raises. That's what the "interrupt line"
  register is for: The operating system fills that one out, for the driver to
  read. Multiple PCI devices can have the same interrupt assigned: The
  interrupt is level based, not edge based, and when the interrupt fires the
  operating system will simply iterate over the drivers of all devices with
  that interrupt. Each driver checks its device's status register, drivers of
  devices that didn't fire report "not mine", and the one (or ones) that did
  handle what they need to handle clear their interrupt condition. Once all
  devices have deasserted, the line goes low.

- Capabilities pointer: A pointer to a linked list that lives in the device
  config space beyond register 15. That is the extension method to report more
  information about the PCI device than what was anticipated when PCI was
  standardized. Each capability "node" in that linked list has fields "type",
  "size", and "next", so that even an operating system that does not understand
  a particular capability can still traverse the list. Capabilities can be
  things like power management (if the device supports suspend / resume
  states), MSI (device can send interrupts by writing to some memory address
  instead of using INTA#-INTD#), PCI-Express (link speed, lane width, error
  reporting), or outright "vendor specific" (whatever the vendor wants the
  device to let the driver know).

That sounds complicated, and frankly, it is, but all in all, it could have been
worse. I wrote a demo kernel that just raw-dumped the entire configuration
space of all PCI devices it could find, and ran it in QEMU. I wanted to see
what the grown-ups put there, imitation being the highest form of flattery and
all of that, you know. Armed with the always helpful [osdev
wiki](https://wiki.osdev.org/PCI) and my new artificial friend at my side, it
was a manageable task to first understand the data and then adjust and
reproduce it in terrarium. It also helped that I liberally ignored half of the
specification, as I won't support BAR PIO mapping or 64-bit BARs any time soon.

Getting the first PCI device *visible* in Linux went quite quickly. Getting it
*working* took a bit longer. One gotcha I found is that you have to pay close
attention to which configuration registers are read-only, write-only and
read-write. Get that part right, or you will get the weirdest results. There
are some four-byte registers where only some bytes of it are read-write, and
some other bytes must survive writes. Operating systems rely on that.

Once terrarium [had a PCI bus](https://youtu.be/RVkmlOU-Nck?t=24), implementing
VirtIO over PCI was actually not that hard. The most notable difference is that
the VirtIO configuration registers are not exposed as one big blob, but rather
several smaller groups of data pieces, pointed to by several PCI "vendor
specific" capabilities, all living somewhere in BAR space. As there was no
reason to have more than one BAR, I crammed everything in BAR0 for all my
VirtIO devices.

Soon, I had my console VirtIO device working (again), this time without any
kernel command line shenanigans. Yay (again)! Alas, the good mood was not to last.
Again. The kernel log from the guest had shown some warnings regarding my PCI
devices, but I had foolishly ignored those because, well, it did work. And
after implementing the second or third VirtIO device, suddenly the first one
stopped being detected properly. What was going on?

## ACPI

Well, there is a bit more to the whole "OS assigns BAR addresses and interrupts
and everybody lived happily ever after" business than I previously mentioned.
The first indication to that was the warning messages that Linux produced:

```
[    1.433343] pci 0000:00:02.0: BAR 0 [mem 0xfe003000-0xfe003fff]: can't claim; no compatible bridge window
(...)
[    1.466288] pci 0000:00:02.0: BAR 0 [mem size 0x00001000]: can't assign; no space
[    1.467263] pci 0000:00:02.0: BAR 0: trying firmware assignment [mem 0xfe003000-0xfe003fff]
[    1.468333] pci 0000:00:02.0: BAR 0 [mem 0xfe003000-0xfe003fff]: conflicts with RAM buffer [mem 0xfe000000-0xffffffff]
[    1.469682] pci 0000:00:02.0: BAR 0 [mem size 0x00001000]: failed to assign
```

Linux wouldn't accept BAR placements in the MMIO region without being
explicitly told "this address range is yours for PCI." Pre-filling the BARs
wasn't enough, Linux was convinced that this region is a potential RAM buffer,
and refused to place BARs there. Even though terrarium pre-filled the right
addresses, it just wouldn't trust those values.

Linux *normally* uses ACPI to assign PCI resources, and only if that is
unavailable falls back to whatever the "BIOS" left in the BAR registers. Since
that method had failed me, it was time to open the next big can of worms: What
is ACPI anyway?

Depending on who you ask, it's either the "[Advanced Configuration and Power
Interface](https://wiki.osdev.org/ACPI)" or "[a complete design disaster in
every way](https://www.linuxjournal.com/article/7279)". Having had to implement
it, I am leaning towards the latter: Specification is about 1000 pages, comes
with its own bytecode language (AML), its own namespace hierarchy, and its own
event model.

The whole things works roughly as follows: The BIOS puts a magic marker (the
string "RSD PTR ", mind the trailing space, who comes up with that stuff...)
somewhere between addresses `0x000E0000` and `0x000FFFFF`, followed by a
checksum byte, six characters of vendor "name" (why six? what an odd number),
followed by one byte of format version ID, followed by four bytes with a memory
address. That is your "Root System Description Pointer", or "RSDP", version
1.0. There is a version 2.0 that supports 64-bit addresses, but let's keep
things ~~simple~~ as simple as is possible with ACPI.

The address in the RSDP points to the RSDT, or "Root System Description Table".
It too begins with a magic marker, thankfully this time without confusing
space: "RSDT". It contains format version, checksum and vendor name fields as
well, but most crucially also a `length` field that determines the number of
pointers that follow this structure. Why the RSDP and RSDT have to be two
separate structures is beyond me, they could have just as easily been joined
together, but who am I to judge...

Each of those RSDT pointers point to another four-letter-name structure. For
terrarium, that's "just" two, the FADT and the MADT, but the FADT requires also
FACS and DSDT. Now what's the deal with this jumble of letters? Let's go over
them, one by one:

The FADT is the "Fixed ACPI Description Table". Note that it is the only table
where the magic marker is not identical to its name, but rather "FACP". The
FADT contains information about the power management hardware, for example
where the `pm1a_evt_blk` and `pm1a_cnt_blk` registers live. These are the
registers that ultimately are used to shut the machine off. Other notable
information in this table is `sci_int`, which contains the interrupt number for
ACPI events such as "the power button was pressed". If you have looked at the
terrarium code on gitlab, you might have seen this already implemented, because
the process of me figuring ACPI out was not as linear as I make it look in
these blog posts.

The FACS, or "Firmware ACPI Control Structure" is thankfully something I was
able to completely ignore for terrarium, it just had to (a) be there and (b) be
valid, otherwise completely empty. On regular systems, this structure deals
with resume-after-sleep behavior, and global lock functions.

In the MADT, or "Multiple APIC Description Table" we mostly reiterate standard
/ default values: This table describes the interrupt controllers present in the
system. The only crucial setting that is made here is in the Interrupt Source
Override entry to declare `sci_int` from the FADT as level based, not edge
triggered.

Last but not least, the DSDT or "Differentiated System Description Table" is
where the real magic happens. Black magic, for all that matters, but that is
kind of a given with ACPI. It contains a series of bytecode ("AML") objects and
I am proud to report that I managed to resist the urge to write an AML
compiler. Only the simplest of helper functions that allow me to piece together
sufficiently valid instructions. The DSDT contains effectively a "program" that
describes the system and the devices in the system: Shutting down the computer
(aka putting it into "S5" state)? Here is *what* to write to the `pm1a_evt_blk`
and `pm1a_cnt_blk` registers mentioned in the FADT. PCI bus? Yes, has bus
number 0. And BAR addresses beginning at... say... `0xFE000000`. And here is a
list of the PCI devices and which interrupt number is connected to which
interrupt line is connected to which device. This also makes Linux realize that
these interrupts come from a PCI device, and PCI interrupts are level-triggered
as per the PCI spec. Without this declaration, Linux falls back to reading the
Interrupt Line register from PCI config space, mistakes it as ISA interrupts,
and does not reconfigure the interrupt controller for level-triggered mode. In
short: Linux will miss interrupts and simply hang on boot sometimes.

ACPI is a pain to work with and I am not feeling bad for admitting that I again
used the help of AI to figure things out. I was successful though: PCI now
works flawlessly.

## VirtIO devices

So, now, what VirtIO devices did I implement? I am certainly not going to
implement all of them, there simply are too many different VirtIO device types
for that. The base protocol is the same for every device type: feature
negotiation, virtqueue setup, device status, but I have to draw the line
somewhere.

**Console** as I have already mentioned. Very, VERY useful for debugging when
the kernel is able to let you know what it didn't like and you have no display
yet. Also the only "really" necessary device to get the guest to do some
meaningful work.

**GPU** was next, and significantly more work. I had a "graphical" VGA mode 13h
sort of working before, but it was clumsy and brittle since I attempted to
implement automatic mode switching but ignored most of the status bytes at the
same time. I wanted to get rid of that and was still riding the high from
getting the console to work. VirtIO GPU doesn't just give you a framebuffer
though, it's a command-based interface with resource management, scanouts, and
2D transfer operations. The guest allocates graphics resources (essentially
textures), draws into them, and tells the host to display them. Luckily I can
simply opt to not implement most of it, so no 3D, no cursors, no multi-head,
just a single scanout with 2D blitting. Won't run
[SuperTuxKart](https://supertuxkart.net) at a million FPS, but it works and is still
SO. MUCH. BETTER. than your 80x25 EGA console to work with.

**Input** was straightforward after the GPU. VirtIO input exposes keyboard and
mouse as evdev-compatible event streams. The host translates SDL events into
Linux input events and pushes them through a virtqueue. A few dozen event codes
for common keys and mouse buttons, and the guest has a working keyboard and
mouse. I'm sure you always wanted your virtual machine manager hardcoded with a
German keyboard layout, so there you go.

**Block storage** got me really giddy. I had left the best part for last.
Similar to VirtIO GPU, this device works on a stream of commands ("read 8
sectors starting at LBA 1024" etc). The old `int 13h` emulation of my
pseudo-BIOS was enough for GRUB, so it could read both a Linux kernel and an
initramfs from disk, but then it left Linux stranded, as it was unable to mount
the root file system. With a VirtIO block device, this was now solved, so Linux
should be able to progress further in the boot sequence.

## Networking (or: why not)

The obvious next device would be VirtIO net. Every real VM has a virtual
network interface. But VirtIO net is complicated: you need a TAP device on the
host, you need packet handling, you need to worry about checksums and
offloading and buffer management. And on top of that you need to ask and answer
questions like: Do I want to have the guest be able to make arbitrary internet
connections? I want to have *some* connectivity, e.g. ssh into the guest, but
am I willing to pay the price for that?

[VirtIO vsock](https://man7.org/linux/man-pages/man7/vsock.7.html) solves that
dilemma: It provides a socket interface between host and guest, without any
network configuration. No IP addresses, no routing tables, no DHCP. The guest
connects to a well-known Context ID "CID", the host accepts the connection, and
data flows. It's the VM equivalent of a Unix domain socket.

This turned out to be a much better fit than networking. I like the security
aspect of it, as vsock keeps the guest completely isolated from the network,
and yet allows me to connect into the guest, or set up an explicit SOCKS proxy, if
I positively want the guest to have internet access. Have a look at the
[documentation / tutorial /
examples](https://gitlab.com/twied/kvmdemo/-/blob/main/terrarium/vsock.md) that
I wrote, and see if you can come up with a use case I haven't covered there
already.

## Debian boots

With console, GPU, input, block storage, and vsock, all pieces were in place. I
had a Debian 13 (trixie) VM image lying around from earlier experiments. I had
installed it from a CD image, in QEMU, but otherwise it was pristine. I pointed
terrarium at it, crossed my fingers, and hit enter:

![Debian 13 running in
terrarium](https://gitlab.com/twied/kvmdemo/-/raw/main/terrarium/kernels/linux/screenshot.png)

Dear reader, I am ecstatic to report that terrarium now boots Debian, from an
unmodified (ok, I installed fastfetch for the screenshot) VM image. A real
Linux distribution, running in a VM monitor I wrote from scratch. No tricks, no
QEMU, no libvirt, no third-party libraries. IT. BOOTS. LINUX. Just KVM, about
10,000 lines of C++, and a few ioctls more. I said it last time, and please
allow me to repeat myself: FUCK YEAH!

---

There are things terrarium still doesn't do. It has one vCPU, because
multi-processor support would mean dealing with interrupt routing, APIC
emulation, and memory ordering. There is no migration, no snapshots, no memory
ballooning, no CPU model emulation. It is not a production hypervisor and
doesn't pretend to be. But it boots Linux. It runs a shell, serves web pages,
compiles code.

The code is on [GitLab](https://gitlab.com/twied/kvmdemo).
