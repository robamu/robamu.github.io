---
title: "Cross-Compile Rust for the Raspberry Pi"
#description: <descriptive text here>
date: 2021-12-27T12:31:51+01:00
draft: true
toc: false
image: ""
tags: ["rust", "cross-compile", "vscode", "cli", "raspberrypi"]
categories: ["rust", "cross-compile", "coding"]
---

After exploring Rust for smaller bare-metal systems like Cortex-M based microcontrollers, I am
trying to learn using Rust when using a Linux runtime. The most common example for this is the
Raspberry Pi, but there are a lot of other boards out there which support Embedded Linux, for
example the Beagle Bone Black or Xilinx hybrid CPU / FPGA solutions like the Zynq 7020.

## Some Background Information

If you're only interested in the result and on how to quickly cross-develop applications for your
Linux board, go to the next section.

I am especially interested in potential of Rust to develop more complex applications and allow
remote development for Linux boards. All of this generally requires cross-compiling. For most
use-cases and simpler projects, compiling and running the applications on the Linux boards directly
is a lot simpler then the effort of setting up a cross-compiling environment on a host machine.

However, my experiences when developing something like satellite software
is the following:

1. Even though there is a Linux runtime, it might not support compiling
   applications on the board directly. For example, the
   [Q7S](https://xiphos.com/products/q7-processor/) used in one of our
   projects has a root file system limit of 32 MB, so it does not even
   contain something like header files.
2. There is generally one monolithic software written in C or C++ and which
   contains all the application logic. It contains most of the dependencies,
   for example on libraries for a certain device like a Star Tracker. This means
   the compilation times are long. Even if the Linux boards support
   compilation, compiling the primary software on the board becomes unfeasable.
3. There are only 1-2 of the On-Board Computers available. For example, one is a
   Flight Model which remains packaged, the other one is the Engineering Model
   which has to stay in the clean room. Allowing convenient development
   requires remote deployment of the application. Otherwise, I'd have to go
   into the cleanroom for every small test and change I want to introduce.
4. The software is complex and debugging can become complex too. Printouts and LEDs
   are not sufficient anymore to debug the software, a full debugger is required.

For that reason, convenient cross-compilation and debugging is a must for me when considering
Rust as an alternative to C/C++ on systems like the [Q7S](https://xiphos.com/products/q7-processor/).
I have only found bits and pieces in the Internet on how to properly do this. Therefore, I have
created a template repository which gathers all those bits and pieces into one package.
I specifically targetted debugging with the command line and with VS Code as those tools are most
commonly used in Rust development. The instrutions provided here have been tested on
Linux (Ubuntu 21.04) and Windows 10, but I really recommend to use a Linux development hosted when
developing anything for an Embedded Linux board.

## Cross-Building a Rust application for the Raspberry Pi

The instructions here are based on
[this excellent guide](https://chacin.dev/blog/cross-compiling-rust-for-the-raspberry-pi/).

Clone the template repository first:

```sh
git clone https://github.com/robamu-org/rpi-rs-crosscompile.git
```

You can also use the template functionality of GitHub to create your own custom repository.
Cross-Building an application for the Raspberry Pi still requires a C/C++ cross-toolchain to compile
Rust with an ARM linker.
Make sure to download a suitable cross-compiler. Most of the automation
performed by the repository is done in a Python script, so it is recommended
to [install Python](https://www.python.org/downloads/) as well.

Make sure you can call it from the command line

**Linux**:

```console
[Raspberry Pi 4] rmueller@power-pinguin:~/Rust/rpi-rs-crosscompile(main)$ python3 --version
Python 3.9.7
```

**Windows**:

```console
```

### Setting up the Raspberry Pi

There are different ways to avoid having to retype the password everytime
when tranferring an application to the Raspberry Pi or running an application
remotely. On Linux, you can use `sshpass` for this.

The most easiest way and the one I recommend is to install your SSH key
on the Raspberry Pi with `ssh-copy-id`. This works on Linux and Windows.
You can follow the steps specified [here](https://www.ssh.com/academy/ssh/copy-id) to do this.

Another way is to use a SSH key file by supplying the `-f SSHFILE` flag
to the `bld-deploy-remote.py` application in your `.cargo/config.toml`
or as an environmental variable `SSHPASS` with the `-e` flag.

### Setup Linux

You can create a cross-compiler built with crosstool-ng
from [here](https://www.dropbox.com/sh/hkn4lw87zr002fh/AAAO-HxFQzfmmPQQ9KVmoooGa?dl=0). There is one
available for the Raspberry Pi 3 and the Raspberry Pi 4.

Following command should get the job done, assuming you
installed the cross-compiler shown above into the `$HOME/x-tools` folder:

```sh
export PATH=$PATH:"$HOME/x-tools/armv8-rpi4-linux-gnueabihf/bin"
```

You can also put this in a script named `rpi4-path.sh` and source it
with `. rpi4-path.sh` or your can put it in your `.bashrc` file to add it
to `$PATH` permanently.

Test with `armv8-rpi4-linux-gnueabihf-gcc --version`:

```console
[Raspberry Pi 4] rmueller@power-pinguin:~/Rust$ armv8-rpi4-linux-gnueabihf-gcc --version
armv8-rpi4-linux-gnueabihf-gcc (crosstool-NG 1.24.0.390_62e9db2) 8.5.0
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

Now you are ready to build the first application for the Raspberry Pi.
The `.cargo/def-config.toml` file is a template for a Cargo configuration
to perform flashing conveniently. Copy it to `.cargo/config.toml` first:

```sh
cd .cargo
cp def-config.toml config.toml
```

Then open the `config.toml` with a text editor of your choice and select
the correct linker first, for example with `armv8-rpi4-linux-gnueabihf-gcc`:

```toml
# If you use a different cross-compiler, adapt this flag accordingly.
linker = "armv8-rpi4-linux-gnueabihf-gcc"
# linker = "arm-linux-gnueabihf-gcc"
# linker = "arm-none-linux-gnueabihf-gcc"
```

Then select the correct runner to run the application instead of debugging it

```toml
# Requires Python3 installation. Takes care of transferring and running the application
# to the Raspberry Pi
runner = "py bld-deploy-remote.py -t -r --source"
# runner = "python3 bld-deploy-remote.py -t -r --source"
...
# runner = "py bld-deploy-remote.py -t -d --source"
# runner = "python3 bld-deploy-remote.py -t -d --source"
...
# runner = "py bld-deploy-remote.py -t -d -s --gdb arm-linux-gnueabihf-gdb --source"
# runner = "python3 bld-deploy-remote.py -t -d --source"
```

Finally select the correct builder depending on whether you have a Raspberry Pi 0/1
or a Raspberry Pi 2/3/4

### Setup Windows

It is recommended to install the cross-toolchain provided by
[SysProgs](https://gnutoolchains.com/raspberry/).

Add the binary path of your installed cross-toolchain for your path.

You can add the toolchain binary path to your system environmental variables
permanently, for example like shown in the following picture:

Image here

If you use `git bash`, you can also use the Linux way shown above.

blablabla

### Running the application

Everything should be ready to run the application remotely now.
Running the application is very simple now: Use `cargo run`

**Linux**:

```console
[Raspberry Pi 4] rmueller@power-pinguin:~/Rust/rpi-rs-crosscompile(main)$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `python3 bld-deploy-remote.py -t -r --source target/armv7-unknown-linux-gnueabihf/debug/rpi-rs-crosscompile`
Running transfer command: sshpass  scp target/armv7-unknown-linux-gnueabihf/debug/rpi-rs-crosscompile pi@raspberrypi.local:"/tmp/rpi-rs-crosscompile"
Running target application: sshpass  ssh pi@raspberrypi.local /tmp/rpi-rs-crosscompile
 __________________________
< Hello fellow Rustaceans! >
 --------------------------
        \
         \
            _~^~^~_
        \) /  o o  \ (/
          '_   -   _'
          / '-----' \

```

**Windows**:


## Debugging the application on the Command Line

