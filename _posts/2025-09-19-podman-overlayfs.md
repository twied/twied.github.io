---
title: "Podman: two minutes to sub-second"
date: 2025-09-19
tags:
  - linux
  - containers
---

If your Podman containers take over two minutes to start, check your storage
driver:

```
$ podman info --debug | grep graphDriverName
graphDriverName: vfs
```

If it says `vfs`, that is your problem. The `vfs` storage driver copies the 
entire container image on every start. No copy-on-write, no layer sharing, just 
a full deep copy. This apparently is the default you get, when installing 
podman on Debian with "recommended packages" disabled.


Install `fuse-overlayfs`, and run:

```
# WARNING! This deletes all containers, all images, all storage, everything!
$ podman system reset
```

Now the output changes to:

```
$ podman info --debug | grep graphDriverName
graphDriverName: overlay
```

Container startup went from 130 to 0.7 seconds, more than two orders of 
magnitude improvement.
