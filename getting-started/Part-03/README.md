This is part of a [series of articles](https://github.com/PLCnext/rust-sample-runtime) that demonstrate how to create a runtime application for PLCnext Control. Each article builds on tasks that were completed in earlier articles, so it is recommended to follow the series in sequence.

## Part 3 - Reading and writing Axioline I/O

A requirement of most PLCnext runtime applications is that they must exchange process data with input and output modules on the local Axioline bus.

The Axioline bus is controlled by a PLCnext Control system component, and this Axioline component must be configured using Technology Independent Configuration (.tic) files.

In this article, we will use [PLCnext Engineer](http://phoenixcontact.net/product/1046008) to generate a set of TIC files for a specific arrangement of Axioline I/O modules. Note that it is possible to configure the Axioline bus without using PLCnext Engineer - a description of how to do this is beyond the scope of this series, but will be covered in a future technical article in the PLCnext Community.

1. In PLCnext Engineer, [create a new project](https://youtu.be/I-FeT3p6cGA) that includes:
   - A PLCnext Control with the correct firmware version.
   - Axioline I/O modules corresponding to your physical hardware arrangement.

   Make sure that there are no Programs or Tasks scheduled to run on either ESM1 or ESM2, and no connections in the PLCnext Port List.

1. Download the PLCnext Engineer project to the PLC.

   This creates a set of .tic files on the PLC that results in the automatic configuration of the Axioline bus when the plcnext process starts.

1. Determine I/O port names from the .tic files created by PLCnext Engineer.

   Examine the .tic file(s) in the PLC directory `/opt/plcnext/projects/PCWE/Io/Arp.Io.AxlC`. The files of interest have long, cryptic file names (e.g. `1da1f65d-b76f-4364-bfc4-f59474ccfdad.tic`). In these files, look for elements labelled "IO:Port", and note down the "NodeId", "Name" and "DataType" of each port. These will be needed for our application.

1. Modify the file `main.rs` so it looks like the following:
   <details>
   <summary>(click to see/hide code)</summary>

   ```rust
   //
   // Copyright (c) 2019 Phoenix Contact GmbH & Co. KG. All rights reserved.
   // Licensed under the MIT. See LICENSE file in the project root for full license information.
   // SPDX-License-Identifier: MIT
   //
   use std::env;
   use std::{thread, time};
   use plcnext_commons::log;

   fn main() {
      // Set the time between scan cycles
      let pause = time::Duration::from_millis(1000);

      // Put the command line arguments into a collection
      let args: Vec<String> = env::args().collect();

      // Tell the PLCnext runtime that we are starting
      plcnext::load("/usr/lib", "runtime", &args[1]);

      // Log a message
      log("Hello, world!");

      // Wait for Axioline configuration to be completed
      // before attempting to access I/O
      thread::sleep(time::Duration::from_secs(30));

      // Declare process data items
      let mut in_value: [u8;1]  = [0x00];
      let mut out_value: [u8; 1] = [0x00];

      loop {
         // Read process inputs
         plcnext::read_input_data("Arp.Io.AxlC", "Arp.Io.AxlC/0.~DI8", &mut in_value).ok();

         // Perform application-specific processing
         // In this case, simply invert the process data bits
         out_value[0] = !in_value[0];

         // Write process outputs
         plcnext::write_output_data("Arp.Io.AxlC", "Arp.Io.AxlC/0.~DO8", &out_value).ok();

         // Wait a short time before repeating
         thread::sleep(pause);
      }
   }
   ```

   </details>

   Notes on the above code:
   - After the call to `plcnext::load`, Axioline bus configuration - which is taking place in the background - must be completed before attempting to access I/O data. In this case a timer is used, but there is a smarter way to do this - as will be shown in a later article.
   - The I/O read and write operations are performed by the functions `plcnext::read_input_data` and `plcnext::write_output_data`.
   - The required format of I/O port names is {Bus Type}/{NodeId}.{Name}, where {Bus Type} = "Arp.Io.AxlC", and {NodeId} and {Name} were obtained from the .tic file on the PLC.
   - In this example, we have hard-coded the I/O details (including the port name) in the read and write functions, but in a real application these types of functions should be general-purpose.

   An important point to note is that, in this project, I/O data is transferred automatically from the GDS to Axioline I/O modules every 500μs. This should be considered for real-time tasks with very short cycle times (in the order of milliseconds).

1. Build the project to generate the `runtime` executable.

   ```sh
   tools/build-axcf2152.sh -t "/opt/pxc/sdk/AXCF2152/2019.9"
   ```

1. Copy the executable to the PLC.

   ```sh
   scp target/armv7-unknown-linux-gnueabihf/debug/runtime admin@192.168.1.10:~/projects/runtime
   ```

   Note: If you receive a "Text file busy" message in response to this command, then the file is probably locked by the PLCnext Control. In this case, stop the plcnext process on the PLC with the command `sudo /etc/init.d/plcnext stop` before copying the file.

   It is assumed that the ACF config and settings files (described in a previous article) are already on the PLC.

1. Open a secure shell session on the PLC:

   ```sh
   ssh admin@192.168.1.10
   ```

1. Restart the plcnext process:

   ```sh
   sudo /etc/init.d/plcnext restart
   ```

   After approximately 30 seconds, you should see the digital outputs set to the inverse of the digital input values.

---

Copyright © 2019 Phoenix Contact Electronics GmbH

All rights reserved. This program and the accompanying materials are made available under the terms of the [MIT License](http://opensource.org/licenses/MIT) which accompanies this distribution.
