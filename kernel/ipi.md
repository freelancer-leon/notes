# IPI

## Check if specific interrupt is enabled ( IPI )
* [Check if specific interrupt is enabled ( IPI )](https://stackoverflow.com/questions/51174097/check-if-specific-interrupt-is-enabled-ipi)
> The term **IPI** designates a *category* of interrupts, specifically: [NMI](https://wiki.osdev.org/Non_Maskable_Interrupt), [SMI](https://wiki.osdev.org/System_Management_Mode), SIPI, INIT and Fixed interrupts.
> Of these, the NMI, SMI, SIPI and INIT cannot be masked without disabling the LAPIC globally.
---
> The [NMI (Non-maskable Interrupt)](https://wiki.osdev.org/Non_Maskable_Interrupt) was designed to be non-maskable from the beginning, as the name implies.
> There is a hack to mask it: setting the `bit7` of port `70h`.
> However, this is a hardware trick, it ties the #NMI (today it is the LINT1 but this is configurable) high.
> The IPIs are sent through a software message so this trick won't work.
> 
> The [SMI (System Management Interrupt)](https://wiki.osdev.org/System_Management_Mode) is used to enter the SMM (see previous link), a mode designed to be as trasparent to > the software as possible.
> It is not maskable, for what the software is concerned, it doesn't exist.
> 
> The [INIT and SIPI (Startup-IPI)](https://wiki.osdev.org/Symmetric_Multiprocessing#Initialisation_of_an_old_SMP_system) interrupts are used to reset and wake up a CPU.
> They cannot be masked by design (the BIOS usually put the APs, Application Processors, to sleep with a `cli / hlt` sequence).
> 
> The Fixed interrupts can be masked with the `IF` flag or when a higher priority fixed interrupt ISR is executing (just like > the legacy IRQs with the PIC).
---
> It's possible to mask only the Fixed interrupts, something that may actually corresponds to the common understanding of the term IPI, by *soft* disabling the APIC.
> This can be done by clearing `bit8` of the *Spurious Interrupt Vector Register* at offset `0f0h` from the LAPIC base (the LAPIC > base is set in the `IA32_APIC_BASE` MSR, its address is 1bh).
> Of course, clearing `IF` will also do.
>
> Alternatively it's possible to disable the LAPIC entirely (it won't respond to any IPI) by clearing bit11 of the `IA32_APIC_BASE` MSR.
---
> To check if the IPIs are enable you have to check the `IF` flag, the *Spurious Interrupt Vector Register* and the `IA32_APIC_BASE` MSR.
