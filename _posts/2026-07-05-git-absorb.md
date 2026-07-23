---
title: "TIL: git absorb"
date: 2026-07-05
tags:
  - git
  - tools
  - TIL
---

`git absorb` looks at your unstaged changes, figures out which commits on your 
feature branch they belong to, and creates the right `--fixup` commits 
automatically. Follow up with `git rebase --autosquash` to fold them in, or use 
the `-r` or `--and-rebase` argument to `git absorb` to run the rebase 
automatically.

It replaces the manual workflow of staring at a diff, deciding "this hunk goes 
with commit 3, that one with commit 7," and running `git commit --fixup` for 
each one. I have been doing that manually for years. And I really don't want to 
talk about how often fixes have ended up in the wrong commit anyway, regardless 
of how careful I have been.

Curious what else I've been missing.

Man page for git absorb: 
[git-absorb](https://man.archlinux.org/man/git-absorb.1.en)

Install with `cargo install git-absorb` or through your distribution's package 
manager (Fedora: `dnf install git-absorb`, Debian: `apt install git-absorb`).
