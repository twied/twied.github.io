---
title: "The Good, the Bad, and the Container"
date: 2026-11-23
excerpt: >
  Containers share the host kernel. Every container is one kernel bug away from
  a breakout. What if you could run a Docker image inside a real VM instead?
tags:
  - terrarium
  - kvm
  - virtualization
  - containers
  - security
---

In the [previous post]({% post_url 2026-10-25-for-a-few-ioctls-more %}),
terrarium learned to boot a full Debian installation with VirtIO devices. A
real Linux distribution, running in a VM monitor I wrote from scratch. A sane
person would have declared victory and moved on, but I have had another one of
my infamous questions pop up in my mind.

## The problem with containers

A container is a process with extra namespaces. It has its own PID namespace,
its own network namespace, its own mount namespace, and a handful of other
isolation mechanisms that make it *look* like a separate machine. But it shares
the host kernel. Always funny seeing a Fedora kernel while in a Ubuntu user
space, but that is besides the point right now: Every system call the container
makes is handled by the same kernel that runs the host. Every one of the
hundreds of syscalls in the Linux kernel is part of the attack surface.

This is not a theoretical concern. 2026 has been a good year for kernel
vulnerabilities, to name just a few:

- [CVE-2026-43499](https://www.techtimes.com/articles/319914/20260708/public-exploit-turns-15-year-linux-kernel-flaw-5-second-root-attack.htm)
  ("GhostLock"): a use-after-free in the kernel's rtmutex path, hiding in the
  kernel since 2011. A public exploit gives any local user full root in five
  seconds. Container escape confirmed.

- [CVE-2026-53362](https://aicybr.com/blog/linux-kernel-cve-2026-53362-container-escape-kev):
  an out-of-bounds write in IPv6 fragmentation. CISA added it to the Known
  Exploited Vulnerabilities catalog. Container escape confirmed. Requires only
  local code execution and the ability to create network namespaces, which
  unprivileged containers have by default.

- [CVE-2026-31431](https://cert.europa.eu/publications/security-advisories/2026-005/)
  ("Copy Fail"): a logic flaw in the kernel's crypto subsystem, present
  since 2017. The public exploit is ten lines long and works 100% of the time.

AI-assisted fuzzing is [surfacing these at an uncomfortable
pace](https://www.tomshardware.com/software/linux/linux-kernel-nears-2-000-cves-per-release-as-ai-bug-hunters-scour-40-million-lines-of-code-maintainers-say-they-are-completely-overwhelmed).
The Linux kernel is approaching 2,000 CVEs per release. A vulnerability in a
syscall path means any container on the host can potentially break out. It
doesn't matter how carefully you configured your seccomp filters or how locked
down your AppArmor profile is. If the kernel itself has a bug in a reachable
code path, the container boundary is an illusion.

In a KVM-based VM, every one of these exploits would fire against the guest
kernel instead of the host. The attacker gets root inside the VM and then has
nowhere to go. This is not a terrarium-specific property; it's how KVM works.
Any VM monitor built on KVM gets this for free.

To be clear: VM isolation is not absolute either.
[Januscape](https://thehackernews.com/2026/07/16-year-old-linux-kvm-flaw-lets-guest.html)
(CVE-2026-53359), disclosed in July 2026, is the first KVM guest-to-host escape
on x86 that works on both Intel and AMD. A 16-year-old use-after-free in KVM's
shadow MMU. Google had been running its
[kvmCTF](https://github.com/google/security-research/blob/master/kvmctf/rules.md)
bug bounty for two years, offering $250,000 for a full VM escape, before anyone
claimed it. That two-year gap tells you something about how hard these bugs are
to find and exploit compared to the container escapes listed above. But they do
exist, and it would be dishonest to pretend otherwise.

And it gets worse. Even setting kernel bugs aside, the "sandboxing" solutions
that are supposed to protect your desktop don't actually isolate your data. A
malicious flatpak or snap application can read your home directory. It can `rm
-rf ${HOME}`. It can read your SSH private key and send it somewhere. It can
mine bitcoin on your CPU. It can encrypt your ~~porn~~ wedding pictures and
ransom you.

Docker, podman, flatpak, snap: they all share this fundamental limitation. They
are isolation by convention, enforced by a kernel that is itself the weakest
link, with filesystem boundaries that are more suggestion than enforcement.
None of them actually *sandbox* in the hardware sense of the word.

## Userspace without kernel, kernel without userspace

A container image is, at its core, just a filesystem tree. It may be layered,
deduplicated, content-addressed, and wrapped in six layers of JSON manifests,
but after all is said and done, it's `/bin`, `/lib`, `/etc`, and all the other
directories you'd find in a Linux installation, minus the kernel. It is
userspace without a kernel.

Terrarium, on the other hand, boots kernels. Give it a bzImage and an
initramfs, and it will set up a VM, load the kernel, and run it. It is a kernel
without userspace.

Now here's my question: What if we take a container image, and a *copy* of the
host kernel (not a shared kernel with namespace isolation, a *completely
seperate kernel*), put it in a blender and season with a custom little init
script? The attack surface is no longer every syscall in Linux. It is `KVM_RUN`
and the handful of device emulation paths that handle VM exits.

## How hard can it be

The conversion from container image to VM image turned out to be way easier
than anticipated. In part, because I cheat and let podman do the heavy lifting
of locating and downloading the images and only concern myself with unpacking
the stacked tarballs that the OCI images served by Docker Hub and the likes
actually are.

```sh
./terrarium/oci/convert.py \
  docker.io/library/alpine:latest alpine.img
```

The trickier part was init. A normal Linux boot expects an init process
at `/sbin/init` (or whatever `init=` points to). Container images don't
have one; they expect the container runtime to set up the environment
and exec the entrypoint directly. I wrote a minimal init in C that does
just enough to be a proper PID 1: mount `/proc` and `/sys`, set up the
console, read the entrypoint from the kernel command line, exec it, and
reap zombies. A few dozen lines. It gets injected into the initramfs
alongside the container's root filesystem.

The networking question answered itself. Terrarium uses vsock instead of a
virtual network interface. The container has no network access by design. No IP
address, no routing table, no way to reach the internet or the host network.
Communication with the host goes through vsock, which is a point-to-point
channel, not a network.

Container-to-container communication would need an orchestrator on the host
that routes vsock streams between VMs. But this is deliberate, and there is no
"terrarium-compose" on the horizon just yet. Traditional containers start with
full capabilities and then try to take them away with namespaces, seccomp
filters, and AppArmor profiles. Terrarium starts from zero. No network. No
filesystem access. No shared kernel. If a container needs to talk to another
container, that capability has to be explicitly granted. Grant from a
zero-capability baseline instead of revoking from a full-capability one. That
is the kind of security model I actually trust.

## It works

```sh
terrarium /boot/vmlinuz-$(uname -r) \
    --module terrarium/oci/initrd.cpio \
    --disk alpine.img \
    --display=off \
    -- console=hvc0 entrypoint=/bin/sh
```

Outputs:

```
Who put that whale in the aquarium?
terrarium oci runner v0.1

/ # cat /etc/os-release
NAME="Alpine Linux"
/ #
```

This is not "Alpine Linux running in a VM." QEMU can do that. This is the
Alpine *container image*, pulled straight from Docker Hub, booted as a VM with
a copy of the host kernel. The container never shipped with a kernel. It never
needed one, because containers share the host's. Terrarium takes that kernel,
pairs it with the container's filesystem, and boots both in hardware isolation.
The result is something between a container and a virtual machine: the
convenience of a container image, the isolation of a VM. A "virtual container"?
A "container machine"? "[Tech
heresy](https://en.wikipedia.org/wiki/Warhammer_40,000#Setting)"? I don't have
a good name for it yet.

The host kernel never sees the container's syscalls. The container can do
whatever it wants inside the VM, and the worst it can do to the host is trigger
a VM exit that terrarium doesn't handle, at which point the VM simply
terminates. No privilege escalation, no namespace escape, no shared kernel.

You could run actively hostile code in there and sleep well at night.

## What it doesn't do

I would be lying if I said this was production-ready. It is not. The
list of things that a real container runtime does and terrarium doesn't
is long:

- **Environment variables** (`docker run -e`): not passed through. Solvable via
  the kernel command line or a config file on the disk image.
- **Port forwarding** (`docker run -p`): not implemented. Would require
  vsock-based proxying between host ports and guest services.
- **Volume mounts** (`docker run -v`): not implemented. Would require a
  virtio-fs or 9p filesystem device, or a shared block device with appropriate
  fencing.
- **Image caching**: every run converts the full image from scratch. No layer
  caching, no deduplication.
- **Startup time**: booting a kernel is slower than starting a container
  process. We're talking seconds, not milliseconds.

None of these are fundamental obstacles. They are engineering work, not
research problems. But they are a lot of engineering work, and this is a hobby
project.

## The landscape

Terrarium is not the first project to put containers in VMs. The idea that
VM-level isolation is stronger than namespace-based isolation is not new.
Several projects have explored this space:

- [Kata Containers](https://katacontainers.io/) runs containers inside
  lightweight VMs using QEMU or Cloud Hypervisor. It's the most mature project
  in this space, with OCI-compatible runtime integration and Kubernetes
  support. Terrarium is a toy by comparison, but it's also about 10,000 lines
  of code versus Kata's considerably larger footprint.

- [Firecracker](https://firecracker-microvm.github.io/) is Amazon's microVM
  monitor, purpose-built for serverless workloads. Extremely fast boot times,
  minimal device model. It does not implement OCI container support directly,
  but AWS Lambda runs on it.

- [gVisor](https://gvisor.dev/) takes a different approach entirely: instead of
  running a real kernel in a VM, it reimplements the Linux syscall interface in
  a userspace kernel written in Go. The container's syscalls never reach the
  host kernel. Clever, but it means maintaining a parallel implementation of
  hundreds of syscalls, and compatibility is never quite complete.

What makes terrarium's approach different is mostly its simplicity. It doesn't
try to be compatible with the container ecosystem. It doesn't integrate with
Kubernetes. It just takes a container image, boots it in a VM, and gives you a
shell. That's it. Whether that simplicity is a feature or a limitation depends
on what you need.

---

This is the end of the terrarium series, at least for now. What started as
"it's just a few ioctls" turned into a 10,000-line hypervisor that can boot
FreeDOS, run Debian, and sandbox container images with hardware isolation. I
learned more about PCI, ACPI, VirtIO, and the Linux boot protocol than I ever
expected to. And I got to write the sentence "Who put that whale in the
aquarium?" into "production" code, how's that?

The code is on [GitLab](https://gitlab.com/twied/kvmdemo).
