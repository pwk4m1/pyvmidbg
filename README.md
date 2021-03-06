# pyvmidbg

[![Join the chat at https://gitter.im/pyvmidbg/Lobby](https://badges.gitter.im/trailofbits/algo.svg)](https://gitter.im/pyvmidbg/Lobby)
[![standard-readme compliant](https://img.shields.io/badge/readme%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/RichardLitt/standard-readme)

> LibVMI-based GDB server, implemented in Python

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Install](#install)
- [Usage](#usage)
    - [Server](#server)
    - [GDB Frontends](#gdb-frontends)
 - [References](#references)
 - [Maintainers](#maintainers)
 - [Contributing](#contributing)
 - [License](#license)

## Overview

This GDB stub allows you to debug a remote process running in a VM with
your favorite GDB frontend.

By leveraging *virtual machine introspection*, the stub remains **stealth** and requires 
**no modification** of the guest.

### Why debugging from the hypervisor ?

Operating systems debug API's are problematic:

1. they have never been designed to deal with malwares, and lack the stealth and robustness required when 
analyzing malicious code
2. they have an observer effect, by implicitly modifying the process environment being debugged
3. this observer effect might be intentional to protect OS features (`Windows PatchGuard`/`Protected Media Path` are disabled)
4. modern OS have a high degree of kernel security mechanisms that narrows the debugger's view of the system
 (`Windows 10 Virtual Secure Mode`)
5. debugging low-level processes and kernel functions interacting directly with the transport protocol used by the debug agent can
    turn into a infinite recursion hell (eg. debugging TCP connections and having a kernel debug stub communicating via TCP)
5. in special cases the "Operating System" lacks debugging capabilities (`unikernels`)

Existing solutions like GDB stubs included in `QEMU`, `VMware` or `VirtualBox` can only
pause the VM and debug the kernel, but lack the guest knowledge to track and follow the rest of the processes.

### Vision

![vmidbg](https://user-images.githubusercontent.com/964610/53520440-f71f9d00-3ad5-11e9-9f5d-216fc7a2bae4.png)

## Features

- intercept process at `CR3` load
- read/write memory
- get/set registers
- continue execution
- singlestep
- breakin (`CTRL-C`)
- insert/remove software breakpoint

## Requirements

- `Python 3`
- `python3-docopt`
- `python3-lxml`
- [`python3-libvmi`](https://github.com/libvmi/python)
- `Xen`

## Install

~~~
virtualenv -p python3 venv
source venv/bin/activate
pip install .
~~~

Note: If you don't want to install `Xen`, [vagrant-xen-pyvmidbg](https://github.com/Wenzel/vagrant-xen-pyvmidbg)
provides a Vagrant environment based on `KVM`, with ready to use `Windows` and `Linux` VMs.

## Usage

### Server

~~~
vmidbg <port> <vm> [<process>]
~~~

example:
~~~
(venv) $ sudo vmidbg 5000 winxp explorer.exe
INFO:root:kernel base address: 0x804d7000
INFO:DebugContext:attaching on explorer.exe
INFO:DebugContext:intercepted System
INFO:DebugContext:intercepted csrss.exe
INFO:DebugContext:intercepted explorer.exe
INFO:server:listening on 127.0.0.1:5000
...
~~~

### GDB Frontends

#### GDB

~~~
$ gdb
(gdb) target remote 127.0.0.1:5000
warning: Remote gdbserver does not support determining executable automatically.
RHEL <=6.8 and <=7.2 versions of gdbserver do not support such automatic executable detection.
The following versions of gdbserver support it:
- Upstream version of gdbserver (unsupported) 7.10 or later
- Red Hat Developer Toolset (DTS) version of gdbserver from DTS 4.0 or later (only on x86_64)
- RHEL-7.3 versions of gdbserver (on any architecture)
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x80545c9f in ?? ()
(gdb) x/10i $eip-3
   0x80545c9c:  mov    %eax,%cr3
=> 0x80545c9f:  mov    %cx,0x66(%ebp)
   0x80545ca3:  jmp    0x80545cb3
   0x80545ca5:  lea    0x0(%ecx),%ecx
   0x80545ca8:  lea    0x540(%ebx),%ecx
   0x80545cae:  call   0x80540c6c
   0x80545cb3:  mov    0x18(%ebx),%eax
   0x80545cb6:  mov    0x3c(%ebx),%ecx
   0x80545cb9:  mov    %ax,0x3a(%ecx)
   0x80545cbd:  shr    $0x10,%eax
~~~

#### radare2

~~~
$ r2 -b 32 -d gdb://127.0.0.1:5000
[0x80545c9f]> pd 10 @eip-3
            0x80545c9c      0f22d8         mov cr3, eax
            ;-- eip:
            0x80545c9f      66894d66       mov word [ebp + 0x66], cx   ; [0x66:2]=0xffff ; 'f' ; 102
        ,=< 0x80545ca3      eb0e           jmp 0x80545cb3
        |   0x80545ca5      8d4900         lea ecx, [ecx]
        |   0x80545ca8      8d8b40050000   lea ecx, [ebx + 0x540]      ; 1344
        |   0x80545cae      e8b9afffff     call 0x80540c6c
        `-> 0x80545cb3      8b4318         mov eax, dword [ebx + 0x18] ; [0x18:4]=-1 ; 24
            0x80545cb6      8b4b3c         mov ecx, dword [ebx + 0x3c] ; [0x3c:4]=-1 ; '<' ; 60
            0x80545cb9      6689413a       mov word [ecx + 0x3a], ax
            0x80545cbd      c1e810         shr eax, 0x10
~~~

## References

- [vmidbg](https://github.com/Zentific/vmidbg): original idea and C implementation
- [plutonium-dbg](https://github.com/plutonium-dbg/plutonium-dbg): [GDB server protocol parsing](https://github.com/plutonium-dbg/plutonium-dbg/blob/master/clients/gdbserver.py)
- [ollydbg2-python](https://github.com/0vercl0k/ollydbg2-python): [GDB server protocol parsing](https://github.com/0vercl0k/ollydbg2-python/blob/master/samples/gdbserver/gdbserver.py)
- [GDB RSP protocol specifications](https://sourceware.org/gdb/onlinedocs/gdb/Remote-Protocol.html)

## Maintainers

[@Wenzel](https://github.com/Wenzel)

## Contributing

PRs accepted.

Small note: If editing the Readme, please conform to the [standard-readme](https://github.com/RichardLitt/standard-readme) specification.

## License

[GNU General Public License v3.0](https://github.com/Wenzel/pyvmidbg/blob/master/LICENSE)