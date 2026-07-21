---
title: "SysExplorer: a read-only window into iOS internals"
date: 2026-07-22 00:01:00 +0300
categories: [iOS, Internals]
tags: [ios, trollstore, mach-o, iokit, xpc, reverse-engineering, arm64]
image:
  path: https://github.com/lavochkaa/SystemExplorer/raw/main/Screenshot.png
  alt: SysExplorer main screen — process list with RAM and thread stats
pin: true
---

## What this is

[SysExplorer](https://github.com/lavochkaa/SystemExplorer) is a read-only iOS system monitor for TrollStore devices. It surfaces the things the system normally hides behind sandboxing and private frameworks: every running process, open file descriptors, entitlements, Mach-O layout, XPC launch daemons, and the full IOKit device registry.

No hooks, no memory writes, no injection. Just read syscalls behind a UIKit UI.

Targets **iOS 15.0+, arm64**. Built in Swift + Objective-C + C/C++.

## What it shows

The app is built as five independent explorers:

| Stage | What it exposes | How it gets there |
| --- | --- | --- |
| Processes | All PIDs, RAM, threads | `proc_listallpids` + `proc_pidinfo` |
| Process detail | Open FDs, entitlements, binary path | `proc_pidinfo(PROC_PIDLISTFDS)` + `csops` |
| Mach-O inspector | Segments, arch, linked dylibs | `mmap` + manual header walk |
| XPC | LaunchDaemons and their `MachServices` | plist scan of `/System/Library/LaunchDaemons/` |
| IOKit | Device tree, class hierarchy | `IORegistryEntryCreateIterator` traversal |

Everything is a straight system call. The interesting engineering isn't in the UI — it's in what the OS lets you ask when you have the right entitlements.

## Deep dive: parsing Mach-O by hand

The Mach-O inspector is the piece I'm happiest with, so let's walk through it.

Given a path to a binary, the naive path is `dlopen` + `_dyld_image_count` — but that only works for things already loaded into your address space. To inspect *any* on-disk binary (system daemons, other apps' executables), you need to parse the file yourself.

The shape:

1. `open()` + `mmap()` the file `PROT_READ`.
2. Read the first 4 bytes. If it's `MH_MAGIC_64` / `MH_CIGAM_64`, it's a thin arm64 binary. If it's `FAT_MAGIC` / `FAT_CIGAM`, it's a FAT binary and you walk `fat_arch` entries to find the arm64(e) slice.
3. Cast the slice base to `struct mach_header_64 *`.
4. Walk `ncmds` load commands starting at `header + 1`. Each `load_command` has a `cmd` and `cmdsize` — advance by `cmdsize` bytes each iteration.

The commands you actually care about for a "what's in this binary" view:

- `LC_SEGMENT_64` → segment name (`__TEXT`, `__DATA`, `__LINKEDIT`, `__DATA_CONST`), vmaddr, vmsize, and nested sections.
- `LC_LOAD_DYLIB` / `LC_LOAD_WEAK_DYLIB` / `LC_REEXPORT_DYLIB` → the linked libraries list. The dylib path lives at `cmd + dylib.name.offset`.
- `LC_UUID` → build identity.
- `LC_CODE_SIGNATURE` → offset/size of the CMS blob (useful if you want to cross-check with `csops` output from stage 2).

Two things that bite you:

- **Endianness.** FAT headers are big-endian on disk even on little-endian devices. If magic is `FAT_CIGAM`, you `OSSwapBigToHostInt32` every field you read. Miss this and you'll seek to a garbage offset and either segfault or hand back nonsense.
- **`cmdsize` alignment.** Load commands are 8-byte aligned in 64-bit Mach-O. Trust `cmdsize` from the command itself — don't compute it from `sizeof(struct)`, because Apple adds fields between SDK versions and your struct definition can lag reality.

The whole parser is a couple hundred lines of C++ with no dependencies. The Swift layer just receives an array of segment structs and renders them.

## Why TrollStore

Most of what SysExplorer does needs entitlements that the App Store will never sign:

```xml
<key>task_for_pid-allow</key><true/>
<key>platform-application</key><true/>
<key>com.apple.private.security.no-sandbox</key><true/>
<key>com.apple.private.security.no-container</key><true/>
```

Without `task_for_pid-allow`, `proc_pidinfo` on foreign PIDs returns `EPERM`. Without `no-sandbox`, most paths in `/System/Library/LaunchDaemons/` come back `EACCES`. TrollStore is what makes these entitlements survive install without a paid developer account or a jailbreak.

If you're on iOS 15–16.6.1 (or one of the arm64e 17.0 windows), you can run this today.

## Install

1. Grab `SysExplorer.tipa` from [Releases](https://github.com/lavochkaa/SystemExplorer/releases).
2. Open it with TrollStore.
3. Launch.

Source, build script, and issues: [github.com/lavochkaa/SystemExplorer](https://github.com/lavochkaa/SystemExplorer).

## What's next

Roadmap candidates I'm weighing:

- Kernel extension list (via `kmod_info` walk, where entitlements permit).
- Sandbox profile viewer for the selected process.
- `dyld` shared cache inspector — the same Mach-O parser applies, the tricky part is the cache's own header format.

If you have a specific corner of iOS internals you'd want surfaced, [open an issue](https://github.com/lavochkaa/SystemExplorer/issues) or ping me on [Twitter](https://twitter.com/name_space1).
