# PLCnext Technology  - Rust Sample Runtime

[![Feature Requests](https://img.shields.io/github/issues/PLCnext/rust-sample-runtime/feature-request.svg)](https://github.com/PLCnext/rust-sample-runtime/issues?q=is%3Aopen+is%3Aissue+label%3Afeature-request+sort%3Areactions-%2B1-desc)
[![Bugs](https://img.shields.io/github/issues/PLCnext/rust-sample-runtime/bug.svg)](https://github.com/PLCnext/rust-sample-runtime/issues?utf8=✓&q=is%3Aissue+is%3Aopen+label%3Abug)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Web](https://img.shields.io/badge/PLCnext-Website-blue.svg)](https://www.phoenixcontact.com/plcnext)
[![Community](https://img.shields.io/badge/PLCnext-Community-blue.svg)](https://www.plcnext-community.net)

## Guide details

|Description   | Value      |
|--------------|------------|
|Created       | 14.11.2019 |
|Last modified | 14.11.2019 |
|Controller    | AXC F 2152 |
|Firmware      | 2019.9.0   |
|SDK           | 2019.9.0   |

Hardware requirements

Starter kit- AXC F 2152 STARTERKIT - (Order number 1046568), consisting of:

1. I/O module AXL F DI8/1 DO8/1 1H (Order number 2701916).
1. I/O module AXL F AI2 AO2 1H (Order number 2702072).
1. Interface module UM 45-IB-DI/SIM8 (Order number 2962997).
1. Patch cable FL CAT5 PATCH 2,0 (Order number 2832289).

Host machine software requirements:

1. Software Development Kit (SDK) for at least one PLCnext Control target.

For steps involving the use of PLCnext Engineer:

1. Windows 10 operating system.
1. PLCnext Engineer version 2019.6.

## Introduction

This article provides a step by step guide to building your own runtime application for PLCnext Control using [Rust](https://www.rust-lang.org/).

For a general introduction to runtime applications on PLCnext Control, please refer to the [README file for the C++ Sample Runtime project](https://github.com/PLCnext/SampleRuntime/blob/master/README.md).

Note that this guide utilises "plcnext" crates from [crates.io](https://crates.io) to access PLCnext Control system functions. **These "plcnext" crates are currently in the early stages of development, and should not be used for production applications.** If you would like to contribute to the development of these crates, please let us know on Github.

## Before getting started

Please note that the application developed in this series of articles uses:

* Linux on the host machine, but it should be possible to adapt the Linux commands for Windows.
* Microsoft Visual Studio Code, but any editor or IDE can be used.
* A PLC with IP address 192.168.1.10, but any valid IP address can be used.

## Getting Started

In the following series of technical articles, you will build your own runtime application for PLCnext Control. Each article builds on tasks that were completed in earlier articles, so it is recommended to follow the series in sequence.

|\#| Topic | Objectives |
| --- | ------ | ------ |
|[01](getting-started/Part-01/README.md)| [Hello PLCnext](getting-started/Part-01/README.md)| Create a simple application in Rust, and run it on a PLCnext Control.|
|[02](getting-started/Part-02/README.md)| [PLCnext Control integration](getting-started/Part-02/README.md)| Use the PLCnext Control to start and stop the application automatically.<br/>Create PLCnext ANSI-C component instances.<br/>Write messages to the application log file. |
|[03](getting-started/Part-03/README.md)| [Reading and writing Axioline I/O](getting-started/Part-03/README.md)| Read digital inputs from an Axioline I/O module, and write digital outputs. |
|[04](getting-started/Part-04/README.md)| [Getting PLCnext Control status via callbacks](getting-started/Part-04/README.md)| Use notifications from the PLCnext Control to start and stop data exchange with Axioline I/O modules.<br/>Read the PLC's unique hardware ID. |
|05                                     | Using RSC Services | Coming soon! |
|06                                     | Creating real-time threads | Coming soon! |

---

## How to get support
You can get support in the forum of the [PLCnext Community](https://www.plcnext-community.net/index.php?option=com_easydiscuss&view=categories&Itemid=221&lang=en).

---

Copyright © 2019 Phoenix Contact Electronics GmbH

All rights reserved. This program and the accompanying materials are made available under the terms of the [MIT License](http://opensource.org/licenses/MIT) which accompanies this distribution.