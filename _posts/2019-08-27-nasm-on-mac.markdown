---
layout: post
title:  "Playing with NASM on MacOS"
date:   2019-08-27 21:54:42 +0200
categories: asm nasm macos gdb
---

Once I was wondering, how different the 'hello world' in assembler on Linux would be from one on MacOS.

It appears that not by much. Just some extra tricks required to make things work as expected: taking into account different `write` and `exit` syscall numbers, as well as adding extra parameters (`-macosx_version_min` and `-lSystem`) to the linker.

The code is pretty straightforward, as rediscovering 'hello world' is hard, thus just taking it from [here][so-asm]:

```asm
SECTION .text
global _main

_main:
mov rax, 0x2000004      ; syscall 4: write (
mov rdi, 1              ;    fd,
mov rsi, Msg            ;    buffer,
mov rdx, Len            ;    size
syscall                 ; )

mov rax, 0x2000001      ; syscall 1: exit (
mov rdi, 0              ;    retcode
syscall                 ; )

SECTION .data
Msg db `Hello, world!\n`
Len: equ $-Msg
```

Next it needs to be compiled (by `nasm` in this case, can be installed with `brew install nasm`) and linked:

```
$ nasm -fmacho64 -o hello.o hello.asm
$ ld -o hello hello.o -macosx_version_min 10.13 -lSystem
$ ./hello
Hello, world
```

More details on NASM in MacOS [here][nasm].

Setting up `gdb` requires some more tricks with code-signing. Those tricks are described [here][so1] and [here][so2]. Short dump of commands for reference:

```
# Keychain Access -> Certificate Assistant -> Create Certificate
# gdb-cert, Self Signed Root, Code Signing, Let me override defaults
# Specify a Location for The Certificate: Keychain = System
# On the certificate: Trust section, Code Signing = Always Trust

$ sudo killall taskgated
$ codesign -fs gdb-cert "$(which gdb)"
$ echo "set startup-with-shell off" > ~/.gdbinit

$ codesign --entitlements gdb.xml -fs gdb-cert /usr/local/bin/gdb
$ cat gdb.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.cs.allow-dyld-environment-variables</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
    <key>com.apple.security.cs.disable-executable-page-protection</key>
    <true/>
    <key>com.apple.security.cs.debugger</key>
    <true/>
    <key>com.apple.security.get-task-allow</key>
    <true/>
</dict>
</plist>
```

After that `gdb` can be run without `sudo`:

```
$ gdb hello
GNU gdb (GDB) 8.3
...
Reading symbols from hello...
(No debugging symbols found in hello)
(gdb) b main
Breakpoint 1 at 0x1fd9
(gdb) r
Starting program: /path/to/hello
...
Thread 2 hit Breakpoint 1, 0x0000000000001fd9 in main ()
(gdb) s
Single stepping until exit from function main,
which has no line number information.
Hello, world!
[Inferior 1 (process 7485) exited normally]
```



[so-asm]: https://stackoverflow.com/a/21130076
[nasm]: http://sevanspowell.net/posts/learning-nasm-on-macos.html
[so1]: https://stackoverflow.com/a/32727069
[so2]: https://stackoverflow.com/a/57398040

