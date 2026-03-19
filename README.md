# GHL POS Terminal Simulator (C Implementation)

## Overview
This repository contains a C-based command-line interface (CLI) application designed to bridge communication between a computer and a physical GHL Point of Sale (POS) terminal (specifically PAX A920 and L920 models). 

This project was rewritten entirely in standard C to facilitate seamless, native integration with the Sentec agent, replacing the legacy Python implementation. It handles standard RS-232/USB serial communication, payload construction, cryptographic checksums, and hexadecimal response parsing based on standard ECR protocols.

## Features
* **Native C Integration:** 100% standard C relying strictly on native Windows APIs (`windows.h`, `setupapi.h`). No external dependencies or frameworks.
* **Hardware Auto-Detection:** Automatically scans the Windows registry to locate and connect to Prolific USB-to-Serial cables.
* **Transaction Support:** Triggers live Sales, Voids, Settlements, and Refunds.
* **Hexadecimal Breakdown:** Intercepts and logs raw TX/RX byte arrays, mapping specific offsets to transaction data (Amounts, Invoices, Authorization Codes, and Terminal Error Codes).
* **Automated Logging:** Maintains a timestamped `communication.log` for debugging and compliance tracking.

## Hardware Setup & Requirements
To successfully run the simulator, you require the following hardware setup:
* Windows OS (due to Windows API dependencies).
* Physical GHL POS Terminal (PAX A920 or L920).
* USB-to-RS232 Serial Cable (Prolific PL2303 chipset recommended).
* Terminal configured to Serial/RS232 communication mode.

## Compilation Instructions
If you need to modify the source code, you can recompile the application using any standard C compiler for Windows, such as GCC (via MinGW).

Because the COM port auto-detection feature relies on native Windows hardware APIs, you must explicitly link the `setupapi` library during compilation. 

From the root directory of this repository, run:
```bash

gcc src/ghl_simulator.c -o bin/GHL_simulator.exe -lsetupapi
```

## Usage
1. Ensure your physical terminal is powered on and the serial cable is connected to your Windows machine.
2. Run `bin/GHL_simulator.exe`.
3. Select **Option 1** to auto-detect the COM port, or **Option 2** to manually enter it.
4. Select a transaction type (e.g., **Option 4** for Sale) and follow the CLI prompts.
5. Interact with the physical POS terminal when prompted (e.g., tap or insert card).
6. Review the console output or the generated `communication.log` in the root directory for the transaction receipt and hex breakdown.

## Packet Structure Protocol
The simulator communicates using a highly specific protocol based on ECR standard framing. Every packet (Transmit/TX and Receive/RX) is wrapped in the following byte envelope: `[STX (0x02)] + [Payload Data] + [8-Byte Checksum/MAC] + [ETX (0x03)]`.

### TX Hex Breakdown (Transmit)
When sending a standard 35-byte transaction packet, the payload (25 bytes) is sliced as follows:
* **Offset [0-3]:** Command Code (`020` = Sale, `022` = Void, `050` = Settlement, `026` = Refund)
* **Offset [3-15]:** Amount in cents (e.g., `000000000001`)
* **Offset [15-21]:** Invoice Number (e.g., `000001`)
* **Offset [21-25]:** Cashier ID (e.g., `  99`)

### RX Hex Breakdown (Receive)
The terminal's response payload is extracted using strict byte offsets:
* **Offset [0-3]:** Command Echo (`021` for Sale Response)
* **Offset [3-5]:** Error / Approval Code (`00` = Success, `CT` = Cancel/Timeout, `51` = Insufficient Funds)
* **Offset [5-27]:** Masked Card Number
* **Offset [27-31]:** Card Expiry (`YYMM`)
* **Offset [31-33]:** Card Type ID (e.g., `08` = MYDEBIT)
* **Offset [33-41]:** Bank Authorization Code
* **Offset [41-53]:** Gross Amount
* **Offset [53-65]:** Net Amount
* **Offset [65-71]:** Trace Number (STAN)
* **Offset [71-77]:** Invoice Number
* **Offset [77-81]:** Cashier ID
* **Offset [81-96]:** Card Brand Name

Extended firmware payload extractions (TID, MID, Batch Number) are also supported if the payload length exceeds 125 bytes.
```
