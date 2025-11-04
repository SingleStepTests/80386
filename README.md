![386_logo](img/386_cow_logo.png)

# 80386

This is a set of emulator CPU tests for the Intel 80386 produced by [Daniel Balsom](https://github.com/dbalsom) using the [ArduinoX86](https://github.com/dbalsom/arduinox86) Arduino-to-CPU interface board, utilizing a `Intel 80386EX KU80C386EX33` CPU.

### Current Version: 1.0.0

- The ```v1_ex_real_mode``` directory contains real mode opcode tests derived from the 386EX.
- The ```v1_ex_unreal_mode``` directory will (eventually) contain unreal mode opcode tests derived from the 386EX.
- The ```v1_ex_protected_mode``` directory will (eventually) contain protected-mode tests derived from the 386EX.
- The ```v1_ex_virtual_mode``` directory will (eventually) contain Virtual 8086-mode tests derived from the 386EX.

## The 386EX

The [386EX](https://en.wikipedia.org/wiki/Intel_80386EX) is a fully static CMOS variation of the 80386 CPU that was intended for the embedded market. It contains a 386CX core and various on-board peripherals. Like the CX and SX, it has a 16-bit data bus. Unlike the 80186 that preceded it, the 386EX's peripherals are largely AT-compatible.

The 386EX has 26 address pins for a maximum address space of 64MiB. Due to GPIO pin limitations on the Arduino, only 24 of the address lines are captured. This limits the effective address space to 16MiB.

There are a few things to keep in mind if using these tests to validate a 32-bit 386DX emulator:

 - All bus cycles will be 16-bit, lengthening instruction timings.
 - Code fetches are still performed 32-bits at a time, via two back-to-back 16-bit code fetches.
 - The introduction of [System Management Mode](https://en.wikipedia.org/wiki/System_Management_Mode) lengthened certain instruction timings. Intel has this to say:
   > SMM also added one to four execution clocks to the following instructions: IN, INS, REP INS, OUT, REP OUT, HALT, and MOV CR0, SRC.  INTR and NMI also need an additional two clocks for interrupt latency. These cycles were added due to the microcode modification for the SMM implementation.

Therefore, unlike my previous test suites, this suite is unlikely to be particularly informative for anyone attempting cycle-accurate CPU emulation of the most common 386 variant, the 386DX, but the practicality of emulating the 386 in a cycle-accurate way is somewhat questionable anyway.

It is theoretically possible that a test suite for the 386DX could be produced in the future, but my hope is that register-level validation of the 386 instruction set will still be of immense value to emulator authors.

## About the Tests

100 - 2,500 tests are provided per valid combination of opcode and operand or address size prefix. 
The number of tests per opcode is roughly determined by their complexity with trivial opcodes such as (`INC reg`, `CLI`, etc.) having the fewest tests.

Each test provides initial and final CPU states including registers and memory. Each test also includes cycle activity for each instruction, including the values of the address and data busses, bus status and control signals, and processor T-state.

All tests assume 16MB of RAM is mapped to the processor and writable.

No wait states are incurred during any of the tests. The interrupt and trap flags are not exercised in the normal test suites.

Due to lack of queue status pins on the 386, and the added complexity of the 386's three-stage decode queue, the prefetch queue is not exercised. All instructions start from a jump to the first opcode or prefix of the instruction, which flushes the queue. The 386 takes some time after a jump to fill the prefetch queue, so most tests will begin with several code fetches.

## Accuracy

Previous tests relied on the MartyPC emulator to help validate test output. Tests were not emitted unless ArduinoX86 and MartyPC agreed on bus activity and final register state. In this manner the CPU and emulator helped verify each other, the CPU showing where MartyPC was inaccurate, and MartyPC catching any errors in hardware test generation. 

The test generator for 386 is a new, standalone implementation that does not tie in to any emulator, relying entirely on the hardware to produce tests. The rationale for rewriting the test generator is to support CPUs that MartyPC does not emulate such as the 286 and 386.

The test generator has a host of error-checking for invalid conditions, but the primary error detection method is a simple one: each test is generated at least twice, and the results are compared.  Two operationally identical tests in a row must be generated for the test to be accepted. This eliminates the vast majority of transient clocking errors. 

I say operationally identical, because periods where bus lines are floating will be - by the nature of physics - random, so these differences are ignored.

After a test is generated, an additional verification pass is performed where the test file is read back from the output `MOO` format, and replayed on the 386 using only the information in the test. This is a final check on test correctness and ensures each test is reproducible.

Still, bugs can creep in. Sometimes a "bug" may not even be incorrect behavior of the CPU, but a divergence from the intent of the test. In any case, in the event that a problematic test is discovered at a later date, the hexadecimal string of the faulty test's `hash` will be added to the `revocation_list.txt` file, one hash per line. 
Since a hash uniquely identifies a test, you can load and compare the hash of each test to the hashes in the revocation list before executing a test.

This way the test suite can be updated for accuracy without requiring large binary updates, although my plan is to periodically correct the source test files and reset the revocation list.

You are encouraged to open issues for any suspect tests you may encounter. If you simply have a question, that's fine too. I am not nearly as knowledgeable about the 386 as I am about the 8088 - creating these tests has been a huge learning experience, so we may have to investigate any oddities together.

## Test Methodology

The tests are created by directly manipulating the 386EX's bus and status pins by connecting them to the GPIO lines of an [Arduino Giga](https://store-usa.arduino.cc/products/giga-r1-wifi) microcontroller. In this manner every cycle of the CPU's activity is known and can be captured.

A randomized initial CPU state can be loaded onto the CPU via the 386's [`LOADALL`](https://www.rcollins.org/articles/loadall/) instruction to begin a test sequence.

Each test is actually a sequence of two instructions, the instruction under test, and a `HALT` opcode **0xF4**. The initial `ram` state includes the opcode byte for the `HALT` instruction.

If the test is a flow control operation, or otherwise triggers an exception, then this initial `HALT` will not be executed. Instead, a `HALT` will be injected at the first code fetch after the jump.

The rationale for injecting `HALT` is to provide a visible signal that an instruction has terminated, since the 386 does not expose queue status pins that allow us to detect instruction boundaries. When the ArduinoX86 CPU server detects the `HALT` bus cycle, it raises the `SMI` line to the CPU which instructs the CPU to enter [System Management Mode](https://en.wikipedia.org/wiki/System_Management_Mode).

Upon entry into SMM, the CPU dumps its internal register state, which we capture as the final register state. When exiting SMM, the CPU reads the internal register state back from memory, in a format similar to but not exactly the same as `LOADALL`.  The test generator takes advantage of each test entering SMM to set the initial state of the next test as we leave `SMM`. This avoids having to perform an expensive CPU reset before each test.  `LOADALL` is used as a fallback to restart the test generation process if the CPU enters shutdown or some other error condition that requires it to be reset.

Since each test terminates via `HALT`, the last included cycle state will encode a `HALT` bus status.

### CPU Shutdown

In certain situations, the CPU may encounter an unrecoverable condition.
 - In real mode due to a stack fault if the stack pointer is odd and < 6.
 - In the case an exception occurs during execution of a double-fault handler, a situation known as a `triple-fault`.

In these cases the CPU will execute a **CPU shutdown**. This is a special `HALT` cycle with an address of `0x000000` placed on the bus. 

To avoid the first situation, `SP` is not allowed to be initialized to less than `0x0008` for any test - this ensures that exception stack frames can be pushed successfully. If during the course of an instruction `SP` becomes < 6, which it can due to arbitrary ALU operations, a shutdown may occur anyway and the test will be rejected and omitted from the test suite. Thus the test suite does not have coverage of this particular scenario.

## Using the Tests

The first thing to do when executing a single step test is to set your emulator state to the initial state provided by the test:

 - Set the register values to the values provided in the initial register and optional descriptor state.
 - Write the bytes listed in the initial memory state to emulator memory at the specified addresses.

Then, begin execution at `CS:IP` as if you have jumped there.
 - End execution at the `HALT` instruction.
 - Compare your emulator's register and descriptor state to the final register and descriptor state. 
 - For each byte in the final memory state, check that the corresponding byte in emulator memory matches.
 - Optionally, confirm your emulator executed the same cycles and/or bus operations as specified in the provided cycles.

## Processor Modes and Test Subsets

The 386 has several modes of operation.

 - Real Mode
 - "Unreal" Mode
 - Protected Mode
 - Virtual 8086 Mode 
 
### Real Mode

Real mode is the CPU's default mode where the CPU's security features are mostly disabled, although segment limit checks are still enforced. 
Typically, only the first 1MB + 64KB of memory was accessible in this mode.

In real mode, segment descriptor bases are initialized directly from the segment register values. The real mode test set will only include the traditional register file in the `initial` state.

The 386's descriptor cache is initialized to default values for each real mode test, which means all segments have a limit of `0xFFFF`.

### Unreal Mode

Unreal mode is still real mode, however using the `LOADALL` instruction, the segment descriptor cache can be initialized to arbitrary values. By setting the segment descriptor base address appropriately, access to the entire 4GB address space is possible in unreal mode.  Unreal mode tests will include the values of the 386's internal descriptor cache entries in the `initial` state.  

### Protected Mode

Protected mode is a mode in which the 386 enforces security for multitasking system operation, enforcing privilege level and segment access checks. A full description of protected mode is beyond the scope of this README.  

Creating tests for protected mode is non-trivial, as memory cannot merely be randomized (or the vast majority of instructions would immediately triple-fault). 

### Virtual 8086 Mode

This is a new mode on the 386 intended to enable multitasking of legacy 8086 software.

Some instructions valid in real mode are not valid in Virtual 8086 mode. Other instructions are otherwise identical.

## Group Opcodes

Some opcodes on the 386 are Group opcodes, where the `reg` field of the ModR/M byte decodes the actual instruction to
execute. In this event the group opcode extension number is appended after the base opcode byte in the filename.

For example, **0x80** extension **4** is AND. The opcode test file for this instruction will be `80.4.MOO`

## Two Byte Opcodes

The 386, like the 286, uses the `0F` byte to signify a two-byte opcode. Two byte instructions will simply include this prefix in filenames, so for example, `0FA0.MOO` is `PUSH FS`.

### FPU Opcodes

FPU opcodes are another form of two-byte opcode, identified by starting with the bit pattern `0b11011`. 11 bits from the first opcode and the second opcode byte are used to differentiate the instruction. For more information on FPU instruction decoding see a relevant 80386 programmer's manual.

FPU opcodes are not currently included in the test set. The 80386 does decode them, but they do nothing useful without a corresponding 80387 math co-processor.

## Operand and Address Size Override Prefixes

Many opcodes on the 386 can be prepended by operand-size and/or address-size override prefixes. These prefixes affect the width of an instruction's base operands, and control the ModR/M table used for address calculations, respectively.

 - The operand-size prefix is `0x66`. 
 - The address-size prefix is `0x67`.

Some instructions can only receive one prefix or the other, where some can accept both. Each supported combination of size override prefix is included as a separate test file. 

For example, opcode **01**, 16-bit `ADD`, can support both an operand-size and address size override prefix. Therefore the test suite will contain the following four opcode test files:

 - `01.MOO`
 - `6601.MOO`
 - `6701.MOO`
 - `676601.MOO`

## Randomization

Previous test suites have blindly randomized register and memory state. There is a minor issue in doing so, in regards to exercising edge cases such as operands of 0 or `0xFFFF`.  For example, given a 16-bit `ADD` instruction, there's only a 0.0015% chance that two random 16-bit numbers sum to 0 and set the zero flag.  For a test set of 5,000 executions, this only gives us a ~7% chance that any of the instructions will do so. When we expand the operand size to 32-bits, the probabilities we exercise the zero flag become vanishingly small.

Therefore, register state is now generated using a beta distribution (α=0.65, β=0.65) that weights values toward the extremes. Additionally, each register has a small chance to be replaced by a list of 'interesting' binary values, that include 0, 1, all-bits-set, and various other bit patterns on specific boundaries designed to exercise edge cases.

In the case of 32-bit register values and displacements used in address calculations, these are masked using the applicable segment limit. This limit mask is additionally shifted right by the scale factor in the SIB byte, if present.

These adjustments keep 32-bit instruction tests from being dominated by General Protection Faults (Exception 13). 

Memory contents, normally randomized, will be forced to all `0x00` bytes or all `0xFF` bytes at a low probability, excluding bytes below address `0x1024`.

Immediate 8-bit, 16-bit and 32-bit operands will also be overridden similar fashion.

Doing this greatly improves test coverage while simultaneously requiring far fewer instructions than previous test suites.

### Stack Pointer

If randomly generated, the stack pointer will be odd with a 50% probability. This is an unnatural condition, and so the stack pointer is specifically forced even at a very high probability. In addition, the stack pointer will be set to `0x0006` at minimum. This is to avoid a processor shutdown when the CPU cannot push a stack frame.

## Instruction Pointer 

The value of IP is not forced to any specific values, but is not allowed to exceed `0xFFF8` to allow the 386 to fill its prefetch queue after a jump.

## Interrupt Vector Table Considerations

In real mode, the Interrupt Vector Table (IVT) exists at address `0`.

Instructions are not allowed to begin below address `0x1024` to avoid writing opcode bytes over the IVT.

The IVT is randomized, thus the corresponding handler address can be anywhere in memory, even at odd alignment.
This will never cause a CPU shutdown, as tests that do so are rejected.

## Segment Override Prefixes

1-5 random segment override prefixes are prepended to a percentage of instructions, even if they may not do anything.

This isn't completely useless - a few bugs have been found where segment overrides had an effect when they should not
have. Instructions where segment prefixes are obviously superfluous are excepted from prefix generation.

It is possible for the number of segment prefixes to increase the instruction length beyond the maximum of 10 bytes. 
An exception interrupt #6 will occur in this case. 

## The LOCK Prefix and LOCK Signal

The `LOCK` prefix is occasionally prepended to instructions, unless they have no memory operands. 
It may appear before, after, or between segment override prefixes.

This is useful for verifying proper handling of lockable vs unlockable instructions. Not all instructions are compatible with the `LOCK` prefix, those that are not will generate Interrupt 6.

The status of the CPU's `LOCK` pin is captured within the included cycle traces. Occasionally, the CPU will assert `LOCK` automatically without a `LOCK` prefix.

## String Prefixes

`REP`, `REPE`, and `REPNE` prefixes are randomly prepended to compatible instructions. In this event, CX is masked to 7 bits to produce reasonably sized tests (A string instruction with `CX==65535` would be several hundred thousand cycles).

## Instruction Prefetching

Bytes fetched beyond the terminating `HALT` opcode will be random. These random bytes are included in the initial `ram` state. It is not critical if your emulator does not fetch all of them.

## Test Formats

The 386 test suite is published in a binary format, `MOO`. This format is a simple and extensible chunked format. 
Traditionally, SingleStepTests have been published in `JSON` format, but this format is not always easily parsed by some languages, like C.

You can find more information about the binary format at the [MOO repository here](https://github.com/dbalsom/moo). 
This repository contains a Rust library crate for working with `MOO` files as well as code for parsing `MOO` files in C, C++ and Python.

If you prefer the traditional `JSON` format, a script `moo2json.py` is available in the `MOO` repository which you can use to convert the test suite from `MOO` to `JSON`. The `JSON` format is slightly different for 386, mostly in the cycles array where things have been simplified and less string parsing is required. See the [Cycle Format](#cycle-format) section below.

Example `JSON` test:

```json
  {
    "idx": 0,
    "name": "add [ss:bp+60h],bl",
    "bytes": [0, 94, 96, 244],
    "initial": {
      "regs": {
        "cr0": 2147418096,
        "cr3": 0,
        "eax": 46917154,
        "ebx": 1747202472,
        "ecx": 3247246033,
        "edx": 4206235108,
        "esi": 8323072,
        "edi": 4054220768,
        "ebp": 524289,
        "esp": 56811,
        "cs": 7970,
        "ds": 809,
        "es": 17715,
        "fs": 0,
        "gs": 3855,
        "ss": 63468,
        "eip": 29344,
        "eflags": 4294707347,
        "dr6": 4294905840,
        "dr7": 0
      },
      "ea": {
        "seg": "SS",
        "sel": 63468,
        "base": 1015488,
        "limit": 65535,
        "offset": 97,
        "l_addr": 1015585,
        "p_addr": 1015585
      },
      "ram": [
        [156864, 0],
        [156865, 94],
        [156866, 96],
        [156867, 244],
        [156868, 63],
        [156869, 216],
        [156870, 35],
        [156871, 243],
        [156872, 48],
        [156873, 40],
        [1015585, 11],
        [156874, 10],
        [156875, 237],
        [156876, 25],
        [156877, 231]
      ],
      "queue": []
    },
    "final": {
      "regs": {
        "eip": 29348,
        "eflags": 4294705298
      },
      "ram": [
        [1015585, 179]
      ],
      "queue": []
    },
    "cycles": [
      [9, 156864, 4, 0, 0, "CODE", 4, "T1"],
      [8, 156864, 4, 0, 24064, "CODE", 4, "T2"],
      [9, 156866, 4, 0, 24064, "CODE", 4, "T1"],
      [8, 156866, 4, 0, 62560, "CODE", 4, "T2"],
      [9, 156868, 4, 0, 62560, "CODE", 4, "T1"],
      [8, 156868, 4, 0, 55359, "CODE", 4, "T2"],
      [9, 156870, 4, 0, 55359, "CODE", 4, "T1"],
      [8, 156870, 4, 0, 62243, "CODE", 4, "T2"],
      [9, 156872, 4, 0, 62243, "CODE", 4, "T1"],
      [8, 156872, 4, 0, 10288, "CODE", 4, "T2"],
      [8, 16777214, 0, 0, 10288, "CODE", 4, "Ti"],
      [9, 1015585, 4, 0, 10288, "MEMR", 6, "T1"],
      [8, 1015585, 4, 0, 2998, "MEMR", 6, "T2"],
      [9, 156874, 4, 0, 2998, "CODE", 4, "T1"],
      [8, 156874, 4, 0, 60682, "CODE", 4, "T2"],
      [9, 156876, 4, 0, 60682, "CODE", 4, "T1"],
      [8, 156876, 4, 0, 59161, "CODE", 4, "T2"],
      [9, 1015585, 1, 0, 45824, "MEMW", 7, "T1"],
      [8, 1015585, 0, 0, 45824, "MEMW", 7, "T2"],
      [8, 16777215, 0, 0, 45824, "MEMW", 7, "Ti"],
      [8, 16777215, 0, 0, 45824, "MEMW", 7, "Ti"],
      [8, 16777215, 0, 0, 45824, "MEMW", 7, "Ti"],
      [8, 16777215, 0, 0, 45824, "MEMW", 7, "Ti"],
      [11, 2, 0, 0, 45824, "HALT", 5, "T1"]
    ],
    "hash": "64456846b886b67084505f8eca4d19943cde4aab"
  },
```
- `idx`: The numerical index of the test within the test file.
- `name`: A human-readable disassembly of the instruction.
- `bytes`: The raw bytes that make up the instruction. This is for your convenience only.
- `initial`: The register and memory state before instruction execution.
- `final`: Changes to registers and memory after instruction execution.
    - Registers and memory locations that are unchanged from the initial state are not included in the final state.
    - The entire value of `flags` is provided if any flag has changed.
- `exception`: An optional key that contains exception data if an exception occurred. See 'Exception Format' below.
- `cycles`: A list of cycle states captured from the CPU. See 'Cycle Format' below.
- `hash`: A SHA1 hash of the original `MOO` test chunk data. It should uniquely identify any test in the suite.

### Effective Address Format

The `ea` key in the initial state is a convenience feature that lists the various components of an effective address calculation in the even that the instruction has a memory operand specified by a ModR/M byte. It contains the following fields:

 - `seg`: The effective segment in human-readable form.
 - `sel`: The segment selector value. In real mode, this is same as 'base >> 4'.
 - `base`: The segment base address. In real mode, this is the same as 'sel << 4'.
 - `limit`: The effective segment's limit. In real mode, this will always be 0xFFFF.
 - `offset`: The value of the calculated offset. If 'offset' > 'limit', an exception should occur.
 - `l_addr`: The logical address calculated from the 'base' + 'offset'. 
 - `p_addr`: The physical address. In real mode, or with paging disabled, this is the same as 'l_addr'.

### Exception Format

The `exception` key is a convenience feature provided so you do not have to attempt exception detection yourself. 
If an exception occurred during instruction execution, the exception number will be given along with the address of the flags register pushed to the stack. This can assist in masking undefined flag values that may exist in the flags register that might otherwise cause memory validation to fail.

```json
    "exception": {
      "number": 13,
      "flag_address": 461420
    },
```

### Cycle Format

It may be tempting to ignore this section, but it contains valuable details about what the physical CPU actually did.

For example, validating by register state and final memory contents alone cannot catch the scenario where your emulator performs excess memory writes. Validating that your emulator performs the same bus cycles as indicated by the `cycles` array may therefore be valuable. This is more of a challenge in this test suite as we are using a CPU with a 16-bit data bus, and a 386 emulator is likely emulating a 32-bit data bus, so exact comparisons are likely not possible. To make it worse, the 386EX can split up 32-bit transfers in unusual ways. Even so, I hope that these cycle states can be useful.

Each entry of the `cycles` array contains the following fields:

 - Pin bitfield
 - Address Bus
 - Memory RW status
 - IO RW status
 - Data Bus
 - Bus Status string
 - Raw Bus Status value
 - T-state string

The first column is a bitfield representing certain chip pin states. 

 - **Bit #0** represents the inverted `ADS` (Address Data Strobe) pin output.  This signal is asserted on `T1` to instruct a motherboard or chipset to store the current address. This is necessary since address calculations are pipelined, so on `T2` the address of the next bus transaction may be on the address lines. **NOTE**: The value of this pin is inverted so that it matches the behavior and meaning of the `ALE` pin in prior test suites.

 - **Bit #1** represents the `BHE` pin output, which is active-low. `BHE` is asserted to activate the upper byte of the data bus. If a bus cycle begins at an even address with `BHE` active, it is a 16-bit transfer. If a bus cycle begins at an odd address with `BHE` active, it is an 8-bit transfer, where the high (odd) byte of the data bus is active.  If a bus cycle begins at an even address with `BHE` inactive, it is an 8-bit transfer where the low (even) byte of the data bus is active. See the [Bus Logic](#bus-logic) section below for a table.

 - **Bit #2** represents the `READY` pin.

 - **Bit #3** represents the `LOCK` pin. The 386 will assert `LOCK` for memory writes when a `LOCK`able instruction is prefixed with the `LOCK` prefix, or in some cases, on its own. 

The **Address Bus** is the value of the 24 address lines, read from the CPU on each cycle. On some cycles the address lines may float.

The **Memory RW status** is a bitfield where bit 2 is the Read signal and bit 0 is the Write signal. Bit 1 is unused.

The **IO RW status** is a bitfield where bit 2 is the Read signal and bit 0 is the Write signal. Bit 1 is unused.

The **Data Bus** is the value of the 16 data bus lines, read from the CPU on each cycle. On some cycles, and given the  state of BHE, some data bus lines may float.

The **Bus Status string** is a decoded bus status for human-readibility. These will be eight values, `INTA`, `IOR`, `IOW`, `MEMR`, `MEMW`, `HALT`, `CODE`, or `PASV`. 

The **Raw Bus Status Value** is a bitfield containing the raw bus status, which may be of some interest as there are more possible states than what are decoded as strings.  

 - **Bit 0** represents the CPU's `RW` pin. 
 - **Bit 1** represents the CPU's `DC` pin. 
 - **Bit 2** represents the CPU's `M/IO` pin.

These three bits uniquely identify 8 different bus status types. Note that a value of '0' can mean passive but is also used by `INTA`. Therefore you will see a lot of `INTA` bus cycles, this does not mean any interrupts are occurring.

The **T-state string** is a convenience string that provides the CPU T-state. This is not a status that is provided directly by the CPU, but is easily calculated based on bus status. The values will be either `T1`, `T2`, or `Ti`.

No queue status is provided. The 386 does not make queue status available.

## Bus Logic

A 16-bit AT bus relies on two signals, `A0`, the first address line, and `BHE`, the Byte High Enable pin.

Read together they determine the size and active halves of the bus:

| BHE    |	A0 | Bus Width | 	   Even/Odd |
|--------|-----|-----------|--------------|
|Active  |	 0 |  	16-bit | even address |
|Active  |	 1 |     8-bit |  odd address |
|Inactive|	 0 |	   8-bit | even address |
|Inactive|	 1 |   	 8-bit |      invalid |

## Undefined Instructions

Unlike the 8088, the 80386 has an `UD` or Invalid Opcode exception, interrupt #6. Most invalid forms of instructions will generate this exception, as will use of a `LOCK` prefix on an incompatible instruction.

Opcodes that only generate a `UD` exception are not included in the test suite.

## Exceptions

On the 386EX, nearly any instruction can potentially execute an exception. Any 16-bit instruction with a memory operand will execute an exception if the address of the operand is `0xFFFF`.  When an exception occurs, the test cycle traces continue through fetching the IVT value for the exception, fetching an injected `HALT` opcode at the ISR address, up until the final `HALT` bus state. 

The `IP` pushed to the stack during exception execution is always the `IP` of the faulting instruction.

Exception execution is noted in both `MOO` and `JSON` format tests, with the exception number provided as well as the address of the flags register pushed to the stack. This is to assist in masking the flag value to handle undefined flags in instructions such as `DIV`. 

## Specific Opcode Notes

| opcode(s)                      | description                                                                                                                                                                                                                                                                                                                                                             |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **0F**                         | This is the first byte of the 386's extended opcode range. Extended opcodes will have a four hex-digit filename such as `0F04.MOO`                                                                                                                                                                                                                                      |
 | **0F 07**                      | The `LOADALL` instruction is not included in the real mode test set. |                                                                                                                                                                                                                                                                                                   |
| **54**                         | `PUSH SP` has slightly different behavior than 8086 class CPUs. The value pushed to the stack is the value of SP before the push.                                                                                                                                                                                                                                       |
| **62**                         | The `BOUND` instruction is a signed range check. Due to the randomized memory used by the instruction generator, 32-bit forms of this instruction will generate a large number of exceptions.                                                                                                                                                                           |                                                                                                                                                                              |
| **6C-6F, A4-A7, AA-AF**        | `CX` is masked to 7 bits. This provides a reasonable test length, as the full 65535 value in `CX` with a `REP` prefix could result in several thousand cycles.                                                                                                                                                                                                          |
| **8F, C6, C7**                 | These forms are only valid for a `reg` field of 0. 5% of tests for these opcodes are allowed to have an invalid `reg` field to help you test your UD exception handling.                                                                                                                                                                                                |
| **C0, C1**                     | The immediate count operand is not masked. The 386 will internally mask the loop count to 5 bits.</br>The `O` flag is undefined for shifts greater than 1. Using the provided flag mask for these opcodes may prevent validation of the `O` flag for counts of 1 and 0.                                                                                                 |
| **D2, D3**                     | `CL` is not masked. The 386 will internally mask the loop count to 5 bits.</br>The `O` flag is undefined for shifts greater than 1. Using the provided flag mask for these opcodes may prevent validation of the `O` flag for counts of 1 and 0.                                                                                                                        |
| **D8-DF**                      | This range includes various floating-point instructions. If tests for these instructions are created in the future, they will have their own subdirectory.                                                                                                                                                                                                              |
| **6C, 6D, E4, E5, EC, ED**     | All IO reads should return `0xFF` or `0xFFFF`.                                                                                                                                                                                                                                                                                                                          |
| **F0**                         | The `LOCK` prefix is exercised in this test set, and the status of the `LOCK` pin is provided in the cycle states.                                                                                                                                                                                                                                                      |                                                                                                                                                                      
| **F1**                         | The `ICEBP` prefix, in the absence of a certain flag set in `DR7` (requiring an actual bond-out CPU), will simply execute an interrupt #1.                                                                                                                                                                                                                              |
| **F4**                         | The `HALT` instruction terminates all instructions in the test suite. The `HALT` instruction itself is included, and simply terminates itself.                                                                                                                                                                                                                          |
| **D4, F6.6, F6.7, F7.6, F7.7** | On the 8088 the return address pushed to the stack on divide exception was the address of the next instruction. On the 286 and later, the address pushed to the stack is the address of the faulting instruction. On the 8088, divisors of `0x80` (for 8-bit division) or `0x8000` (for 16-bit division) generate divide exceptions. On the 286 and later, they do not. |
| **FA**                         | `CLI` is essentially a `NOP` in the test suite as the `I` flag is never set.                                                                                                                                                                                                                                                                                            |


## 80386.csv

This is a database of opcode information for the 80386 instruction set. Each row corresponds to a unique instruction.
This file replaces the `metadata.json` file from previous test suites.

Of most interest to most will likely be the `f_umask` field here that provides 16-bit mask values for flag values left undefined by certain instructions. 

The following fields are defined. A bool value is considered false if either `0` or no value is present.
Additional fields may be present, but they may not be validated for correctness - use at your discretion.

| Field       | Type           | Description                                       |
|-------------|----------------|---------------------------------------------------|
| `op`        | hexadecimal    | Opcode, either two or four hex digits             |
| `re`        | bool           | Instruction has opcode-encoded register operand   |
| `g`         | decimal        | Sandpile.org group number                         |
| `ex`        | decimal        | ModR\M-encoded opcode extension (0-7)             |
| `ud`        | bool           | Instruction form is undefined / invalid           |
| `pf`        | bool           | Opcode is a prefix                                |
| `66`        | bool           | Opcode can accept a operand-size override prefix  | 
| `67`        | bool           | Opcode can accept an address-size override prefix |
| `fc`        | bool           | Opcode is a flow control/branch instruction       |
| `pm`        | bool           | Opcode only valid in protected mode               |
| `lck`       | bool           | Opcode can accept a `LOCK` prefix                   |
| `rep`       | bool           | Opcode can accept a `REP` prefix                    |
| `so`        | bool           | Opcode can accept a segment override prefix       |
| `reg`       | bool           | Register form valid for ModR\M operand            |
| `m`         | bool           | Opcode has ModR\M operand                         |
| `mnemonic`  | ASCII          | Mnemonic corresponding to this instruction        |
| `f_umask`   | hexadecimal    | Undefined flag mask                               |
| `desc`      | ASCII          | Human-readable description                        |


## FAQ
 
### Why not an 386DX?

There are several advantages to the 386EX in regards to the task of producing CPU tests.
 - Fully static design: This means we can clock it as slowly as we need to
 - Lower pin count: The 16-bit data bus requires fewer GPIO lines
 - System Management Mode: The inclusion of SMM on the 386EX allows us to capture the internal register state after instruction execution
 - Smaller size: Fits on a shield
 - Lower voltage and power requirements

### Why are instructions longer than expected?

The 386 fills its instruction prefetch queue after a jump. Thus every instruction execution will begin with several code fetch bus cycles before any opcode is executed. The lack of queue status pins on the 386 prevents the precise queue status and instruction execution tracking that was possible on the 8088, therefore we use a `HALT` instruction after the instruction under test to determine when execution has ended. `HALT` takes a number of cycles itself to actually halt on the 386EX, therefore execution is lengthened further.

As a general rule of thumb, you can subtract 14 cycles from each test to represent this overhead, but this isn't perfect. 

Some instruction executions may generate interrupts or exceptions. Test cycles in this case continue through fetching the IVT entry for the exception, jumping there, and then executing a `HALT` within the corresponding exception handler.

### How do I handle instructions with undefined flags that generate exceptions?

Instructions that generate exceptions will push the value of the flags register to the stack. If the flags register contains undefined flag values, this may present a difficulty for emulator authors who have yet to implement undefined flag behavior.

Exceptions are noted in the test files for you, with the exception number that was executed and the address of the flag word. You can use this address to mask the undefined flags before comparing your final `ram` state. 

### When will we see protected mode tests?

Creating protected mode tests is non-trivial, as memory cannot simply be randomized - valid descriptor tables must be generated in memory to avoid instructions immediately triple-faulting. I cannot say when protected mode tests will appear, all I can do is promise that I'll work on it.

### Credits

 - 80386.csv contains information derived from the excellent [x86reference](https://github.com/mazegen/x86reference) by MazeGen.

### Further Resources
 
 - [ref.x86asm.net](http://ref.x86asm.net/) - an excellent x86 ISA reference by MazeGen and contributors
 - [sandpile.org](https://www.sandpile.org/x86/) - another excellent x86 ISA reference
 - [test386](https://github.com/barotto/test386.asm) - an excellent test ROM for the 386 by Marco Bortolin
 - [The 386DX Microprocessor Programmer's Reference Manual](http://www.bitsavers.org/components/intel/80386/230985-003_386DX_Microprocessor_Programmers_Reference_Manual_1990.pdf) - (Bitsavers.org)
 - [The LOADALL instruction](https://www.rcollins.org/articles/loadall/) - (rcollins.org)

### Special Thanks To

 - **Thomas Harte** for developing the first SingleStepTests
 - **Andreas Jonsson** for giving me the idea of hardware validation
 - **Robert Collins** for his excellent articles on the 80386
 - **reenigne** for reverse-engineering the 80386 microcode
 - **Everyone** who has used these tests and given me feedback!


