This is part of a [series of articles](https://github.com/PLCnext/rust-sample-runtime) that demonstrate how to create a runtime application for PLCnext Control. Each article builds on tasks that were completed in earlier articles, so it is recommended to follow the series in sequence.

## Part 1 - Hello PLCnext

In this article, we will set up Rust on the host machine, create a simple application, and run it on a PLCnext Control.

This article draws heavily on the procedures described in the [japaric/rust-cross](https://github.com/japaric/rust-cross) project and in the article ["Embedded development with Yocto and Rust"](https://pagefault.blog/2018/07/04/embedded-development-with-yocto-and-rust/), both of which include detailed technical information on some of the steps in this procedure.

1. Install the PLCnext CLI tool (`plcncli`), and the Software Development Kit (SDK) corresponding to the firmware version on your PLCnext Control.

1. Install `rustup` using [the procedure on the Rust website](https://www.rust-lang.org/learn/get-started).

1. Because the AXC F 2152 is an ARMv7 device, we can use standard crates that were cross-compiled with the `armv7-unknown-linux-gnueabihf` toolchain.

   ```sh
   rustup target add armv7-unknown-linux-gnueabihf
   ```

   Note that a different set of standard crates must be used for PLCnext Control platforms based on a different architecture (e.g. RFC 4072S).

1. Configure cargo for cross compilation. If it does not exist already, create a file named `config` in the directory `~/.cargo`. Add the following text to the end of the `config` file:

   ```
   [target.armv7-unknown-linux-gnueabihf]
   linker = "arm-pxc-linux-gnueabi-gcc"
   ```

   In this case, we are using the linker that comes with the PLCnext Control SDK.

1. Use cargo to create and build a new executable project. We will call our project `runtime`, but you can call it whatever you want.

   ```sh
   cargo new --bin runtime
   ```

1. Set up the build environment.

   ```sh
   source /path/to/your/sdk/environment-setup-cortexa9t2hf-neon-pxc-linux-gnueabi
   export RUSTFLAGS="-C link-arg=-march=armv7-a -C link-arg=-mthumb -C link-arg=-mfpu=neon -C link-arg=-mfloat-abi=hard -C link-arg=-mcpu=cortex-a9 -C link-arg=--sysroot=${SDKTARGETSYSROOT}"
   ```

1. Build the project.

   ```sh
   cd runtime
   cargo build --target=armv7-unknown-linux-gnueabihf
   ```

1. Check that the binary has been built for the correct architecture.

   ```sh
   file target/armv7-unknown-linux-gnueabihf/debug/runtime
   ```

   You should see something like:

   ```sh
   runtime: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=d543f7122e7280846dd1d89ebd9484c2c1a23c24, not stripped
   ```

1. Deploy the executable to the PLC.

   In a terminal window on the host, from the project's root directory (substituting the IP address of your PLC):

   ```sh
   ssh admin@192.168.1.10 'mkdir -p projects/runtime'
   scp target/armv7-unknown-linux-gnueabihf/debug/runtime admin@192.168.1.10:~/projects/runtime
   ```

1. Open a secure shell session on the PLC:

   ```sh
   ssh admin@192.168.1.10
   ```

1. Run the program:

   ```sh
   projects/runtime/runtime
   ```

   You should see the output in the terminal:

   ```sh
   Hello, world!
   ```

---

Copyright © 2019 Phoenix Contact Electronics GmbH

All rights reserved. This program and the accompanying materials are made available under the terms of the [MIT License](http://opensource.org/licenses/MIT) which accompanies this distribution.