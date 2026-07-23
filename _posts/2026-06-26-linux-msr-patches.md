---
title: The msr files are weird
date: 2026-06-26
last_modified_at: 2026-07-21
tags:
  - linux
  - kernel
---

Have you ever read from or written to the `/dev/cpu/*/msr` files? If you aren't
in virtualization, RAPL monitoring tools, or performance analysis tools,
chances are low. But if you have, you might have stumbled across some very
weird behavior:
<!--more-->

- Reading into a buffer that is bigger than 8 bytes (the size of one MSR
  register), does not read consecutive registers, but the same register over
  and over.

- Writing from a buffer that is bigger than 8 bytes does not write consecutive
  registers, but the same register over and over.

- Access to these files is locked behind file mode 0600 *and* `CAP_SYS_RAWIO`.

My attempt to let ordinary users read at least *some* MSR registers in 2023
[failed](
https://lore.kernel.org/kvm/20230530102358.16430-1-twiederh@redhat.com/),
but I want to try again and make at least the reading and writing behavior less
surprising. Patch is [here](
https://lore.kernel.org/lkml/20260626174037.1128563-1-twiederh@redhat.com/).

**Update, 2026-06-26:**

[Review](
https://lore.kernel.org/lkml/33F430D0-EFE6-471F-B4A1-D5D20E1F65AD@zytor.com/) 
is in. Need to drop the restriction to single writes. I can see the argument, 
but I don't know if the overhead of multiple `pwrite` calls would really be so 
noticeable.

**Update, 2026-06-26, but later:**

[Review for v2](
https://lore.kernel.org/lkml/0239cda7-b638-44df-9739-746a7173a577@zytor.com/) 
is here as well, and there goes the restriction to the read function. 
Unsatisfactory. I hope at least the changes to the file comment go through so
that the next developer who stumbles across this knows that there are users for
this behavior.

**Update, 2026-07-21:**

[They did!](
https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?id=b096d79bc1eed9eda57c12ccd76d6797cdba4cf2)
