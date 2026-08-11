# Overview

This directory contains the FreeRTOS Tasking port for the Infineon AURIX™ TC4xx family of MCUs equipped with the TriCore™ core.

## Tool Dependencies

- [AURIX™ Development Studio (ADS)](https://www.infineon.com/design-resources/platforms/aurix-software-tools/aurix-tools/aurix-development-studio) — includes free Tasking compiler and an integrated debugger. Please install via the [Infineon Developer Center Launcher](https://www.infineon.com/cms/en/design-support/tools/utilities/infineon-developer-center-idc-launcher/)

## How To Use This Port

Add `port.c`, `portmacro.h`, and `portmacro_tasking.h` from this directory to the application's FreeRTOS portable layer.

The application must provide a `FreeRTOSConfig.h` that defines the TCxxx-specific options listed below. It must also provide the iLLD trap hook configuration described in the System Call Trap Integration section so `portYIELD()` can reach `vPortSyscallHandler()`.

Supported demos for TC4D7 are available in `/FreeRTOS-Partner-Supported-Demos/AURIX_TC4D7_ADS`.

## Required FreeRTOSConfig.h Options

In addition to the standard FreeRTOS kernel configuration options such as `configCPU_CLOCK_HZ`, `configTICK_RATE_HZ`, `configMAX_PRIORITIES`, `configMINIMAL_STACK_SIZE`, and `configTOTAL_HEAP_SIZE`, an application that uses this port must define the TC4xx-specific options below in `FreeRTOSConfig.h`. The companion demo provides example values for TC4D7 CPU0.

When using ADS/iLLD register symbols in these macros, include the corresponding device register headers before the macro definitions, for example `IfxCpu_reg.h` and `IfxSrc_reg.h`. A `configASSERT()` definition is recommended; the companion demo maps it to the TriCore debug instruction in `DEBUG` builds and to an empty macro otherwise.

| Macro | Purpose |
| --- | --- |
| `configCPU` | Base address of the per-CPU register block that contains the STM used for the tick timer. |
| `configCPU_STM_SRC` | Address of the STM service request register used by the tick interrupt. |
| `configCPU_STM_CLOCK_HZ` | STM input clock frequency used to calculate the tick compare interval. |
| `configCONTEXT_SRC` | Address of the service request register used for software-triggered context switches. |
| `configCPU_NR` | TriCore CPU number used in SRC TOS fields and in the generated interrupt/trap vector table selection. |
| `configVM_NR` | TC4xx virtual-machine number used to select the STM VM register offset and the SRC VM field. |
| `configUSE_VIRTUALIZATION` | Set to `0` for non-virtualized applications; set to non-zero to emit `__vm( configVM_NR )` on the port's interrupt and trap handlers. |
| `configCONTEXT_INTERRUPT_PRIORITY` | Interrupt priority for the context-switch service request. |
| `configTIMER_INTERRUPT_PRIORITY` | Interrupt priority for the STM tick service request (must not be higher than `configCONTEXT_INTERRUPT_PRIORITY`). |
| `configMAX_API_CALL_INTERRUPT_PRIORITY` | CCPN mask threshold used by critical sections and FromISR API masking. |
| `configSYSCALL_CALL_DEPTH` | Call depth used when saving and restoring context from `vPortSyscallYield()`. |
| `configPROVIDE_SYSCALL_TRAP` | Set to `0` when the ADS/iLLD trap table dispatches the syscall trap (see below); set to non-zero to let the port emit its own class 6 trap vector. |

The following TC4xx-specific options are optional or have port-provided defaults:

| Macro | Purpose |
| --- | --- |
| `configCPU_STM_DEBUG` | Optional debug assert for missed STM ticks; undefined or `0` disables the check. |
| `configTICK_STM_DEBUG` | Optional STM debug-control setup during tick timer initialization; undefined or `0` disables it. |
| `configYIELD_SYSCALL_ID` | Syscall ID used by `portYIELD()`; defaults to `0` when not defined. |

## System Call Trap Integration

`portYIELD()` uses the TriCore `syscall` instruction with `configYIELD_SYSCALL_ID` (default `0`). The syscall enters the TriCore system-call trap class (class 6 / SYS); the trap identification number (TIN) is passed to `vPortSyscallHandler()` and is matched against `configYIELD_SYSCALL_ID`.

The TC4xx Tasking port does not emit its own trap-vector entry for `vPortSyscallHandler()`. In an ADS/iLLD project, the startup code sets the Base Trap Vector register (BTV) from the linker-provided `__TRAPTAB_CPUx` symbols, for example `Libraries/Infra/Ssw/TC4xx/Tricore/Ifx_Ssw_Tc0.c` writes `CPU_BTV` from `__TRAPTAB(0)`. The Tasking linker script `Lcf_Tasking_Tricore_Tc.lsl` places the iLLD trap table sections, for example `.text.traptab_cpu0`, at those addresses. The system-call trap table entry in `Libraries/iLLD/TC4xx/Tricore/Cpu/Trap/IfxCpu_Trap.c` dispatches class 6 / SYS traps through the `IFX_CFG_CPU_TRAP_SYSCALL_CPUx_HOOK()` macros, but only when `IFX_CFG_EXTEND_TRAP_HOOKS` is defined, otherwise `IfxCpu_Trap.c` falls back to a stub that ignores the trap.

For a CPU0-only FreeRTOS integration using the ADS/iLLD trap handler, enable the trap hook extension in `Configurations/Ifx_Cfg.h`:

```c
#define IFX_CFG_EXTEND_TRAP_HOOKS
```

The `Configurations` files used by the companion demo can be found under `/FreeRTOS-Partner-Supported-Demos/AURIX_TC4D7_ADS/Configurations`.

Then provide the CPU0 syscall hook in `Configurations/Ifx_Cfg_Trap.h`:

```c
extern int vPortSyscallHandler( unsigned char id );
#define IFX_CFG_CPU_TRAP_SYSCALL_CPU0_HOOK( t )    vPortSyscallHandler( t.tId )
```

## Test Coverage

This port has been verified against the qualification checklist in the [FreeRTOS Third-Party Template README](https://github.com/FreeRTOS/FreeRTOS/blob/main/FreeRTOS/Demo/ThirdParty/Template/README.md). All required tests execute continuously without reporting an error on TC4D7 CPU0.

## Support

- For support queries, please open an issue on the [FreeRTOS-Kernel-Partner-Supported-Ports](https://github.com/FreeRTOS/FreeRTOS-Kernel-Partner-Supported-Ports/issues) repository
