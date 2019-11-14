This is part of a [series of articles](https://github.com/PLCnext/rust-sample-runtime) that demonstrate how to create a runtime application for PLCnext Control. Each article builds on tasks that were completed in earlier articles, so it is recommended to follow the series in sequence.

## Part 4 - Getting PLCnext Control status via callbacks

It is possible to get notifications of PLCnext Control state changes via a callback function. In this article, we will use these notifications to start and stop process data exchange with the Axioline I/O modules.

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
   use plcnext::PlcOperation;
   use plcnext_commons::log;

   fn main() {
      // Set the time between scan cycles
      let pause = time::Duration::from_millis(1000);

      // Put the command line arguments into a collection
      let args: Vec<String> = env::args().collect();

      // Functions to start and stop runtime processing
      static mut PROCESSING: bool = false;
      fn start_processing() {
         log("Start processing");
         unsafe {PROCESSING = true};
      };
      fn stop_processing() {
         log("Stop processing");
         unsafe {PROCESSING = false};
      };

      // Closure that handles PLC operations
      let plc_operation_handler = |operation: PlcOperation| {
         match operation {
               PlcOperation::Load => {
                  log("Call of PLC Load");
               },
               PlcOperation::Setup => {
                  log("Call of PLC Setup");
               },
               PlcOperation::StartCold => {
                  log("Call of PLC Start Cold");
                  start_processing();
               },
               PlcOperation::StartWarm => {
                  log("Call of PLC Start Warm");
                  // When this state-change occurs, the firmware is ready to serve requests.
                  // Now we can request the needed service interfaces.
                  start_processing();
               },
               PlcOperation::StartHot => {
                  log("Call of PLC Start Hot");
                  start_processing();
               },
               PlcOperation::Stop => {
                  log("Call of PLC Stop");
                  stop_processing();
               },
               PlcOperation::Reset => {
                  log("Call of PLC Reset");
                  stop_processing();
               },
               PlcOperation::Unload => {
                  log("Call of PLC Unload");
                  stop_processing();
               },
               PlcOperation::None => {
                  log("Call of PLC None");
               },
               PlcOperation::Unknown => {
                  log("Call of PLC Unknown");
               }
         }
      };

      // Set the PLC operation handler
      plcnext::set_handler(Some(plc_operation_handler));

      // Tell the PLCnext runtime that we are starting
      plcnext::load("/usr/lib", "runtime", &args[1]);

      // Declare process data items
      let mut in_value: [u8;1]  = [0x00];
      let mut out_value: [u8; 1] = [0x00];

      let mut loop_counter: u64 = 0;

      loop {
         if unsafe {PROCESSING} {
               loop_counter = loop_counter+1;

               // Log the hardware ID of the PLC on the first loop
               if loop_counter == 1 {
                  let hardware_id = plcnext::get_unique_hardware_id();
                  log(&format!("Unique hardware ID: {:02x?}", hardware_id));
               }

               // Read process inputs
               plcnext::read_input_data("Arp.Io.AxlC", "Arp.Io.AxlC/0.~DI8", &mut in_value).ok();

               // Perform application-specific processing
               // In this case, simply invert the process data bits
               out_value[0] = !in_value[0];

               // Write process outputs
               plcnext::write_output_data("Arp.Io.AxlC", "Arp.Io.AxlC/0.~DO8", &out_value).ok();
         } 

         // Wait a short time before repeating
         thread::sleep(pause);
      }
   }
   ```

   </details>

   Notes on the above code:
   - The startup delay timer from the earlier example has been removed.
   - The boolean variable `PROCESSING` has been added. This is used to enable and disable I/O processing in the `main` function. Because this flag is static and mutable, it can only be accessed using the `unsafe` keyword.
   - The closure `plc_operation_handler` has been defined, which sets and resets the `PROCESSING` flag based on the state of the PLC.
   - The closure is registered with the PLCnext Control (via `plcnext::set_handler`) **before** the `plcnext::load` function is called. This ensures that all PLC state changes will be captured during startup.
   - On the first cycle of the endless loop, the unique hardware ID of the PLC is retrieved and logged.

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

1. Check the log file to see messages for each PLC state change, and a message containing the PLC's unique 32-byte hardware ID.

   ```sh
   cat /opt/plcnext/projects/runtime/logs/runtime.log
   ```

---

Copyright © 2019 Phoenix Contact Electronics GmbH

All rights reserved. This program and the accompanying materials are made available under the terms of the [MIT License](http://opensource.org/licenses/MIT) which accompanies this distribution.
