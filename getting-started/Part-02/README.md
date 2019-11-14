This is part of a [series of articles](https://github.com/PLCnext/rust-sample-runtime) that demonstrate how to create a runtime application for PLCnext Control. Each article builds on tasks that were completed in earlier articles, so it is recommended to follow the series in sequence.

## Part 2 - PLCnext Control integration

In the previous article, we built a simple "Hello PLCnext" application and started it manually on a PLCnext Control.

In order for a runtime application to access PLCnext Control services like Axioline I/O, a specific set of ARP components must be running. Most of these components are started automatically by the ARP, but two additional components - required for ANSI-C access - must be started in our application process.

Using an ACF configuration file, we will instruct the PLCnext to automatically start and stop our runtime application with the ARP, and to load the required application-specific PLCnext components.

Then, using an ACF settings file, we will specify a directory where we want application log files to be created and stored.

1. On the host machine, install `llvm-dev`:

   ```sh
    sudo apt install llvm-dev
   ```

   This package contains `llvm-config`, which is required for part of the build process.

1. From the project root directory, create sub-directories:

   ```sh
   mkdir data tools
   ```

1. In the project's `data` directory, create a file named `runtime.acf.config`, containing the following text:
   <details>
   <summary>(click to see/hide code)</summary>

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <AcfConfigurationDocument
   xmlns="http://www.phoenixcontact.com/schema/acfconfig"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.phoenixcontact.com/schema/acfconfig.xsd"
   schemaVersion="1.0" >

   <Processes>
      <Process name="runtime"
               binaryPath="$ARP_PROJECTS_DIR$/runtime/runtime"
               workingDirectory="$ARP_PROJECTS_DIR$/runtime"
               args="runtime.acf.settings"/>
   </Processes>

   <Libraries>
      <Library name="Arp.Plc.AnsiC.Library"
               binaryPath="$ARP_BINARY_DIR$/libArp.Plc.AnsiC.so" />
   </Libraries>

   <Components>

      <Component name="Arp.Plc.AnsiC"
                 type="Arp::Plc::AnsiC::AnsiCComponent"
                 library="Arp.Plc.AnsiC.Library"
                 process="runtime">
         <Settings path="" />
      </Component>

      <Component name="Arp.Plc.DomainProxy.IoAnsiCAdaption"
                 type="Arp::Plc::Domain::PlcDomainProxyComponent"
                 library="Arp.Plc.Domain.Library"
                 process="runtime">
         <Settings path="" />
      </Component>

   </Components>

   </AcfConfigurationDocument>
   ```

   </details>

   In this configuration file, the `Processes` element tells the Application Component Framework (ACF) to start and stop our `runtime` application with the plcnext process. The value of the `args` element is passed as a command-line argument to our application. The `Libraries` and `Components` elements tell the ACF to start special PLCnext Control components that our application will use to (for example) access Axioline I/O through an ANSI-C interface.

1. In the project's `data` directory, create a file named `runtime.acf.settings`, containing the following text:
   <details>
   <summary>(click to see/hide code)</summary>

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <AcfSettingsDocument
   xmlns="http://www.phoenixcontact.com/schema/acfsettings"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.phoenixcontact.com/schema/acfsettings.xsd"
   schemaVersion="1.0" >
   
   <RscSettings path="/etc/plcnext/device/System/Rsc/Rsc.settings"/>
   
   <LogSettings logLevel="Debug" logDir="logs" />

   <EnvironmentVariables>
      <EnvironmentVariable name="ARP_BINARY_DIR" value="/usr/lib" /> <!-- Directory of PLCnext binaries -->
   </EnvironmentVariables>

   </AcfSettingsDocument>
   ```

   </details>

   This file gives the ACF additional information in order to run our application.

   The value of the `logDir` attribute in the `LogSettings` element is the path where log files for this application will be created, relative to the application's working directory.

1. Add the two crates [plcnext](https://crates.io/crates/plcnext) and [plcnext-commons](https://crates.io/crates/plcnext-commons) to the project by adding the following to the project's Cargo.toml file:

   ```toml
   [dependencies]
   plcnext = "0.2.0"
   plcnext-commons = "0.1.0"
   ```

1. In the `main.rs` source file:
   - The application must call the function `plcnext::load`, which notifies the ACF that the application has started successfully. Also, through this call, the path to an `.acf.settings` file is provided. If this function is not called, the application will not be able to access any PLCnext services, and it will be stopped after a short timeout period.

   - The application must never end. The PLCnext Control expects applications like this to keep running until the PLCnext process shuts it down.

   The modified source file should look like this:
   <details>
   <summary>(click to see/hide code)</summary>

   ```rust
   //
   // Copyright (c) 2019 Phoenix Contact GmbH & Co. KG. All rights reserved.
   // Licensed under the MIT. See LICENSE file in the project root for full license information.
   // SPDX-License-Identifier: MIT
   //
   use std::{thread, time};
   use plcnext_commons::log;

   fn main() {
      // Set the time between scan cycles
      let pause = time::Duration::from_millis(1000);

      // Put the command line arguments into a collection
      let args: Vec<String> = env::args().collect();

      // Tell the PLCnext runtime that we are starting
      plcnext::load("/usr/lib", "runtime", args[1]);

      // Log a message
      log("Hello, world!");

      // Loop forever. The PLCnext runtime will kill this process as it shuts down.
      loop {
         thread::sleep(pause);
      }
   }
   ```

   </details>

   Notes on the above code:
   - The call to `println` in the earlier example has been replaced with a call to the PLCnext Control function `plcnext_commons::log`. This function writes to a file in the directory we specified in the `.acf.settings` file.

   - We have included an infinite loop that simply sleeps for 1 second while waiting for the PLCnext Control to kill the runtime process.

1. Create a build script. In the project's `tools` directory, create a file named `build-axcf2152.sh`, containing the following text:

   <details>
   <summary>(click to see/hide code)</summary>

   ```sh
   #!/bin/bash
   #//
   #// Copyright (c) 2019 Phoenix Contact GmbH & Co. KG. All rights reserved.
   #// Licensed under the MIT. See LICENSE file in the project root for full license information.
   #// SPDX-License-Identifier: MIT
   #//
   while getopts t: option
   do
   case "${option}"
   in
   t) TOOLCHAIN=${OPTARG};;
   esac
   done

   # Get the directory of this script
   DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"

   # Set PLCnext environment variables
   source ${TOOLCHAIN}/environment-setup-cortexa9t2hf-neon-pxc-linux-gnueabi

   # Set RUSTFLAGS
   export RUSTFLAGS="-C link-arg=-march=armv7-a -C link-arg=-mthumb -C link-arg=-mfpu=neon -C link-arg=-mfloat-abi=hard -C link-arg=-mcpu=cortex-a9 -C link-arg=--sysroot=${SDKTARGETSYSROOT}"

   # BINDGEN_EXTRA_CLANG_ARGS is required for the "plcnext-sys" crate to successfully generate bindings for the PLCnext ANSI-C functions.
   export BINDGEN_EXTRA_CLANG_ARGS="-target arm-pxc-linux-gnueabi --sysroot=${TOOLCHAIN} -I${SDKTARGETSYSROOT}/usr/include/plcnext"

   # CC_armv7_unknown_linux_gnueabihf is required to build the "backtrace-sys" dependency.
   unset CC
   CC_armv7_unknown_linux_gnueabihf="arm-pxc-linux-gnueabi-gcc -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -mcpu=cortex-a9"

   # PLCNEXT_HEADERS is used to build bindings to PLCnext C++ libraries.
   export PLCNEXT_HEADERS="${SDKTARGETSYSROOT}/usr/include/plcnext"

   # Build the project
   (cd "${DIR}/.." && cargo build --target=armv7-unknown-linux-gnueabihf)
   ```

   </details>

1. Build the project to generate the `runtime` executable.

   ```sh
   tools/build-axcf2152.sh -t "/opt/pxc/sdk/AXCF2152/2019.9"
   ```

1. Deploy the executable to the PLC.

   ```sh
   scp target/armv7-unknown-linux-gnueabihf/debug/runtime admin@192.168.1.10:~/projects/runtime
   ```

   Note: If you receive a "Text file busy" message in response to this command, then the file is probably locked by the PLCnext Control. In this case, stop the plcnext process on the PLC with the command `sudo /etc/init.d/plcnext stop` before copying the file.

1. Deploy the `runtime.acf.settings` file to your project directory on the PLC.

   ```sh
   scp data/runtime.acf.settings admin@192.168.1.10:~/projects/runtime
   ```

   The destination directory is the one we specified in the call to `plcnext::load`.

1. Deploy the `runtime.acf.config` file to the PLC's `Default` project directory.

   ```sh
   scp data/runtime.acf.config admin@192.168.1.10:~/projects/Default
   ```

   All `.acf.config` files in this directory are processed by the ACF when the plcnext process starts up, via the `Default.acf.config` file in the same directory.

1. Open a secure shell session on the PLC:

   ```sh
   ssh admin@192.168.1.10
   ```

1. Create the `logs` directory for our application:

   ```sh
   mkdir /opt/plcnext/projects/runtime/logs
   ```

1. Restart the plcnext process:

   ```sh
   sudo /etc/init.d/plcnext restart
   ```

1. Give the plcnext process a short time to start our application, and then check that our application has started successfully by examining the plcnext log file:

   ```sh
   cat /opt/plcnext/logs/Output.log | grep runtime
   ```

   The result should be something like:

   ```sh
   15.07.19 11:13:04.306 Arp.System.Acf.Internal.Sm.ProcessesController               INFO  - Process 'runtime' started successfully.
   15.07.19 11:13:06.498 Arp.System.Acf.Internal.Sm.ProcessesController               INFO  - Library 'Arp.Plc.AnsiC.Library' in process 'runtime' loaded.
   15.07.19 11:13:06.506 Arp.System.Acf.Internal.Sm.ProcessesController               INFO  - Library 'Arp.Plc.Domain.Library' in process 'runtime' loaded.
   15.07.19 11:13:06.702 Arp.System.Acf.Internal.Sm.ComponentsController              INFO  - Component 'Arp.System.UmRscAuthorizator.runtime' in process 'MainProcess' created.
   15.07.19 11:13:06.707 Arp.System.Acf.Internal.Sm.ComponentsController              INFO  - Component 'Arp.Plc.AnsiC' in process 'runtime' created.
   15.07.19 11:13:06.712 Arp.System.Acf.Internal.Sm.ComponentsController              INFO  - Component 'Arp.Plc.DomainProxy.IoAnsiCAdaption' in process 'runtime' created.
   ```

1. Check the contents of the application log file:

   ```sh
   cat /opt/plcnext/projects/runtime/logs/runtime.log
   ```

   You should see a number of formatted output messages, including a message similar to the following:

   ```sh
   16.07.19 04:19:23.355 root     INFO    - Hello, world!
   ```

---

Copyright © 2019 Phoenix Contact Electronics GmbH

All rights reserved. This program and the accompanying materials are made available under the terms of the [MIT License](http://opensource.org/licenses/MIT) which accompanies this distribution.