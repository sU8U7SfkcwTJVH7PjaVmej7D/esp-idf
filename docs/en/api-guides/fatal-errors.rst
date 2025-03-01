Fatal Errors
============
:link_to_translation:`zh_CN:[中文]`

.. _Overview:

Overview
--------

In certain situations, execution of the program can not be continued in a well defined way. In ESP-IDF, these situations include:

- CPU Exceptions: |CPU_EXCEPTIONS_LIST|
- System level checks and safeguards:

  - :doc:`Interrupt watchdog <../api-reference/system/wdts>` timeout
  - :doc:`Task watchdog <../api-reference/system/wdts>` timeout (only fatal if :ref:`CONFIG_ESP_TASK_WDT_PANIC` is set)
  - Cache access error
  - Brownout detection event
  - Stack overflow
  - Stack smashing protection check
  - Heap integrity check
  - Undefined behavior sanitizer (UBSAN) checks

- Failed assertions, via ``assert``, ``configASSERT`` and similar macros.

This guide explains the procedure used in ESP-IDF for handling these errors, and provides suggestions on troubleshooting the errors.

Panic Handler
-------------

Every error cause listed in the `Overview`_ will be handled by *panic handler*.

Panic handler will start by printing the cause of the error to the console. For CPU exceptions, the message will be similar to

.. parsed-literal::

    Guru Meditation Error: Core 0 panic'ed (|ILLEGAL_INSTR_MSG|). Exception was unhandled.

For some of the system level checks (interrupt watchdog, cache access error), the message will be similar to

.. parsed-literal::

    Guru Meditation Error: Core 0 panic'ed (|CACHE_ERR_MSG|). Exception was unhandled.

In all cases, error cause will be printed in parentheses. See `Guru Meditation Errors`_ for a list of possible error causes.

Subsequent behavior of the panic handler can be set using :ref:`CONFIG_ESP_SYSTEM_PANIC` configuration choice. The available options are:

- Print registers and reboot (``CONFIG_ESP_SYSTEM_PANIC_PRINT_REBOOT``) — default option.

  This will print register values at the point of the exception, print the backtrace, and restart the chip.

- Print registers and halt (``CONFIG_ESP_SYSTEM_PANIC_PRINT_HALT``)

  Similar to the above option, but halt instead of rebooting. External reset is required to restart the program.

- Silent reboot (``CONFIG_ESP_SYSTEM_PANIC_SILENT_REBOOT``)

  Don't print registers or backtrace, restart the chip immediately.

- Invoke GDB Stub (``CONFIG_ESP_SYSTEM_PANIC_GDBSTUB``)

  Start GDB server which can communicate with GDB over console UART port. See `GDB Stub`_ for more details.

Behavior of panic handler is affected by two other configuration options.

- If :ref:`CONFIG_{IDF_TARGET_CFG_PREFIX}_DEBUG_OCDAWARE` is enabled (which is the default), panic handler will detect whether a JTAG debugger is connected. If it is, execution will be halted and control will be passed to the debugger. In this case registers and backtrace are not dumped to the console, and GDBStub / Core Dump functions are not used.

- If :doc:`Core Dump <core_dump>` feature is enabled, then system state (task stacks and registers) will be dumped either to Flash or UART, for later analysis.

- If :ref:`CONFIG_ESP_PANIC_HANDLER_IRAM` is disabled (disabled by default), the panic handler code is placed in flash memory not IRAM. This means that if ESP-IDF crashes while flash cache is disabled, the panic handler will automatically re-enable flash cache before running GDB Stub or Core Dump. This adds some minor risk, if the flash cache status is also corrupted during the crash.

  If this option is enabled, the panic handler code (including required UART functions) is placed in IRAM. This may be necessary to debug some complex issues with crashes while flash cache is disabled (for example, when writing to SPI flash) or when flash cache is corrupted when an exception is triggered.

The following diagram illustrates panic handler behavior:

.. blockdiag::
    :scale: 100%
    :caption: Panic Handler Flowchart (click to enlarge)
    :align: center

    blockdiag panic-handler {
        orientation = portrait;
        edge_layout = flowchart;
        default_group_color = white;
        node_width = 160;
        node_height = 60;

        cpu_exception [label = "CPU Exception", shape=roundedbox];
        sys_check [label = "Cache error,\nInterrupt WDT,\nabort()", shape=roundedbox];
        check_ocd [label = "JTAG debugger\nconnected?", shape=diamond, height=80];
        print_error_cause [label = "Print error/\nexception cause"];
        use_jtag [label = "Send signal to\nJTAG debugger", shape=roundedbox];
        dump_registers [label = "Print registers\nand backtrace"];
        check_coredump [label = "Core dump\nenabled?", shape=diamond, height=80];
        do_coredump [label = "Core dump\nto UART or Flash"];
        check_gdbstub [label = "GDB Stub\nenabled?", shape=diamond, height=80];
        do_gdbstub [label = "Start GDB Stub", shape=roundedbox];
        halt [label = "Halt", shape=roundedbox];
        reboot [label = "Reboot", shape=roundedbox];
        check_halt [label = "Halt?", shape=diamond, height=80];

        group {cpu_exception, sys_check};

        cpu_exception -> print_error_cause;
        sys_check -> print_error_cause;
        print_error_cause -> check_ocd;
        check_ocd -> use_jtag [label = "Yes"];
        check_ocd -> dump_registers [label = "No"];
        dump_registers -> check_coredump
        check_coredump -> do_coredump [label = "Yes"];
        do_coredump -> check_gdbstub;
        check_coredump -> check_gdbstub [label = "No"];
        check_gdbstub -> check_halt [label = "No"];
        check_gdbstub -> do_gdbstub [label = "Yes"];
        check_halt -> halt [label = "Yes"];
        check_halt -> reboot [label = "No"];
    }

Register Dump and Backtrace
---------------------------

Unless ``CONFIG_ESP_SYSTEM_PANIC_SILENT_REBOOT`` option is enabled, panic handler prints some of the CPU registers, and the backtrace, to the console

.. only:: CONFIG_IDF_TARGET_ARCH_XTENSA

    ::

        Core 0 register dump:
        PC      : 0x400e14ed  PS      : 0x00060030  A0      : 0x800d0805  A1      : 0x3ffb5030
        A2      : 0x00000000  A3      : 0x00000001  A4      : 0x00000001  A5      : 0x3ffb50dc
        A6      : 0x00000000  A7      : 0x00000001  A8      : 0x00000000  A9      : 0x3ffb5000
        A10     : 0x00000000  A11     : 0x3ffb2bac  A12     : 0x40082d1c  A13     : 0x06ff1ff8
        A14     : 0x3ffb7078  A15     : 0x00000000  SAR     : 0x00000014  EXCCAUSE: 0x0000001d
        EXCVADDR: 0x00000000  LBEG    : 0x4000c46c  LEND    : 0x4000c477  LCOUNT  : 0xffffffff

        Backtrace: 0x400e14ed:0x3ffb5030 0x400d0802:0x3ffb5050

.. only:: CONFIG_IDF_TARGET_ARCH_RISCV

    ::

        Core  0 register dump:
        MEPC    : 0x420048b4  RA      : 0x420048b4  SP      : 0x3fc8f2f0  GP      : 0x3fc8a600
        TP      : 0x3fc8a2ac  T0      : 0x40057fa6  T1      : 0x0000000f  T2      : 0x00000000
        S0/FP   : 0x00000000  S1      : 0x00000000  A0      : 0x00000001  A1      : 0x00000001
        A2      : 0x00000064  A3      : 0x00000004  A4      : 0x00000001  A5      : 0x00000000
        A6      : 0x42001fd6  A7      : 0x00000000  S2      : 0x00000000  S3      : 0x00000000
        S4      : 0x00000000  S5      : 0x00000000  S6      : 0x00000000  S7      : 0x00000000
        S8      : 0x00000000  S9      : 0x00000000  S10     : 0x00000000  S11     : 0x00000000
        T3      : 0x00000000  T4      : 0x00000000  T5      : 0x00000000  T6      : 0x00000000
        MSTATUS : 0x00001881  MTVEC   : 0x40380001  MCAUSE  : 0x00000007  MTVAL   : 0x00000000
        MHARTID : 0x00000000

Register values printed are the register values in the exception frame, i.e. values at the moment when CPU exception or other fatal error has occured.

Register dump is not printed if the panic handler was executed as a result of an ``abort()`` call.

.. only:: CONFIG_IDF_TARGET_ARCH_XTENSA
    
    In some cases, such as interrupt watchdog timeout, panic handler may print additional CPU registers (EPC1-EPC4) and the registers/backtrace of the code running on the other CPU.

    Backtrace line contains PC:SP pairs, where PC is the Program Counter and SP is Stack Pointer, for each stack frame of the current task. If a fatal error happens inside an ISR, the backtrace may include PC:SP pairs both from the task which was interrupted, and from the ISR.

If :doc:`IDF Monitor <tools/idf-monitor>` is used, Program Counter values will be converted to code locations (function name, file name, and line number), and the output will be annotated with additional lines

.. only:: CONFIG_IDF_TARGET_ARCH_XTENSA

    ::

        Core 0 register dump:
        PC      : 0x400e14ed  PS      : 0x00060030  A0      : 0x800d0805  A1      : 0x3ffb5030
        0x400e14ed: app_main at /Users/user/esp/example/main/main.cpp:36

        A2      : 0x00000000  A3      : 0x00000001  A4      : 0x00000001  A5      : 0x3ffb50dc
        A6      : 0x00000000  A7      : 0x00000001  A8      : 0x00000000  A9      : 0x3ffb5000
        A10     : 0x00000000  A11     : 0x3ffb2bac  A12     : 0x40082d1c  A13     : 0x06ff1ff8
        0x40082d1c: _calloc_r at /Users/user/esp/esp-idf/components/newlib/syscalls.c:51

        A14     : 0x3ffb7078  A15     : 0x00000000  SAR     : 0x00000014  EXCCAUSE: 0x0000001d
        EXCVADDR: 0x00000000  LBEG    : 0x4000c46c  LEND    : 0x4000c477  LCOUNT  : 0xffffffff

        Backtrace: 0x400e14ed:0x3ffb5030 0x400d0802:0x3ffb5050
        0x400e14ed: app_main at /Users/user/esp/example/main/main.cpp:36

        0x400d0802: main_task at /Users/user/esp/esp-idf/components/{IDF_TARGET_PATH_NAME}/cpu_start.c:470

.. only:: CONFIG_IDF_TARGET_ARCH_RISCV

    ::

        Core  0 register dump:
        MEPC    : 0x420048b4  RA      : 0x420048b4  SP      : 0x3fc8f2f0  GP      : 0x3fc8a600
        0x420048b4: app_main at /Users/user/esp/example/main/hello_world_main.c:20

        0x420048b4: app_main at /Users/user/esp/example/main/hello_world_main.c:20

        TP      : 0x3fc8a2ac  T0      : 0x40057fa6  T1      : 0x0000000f  T2      : 0x00000000
        S0/FP   : 0x00000000  S1      : 0x00000000  A0      : 0x00000001  A1      : 0x00000001
        A2      : 0x00000064  A3      : 0x00000004  A4      : 0x00000001  A5      : 0x00000000
        A6      : 0x42001fd6  A7      : 0x00000000  S2      : 0x00000000  S3      : 0x00000000
        0x42001fd6: uart_write at /Users/user/esp/esp-idf/components/vfs/vfs_uart.c:201

        S4      : 0x00000000  S5      : 0x00000000  S6      : 0x00000000  S7      : 0x00000000
        S8      : 0x00000000  S9      : 0x00000000  S10     : 0x00000000  S11     : 0x00000000
        T3      : 0x00000000  T4      : 0x00000000  T5      : 0x00000000  T6      : 0x00000000
        MSTATUS : 0x00001881  MTVEC   : 0x40380001  MCAUSE  : 0x00000007  MTVAL   : 0x00000000
        MHARTID : 0x00000000

    Moreover, the :doc:`IDF Monitor <tools/idf-monitor>` is also capable of generating and printing a backtrace thanks to the stack dump provided by the board in the panic handler. 
    The output looks like this:

    ::

        Backtrace:

        0x42006686 in bar (ptr=ptr@entry=0x0) at ../main/hello_world_main.c:18
        18	    *ptr = 0x42424242;
        #0  0x42006686 in bar (ptr=ptr@entry=0x0) at ../main/hello_world_main.c:18
        #1  0x42006692 in foo () at ../main/hello_world_main.c:22
        #2  0x420066ac in app_main () at ../main/hello_world_main.c:28
        #3  0x42015ece in main_task (args=<optimized out>) at /Users/user/esp/components/freertos/port/port_common.c:142
        #4  0x403859b8 in vPortEnterCritical () at /Users/user/esp/components/freertos/port/riscv/port.c:130
        #5  0x00000000 in ?? ()
        Backtrace stopped: frame did not save the PC

    While the backtrace above is very handy, it requires the user to use :doc:`IDF Monitor <tools/idf-monitor>`. Thus, in order to generate and print a backtrace while using another monitor program, it is possible to activate :ref:`CONFIG_ESP_SYSTEM_USE_EH_FRAME` option from the menuconfig.

    This option will let the compiler generate DWARF information for each function of the project. Then, when a CPU exception occurs, the panic handler will parse these data and determine the backtrace of the task that failed. The output looks like this:
    
    ::

        Backtrace: 0x42009e9a:0x3fc92120 0x42009ea6:0x3fc92120 0x42009ec2:0x3fc92130 0x42024620:0x3fc92150 0x40387d7c:0x3fc92160 0xfffffffe:0x3fc92170    
    
    These ``PC:SP`` pairs represent the PC (Program Counter) and SP (Stack Pointer) for each stack frame of the current task.


    The main benefit of the :ref:`CONFIG_ESP_SYSTEM_USE_EH_FRAME` option is that the backtrace is generated by the board itself (without the need for :doc:`IDF Monitor <tools/idf-monitor>`). However, the option's drawback is that it results in an increase of the compiled binary's size (ranging from 20% to 100% increase in size). Furthermore, this option causes debug information to be included within the compiled binary. Therefore, users are strongly advised not to enable this option in mass/final production builds.
    
To find the location where a fatal error has happened, look at the lines which follow the "Backtrace" line. Fatal error location is the top line, and subsequent lines show the call stack.

.. _GDB-Stub:

GDB Stub
--------

If ``CONFIG_ESP_SYSTEM_PANIC_GDBSTUB`` option is enabled, panic handler will not reset the chip when fatal error happens. Instead, it will start GDB remote protocol server, commonly referred to as GDB Stub. When this happens, GDB instance running on the host computer can be instructed to connect to the {IDF_TARGET_NAME} UART port.

If :doc:`IDF Monitor <tools/idf-monitor>` is used, GDB is started automatically when GDB Stub prompt is detected on the UART. The output would look like this::

    Entering gdb stub now.
    $T0b#e6GNU gdb (crosstool-NG crosstool-ng-1.22.0-80-gff1f415) 7.10
    Copyright (C) 2015 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
    and "show warranty" for details.
    This GDB was configured as "--host=x86_64-build_apple-darwin16.3.0 --target={IDF_TARGET_TOOLCHAIN_PREFIX}".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    <http://www.gnu.org/software/gdb/bugs/>.
    Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.
    For help, type "help".
    Type "apropos word" to search for commands related to "word"...
    Reading symbols from /Users/user/esp/example/build/example.elf...done.
    Remote debugging using /dev/cu.usbserial-31301
    0x400e1b41 in app_main ()
        at /Users/user/esp/example/main/main.cpp:36
    36      *((int*) 0) = 0;
    (gdb)

GDB prompt can be used to inspect CPU registers, local and static variables, and arbitrary locations in memory. It is not possible to set breakpoints, change PC, or continue execution. To reset the program, exit GDB and perform external reset: Ctrl-T Ctrl-R in IDF Monitor, or using external reset button on the development board.

.. _Guru-Meditation-Errors:

Guru Meditation Errors
----------------------

.. Note to editor: titles of the following section need to match exception causes printed by the panic handler. Do not change the titles (insert spaces, reword, etc.) unless panic handler messages are also changed.

.. Note to translator: When translating this section, avoid translating the following section titles. "Guru Meditation" in the title of this section should also not be translated. Keep these two notes when translating.

This section explains the meaning of different error causes, printed in parens after ``Guru Meditation Error: Core panic'ed`` message.

.. note:: See `Wikipedia article <https://en.wikipedia.org/wiki/Guru_Meditation>`_ for historical origins of "Guru Meditation".


|ILLEGAL_INSTR_MSG|
^^^^^^^^^^^^^^^^^^^

This CPU exception indicates that the instruction which was executed was not a valid instruction. Most common reasons for this error include:

- FreeRTOS task function has returned. In FreeRTOS, if task function needs to terminate, it should call :cpp:func:`vTaskDelete` function and delete itself, instead of returning.

- Failure to load next instruction from SPI flash. This usually happens if:

  - Application has reconfigured SPI flash pins as some other function (GPIO, UART, etc.). Consult Hardware Design Guidelines and the Datasheet for the chip or module for details about SPI flash pins.

  - Some external device was accidentally connected to SPI flash pins, and has interfered with communication between {IDF_TARGET_NAME} and SPI flash.

.. only:: CONFIG_IDF_TARGET_ARCH_XTENSA

    InstrFetchProhibited
    ^^^^^^^^^^^^^^^^^^^^

    This CPU exception indicates that CPU could not load an instruction because the the address of the instruction did not belong to a valid region in instruction RAM or ROM.

    Usually this means an attempt to call a function pointer, which does not point to valid code. ``PC`` (Program Counter) register can be used as an indicator: it will be zero or will contain garbage value (not ``0x4xxxxxxx``).

    LoadProhibited, StoreProhibited
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    This CPU exception happens when application attempts to read from or write to an invalid memory location. The address which was written/read is found in ``EXCVADDR`` register in the register dump. If this address is zero, it usually means that application attempted to dereference a NULL pointer. If this address is close to zero, it usually means that application attempted to access member of a structure, but the pointer to the structure was NULL. If this address is something else (garbage value, not in ``0x3fxxxxxx`` - ``0x6xxxxxxx`` range), it likely means that the pointer used to access the data was either not initialized or was corrupted.

    IntegerDivideByZero
    ^^^^^^^^^^^^^^^^^^^

    Application has attempted to do integer division by zero.

    LoadStoreAlignment
    ^^^^^^^^^^^^^^^^^^

    Application has attempted to read or write memory location, and address alignment did not match load/store size. For example, 32-bit load can only be done from 4-byte aligned address, and 16-bit load can only be done from a 2-byte aligned address.

    LoadStoreError
    ^^^^^^^^^^^^^^

    This exception may happen in the following cases:

    - If the application has attempted to do an 8- or 16- bit load/store from a memory region which only supports 32-bit loads/stores. For example, dereferencing a ``char*`` pointer to intruction memory (IRAM, IROM) will result in such an error.

    - If the application has attempted a store to a read-only memory region, such as IROM or DROM.

    Unhandled debug exception
    ^^^^^^^^^^^^^^^^^^^^^^^^^

    This will usually be followed by a message like::

        Debug exception reason: Stack canary watchpoint triggered (task_name)

    This error indicates that application has written past the end of the stack of ``task_name`` task. Note that not every stack overflow is guaranteed to trigger this error. It is possible that the task writes to stack beyond the stack canary location, in which case the watchpoint will not be triggered.

.. only:: CONFIG_IDF_TARGET_ARCH_RISCV
    
    Instruction address misaligned
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    This CPU exception indicates that the address of the instruction to execute is not 2-byte aligned.

    Instruction access fault, Load access fault, Store access fault
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    This CPU exception happens when application attempts to execute, read from or write to an invalid memory location. The address which was written/read is found in ``MTVAL`` register in the register dump. If this address is zero, it usually means that application attempted to dereference a NULL pointer. If this address is close to zero, it usually means that application attempted to access member of a structure, but the pointer to the structure was NULL. If this address is something else (garbage value, not in ``0x3fxxxxxx`` - ``0x6xxxxxxx`` range), it likely means that the pointer used to access the data was either not initialized or was corrupted.

    Breakpoint
    ^^^^^^^^^^

    This CPU exception happens when the instruction ``EBREAK`` is executed.

    Load address misaligned, Store address misaligned
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    Application has attempted to read or write memory location, and address alignment did not match load/store size. For example, 32-bit load can only be done from 4-byte aligned address, and 16-bit load can only be done from a 2-byte aligned address.

Interrupt wdt timeout on CPU0 / CPU1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Indicates that interrupt watchdog timeout has occured. See :doc:`Watchdogs <../api-reference/system/wdts>` for more information.

|CACHE_ERR_MSG|
^^^^^^^^^^^^^^^

In some situations ESP-IDF will temporarily disable access to external SPI Flash and SPI RAM via caches. For example, this happens with spi_flash APIs are used to read/write/erase/mmap regions of SPI Flash. In these situations, tasks are suspended, and interrupt handlers not registered with ``ESP_INTR_FLAG_IRAM`` are disabled. Make sure that any interrupt handlers registered with this flag have all the code and data in IRAM/DRAM. Refer to the :ref:`SPI flash API documentation <iram-safe-interrupt-handlers>` for more details.

Other Fatal Errors
------------------

Brownout
^^^^^^^^

{IDF_TARGET_NAME} has a built-in brownout detector, which is enabled by default. Brownout detector can trigger system reset if supply voltage goes below safe level. Brownout detector can be configured using :ref:`CONFIG_{IDF_TARGET_CFG_PREFIX}_BROWNOUT_DET` and :ref:`CONFIG_{IDF_TARGET_CFG_PREFIX}_BROWNOUT_DET_LVL_SEL` options.

When brownout detector triggers, the following message is printed::

    Brownout detector was triggered

Chip is reset after the message is printed.

Note that if supply voltage is dropping at a fast rate, only part of the message may be seen on the console.

Corrupt Heap
^^^^^^^^^^^^

ESP-IDF heap implementation contains a number of run-time checks of heap structure. Additional checks ("Heap Poisoning") can be enabled in menuconfig. If one of the checks fails, message similar to the following will be printed::

    CORRUPT HEAP: Bad tail at 0x3ffe270a. Expected 0xbaad5678 got 0xbaac5678
    assertion "head != NULL" failed: file "/Users/user/esp/esp-idf/components/heap/multi_heap_poisoning.c", line 201, function: multi_heap_free
    abort() was called at PC 0x400dca43 on core 0

Consult :doc:`Heap Memory Debugging <../api-reference/system/heap_debug>` documentation for further information.

Stack Smashing
^^^^^^^^^^^^^^

Stack smashing protection (based on GCC ``-fstack-protector*`` flags) can be enabled in ESP-IDF using :ref:`CONFIG_COMPILER_STACK_CHECK_MODE` option. If stack smashing is detected, message similar to the following will be printed::

    Stack smashing protect failure!

    abort() was called at PC 0x400d2138 on core 0

    Backtrace: 0x4008e6c0:0x3ffc1780 0x4008e8b7:0x3ffc17a0 0x400d2138:0x3ffc17c0 0x400e79d5:0x3ffc17e0 0x400e79a7:0x3ffc1840 0x400e79df:0x3ffc18a0 0x400e2235:0x3ffc18c0 0x400e1916:0x3ffc18f0 0x400e19cd:0x3ffc1910 0x400e1a11:0x3ffc1930 0x400e1bb2:0x3ffc1950 0x400d2c44:0x3ffc1a80
    0

The backtrace should point to the function where stack smashing has occured. Check the function code for unbounded access to local arrays.

.. only:: CONFIG_IDF_TARGET_ARCH_XTENSA

    .. |CPU_EXCEPTIONS_LIST| replace:: Illegal Instruction, Load/Store Alignment Error, Load/Store Prohibited error, Double Exception.
    .. |ILLEGAL_INSTR_MSG| replace:: IllegalInstruction
    .. |CACHE_ERR_MSG| replace:: Cache disabled but cached memory region accessed

.. only:: CONFIG_IDF_TARGET_ARCH_RISCV

    .. |CPU_EXCEPTIONS_LIST| replace:: Illegal Instruction, Load/Store Alignment Error, Load/Store Prohibited error.
    .. |ILLEGAL_INSTR_MSG| replace:: Illegal instruction
    .. |CACHE_ERR_MSG| replace:: Cache error

Undefined behavior sanitizer (UBSAN) checks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Undefined behavior sanitizer (UBSAN) is a compiler feature which adds run-time checks for potentially incorrect operations, such as:

- overflows (multiplication overflow, signed integer overflow)
- shift base or exponent errors (e.g. shift by more than 32 bits)
- integer conversion errors

See `GCC documentation <https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html>`_ of ``-fsanitize=undefined`` option for the complete list of supported checks.

Enabling UBSAN
""""""""""""""

UBSAN is disabled by default. It can be enabled at file, component, or project level by adding ``-fsanitize=undefined`` compiler option in the build system.

When enabling UBSAN for the code which uses hardware register header files (``soc/xxx_reg.h``), it is recommended to disable shift-base sanitizer using ``-fno-sanitize=shift-base`` option. This is due to the fact that ESP-IDF register header files currently contain patterns which cause false positives for this specific sanitizer option.

To enable UBSAN at project level, add the following at the end of the project CMakeLists.txt file::

    idf_build_set_property(COMPILE_OPTIONS "-fsanitize=undefined" "-fno-sanitize=shift-base" APPEND)

Alternatively, pass these options through ``EXTRA_CFLAGS`` and ``EXTRA_CXXFLAGS`` environment variables.

Enabling UBSAN results in significant increase of code and data size. Most applications, except for the trivial ones, will not fit into the available RAM of the microcontroller when UBSAN is enabled for the whole application. Therefore it is recommended that UBSAN is instead enabled for specific components under test.

To enable UBSAN for the specific component (``component_name``) from the project CMakeLists.txt file, add the following at the end of the file::

    idf_component_get_property(lib component_name COMPONENT_LIB)
    target_compile_options(${lib} PRIVATE "-fsanitize=undefined" "-fno-sanitize=shift-base")

.. note:: See the build system documentation for more information about :ref:`build properties<cmake-build-properties>` and :ref:`component properties<cmake-component-properties>`.

To enable UBSAN for the specific component (``component_name``) from CMakeLists.txt of the same component, add the following at the end of the file::

    target_compile_options(${COMPONENT_LIB} PRIVATE "-fsanitize=undefined" "-fno-sanitize=shift-base")

UBSAN output
""""""""""""

When UBSAN detects an error, a message and the backtrace are printed, for example::

    Undefined behavior of type out_of_bounds

    Backtrace:0x4008b383:0x3ffcd8b0 0x4008c791:0x3ffcd8d0 0x4008c587:0x3ffcd8f0 0x4008c6be:0x3ffcd950 0x400db74f:0x3ffcd970 0x400db99c:0x3ffcd9a0

When using :doc:`IDF Monitor <tools/idf-monitor>`, the backtrace will be decoded to function names and source code locations, pointing to the location where the issue has happened (here it is ``main.c:128``)::

    0x4008b383: panic_abort at /path/to/esp-idf/components/esp_system/panic.c:367

    0x4008c791: esp_system_abort at /path/to/esp-idf/components/esp_system/system_api.c:106

    0x4008c587: __ubsan_default_handler at /path/to/esp-idf/components/esp_system/ubsan.c:152

    0x4008c6be: __ubsan_handle_out_of_bounds at /path/to/esp-idf/components/esp_system/ubsan.c:223

    0x400db74f: test_ub at main.c:128

    0x400db99c: app_main at main.c:56 (discriminator 1)

The types of errors reported by UBSAN can be as follows:

.. list-table::
  :widths: 40 60
  :header-rows: 1

  * - Name
    - Meaning
  * - ``type_mismatch``, ``type_mismatch_v1``
    - Incorrect pointer value: null, unaligned, not compatible with the given type.
  * - ``add_overflow``, ``sub_overflow``, ``mul_overflow``, ``negate_overflow``
    - Integer overflow during addition, subtraction, multiplication, negation.
  * - ``divrem_overflow``
    - Integer division by 0 or ``INT_MIN``.
  * - ``shift_out_of_bounds``
    - Overflow in left or right shift operators.
  * - ``out_of_bounds``
    - Access outside of bounds of an array.
  * - ``unreachable``
    - Unreachable code executed.
  * - ``missing_return``
    - Non-void function has reached its end without returning a value (C++ only).
  * - ``vla_bound_not_positive``
    - Size of variable length array is not positive.
  * - ``load_invalid_value``
    - Value of ``bool`` or ``enum`` (C++ only) variable is invalid (out of bounds).
  * - ``nonnull_arg``
    - Null argument passed to a function which is declared with a ``nonnull`` attribute.
  * - ``nonnull_return``
    - Null value returned from a function which is declared with ``returns_nonnull`` attribute.
  * - ``builtin_unreachable``
    - ``__builtin_unreachable`` function called.
  * - ``pointer_overflow``
    - Overflow in pointer arithmetic.
