* User-space is called syscall by "name" but later it is referred by "number" from "sys_call_table"
* User-space app cannot call kernel code directly: it cannot make function call to a method existing in kernel-space
* Kernel exists in protected memory space
* User-space signal to the kernel that it wants to execute syscall - software interrupt; trigger exception and system will switch to kernel mode and execute exception handler
* Exception handler is syscall handler
* User-space causes exception or trap to enter the kernel. In response kernel will switch in kernel mode and execute exception handler which is function named "system_call()". In newer systems, the entrance is done via "sysenter" - to provide faster access
* All system call are entering the kernel in the same manner hence systemcall number must be passed to the kernel
* x86 syscall is given to the kernel via "eax" register
* Before triggering exception, user-space app puts in "eax" register the number of desired syscall. Syscall handler then reads the number from "eax" register
* system_call() validates the syscall number against NR_syscalls (int range), if >= NR_syscalls then returned -ENOSYS.
* system_call() calls sys_call_table(, %rax,8) - gets the syscall from table
* Syscall parameters can be passed via registers. In case of a lot of parameters are passed then single register holds the pointer to user-space where all parameters are stored
* Syscall return value is passed back via register as well (x86: "eax")
* Syscall parameters must be validated (must not read the memory which is not belonging to process)
* Kernel should never follow the pointer which is specified in user-space
* C lib is syscall interface - to execute something on users behalf
* Kernel is in "process context" if app executing the syscall in kernel space
* Kernel in "process context" during syscall execution is capable of sleeping or preemptive hence majority of kernel functionality is available 
* After syscall is completed, control is given back to "system_call()" which ultimately will continue process execution
* Adding new syscall: add to system_call_table for each arch where it is supported; update "asm/unistd.h"; compile the code into kernel image 
* glibc (C lib) is just wrapper for support of syscall. If newly syscall is appeared then it needs to be added to glibc
* If syscall is not part of glibc then syscall can be called with marco "_syscalln()" when "n" is the 0<n<=6 - number of parameters to push to registers
** syscall open: long open(const char *filename, int flags, int mode) -> "#define __NR_open 5 _syscall3(long, open, const char*, filename, int, flags, int, mode)" - 3 is parameters; 5 is number of system call. Push syscall number and parameters into registers and issue software interrupt__
* Syscall is not easy to change without breaking user-space apps
* Linked list is not suited for random access operation; suited for dynamic addition and removal of elements
* Producer and consumer pattern is implemented via "queue"
* Maps is "associative array" which is unique keys where each key associated with value. Relationship between key and value is called mapping
* Hash table can be a "map"
* Hash has good average-case asymptotic complexity
* Binary tree has good worst-case behavior 
* Binary tree - each node has 0,1,2 children
* BST (binary search tree) - specific ordering tree - left nodes values are less than root, right nodes value are greater than root
* Red-black tree is self-balancing binary search tree (BST); self-balancing BST is tree which attempts to balance to have the depth of all leaves differs by no more than 1
* Do not optimize for the level of scalability you will never need to support
* Processors can give orders faster than hardware can handle it hence not ideal for kernel to issue the request and wait for response (sync call)
* Kernel must work in async more: dispatch the work and switch to other task and then collect the outcome of dispatched work later
* Kernel must dispatch the work to hardware and switch to other work and dealing with hardware when only hardware is completed the work (calls: interrupt = async)
* When hardware completed the request it will signal back to kernel that "I'm ready - please collect the result" by issuing interrupt
* Kernel handles the interrupt from hardware with interrupt handlers
* When keyboard is pressed -> keyboard controller captures the press and signals (electric signal = interrupt) the CPU about it -> CPU notifies kernel via interrupt that keyboard was pressed -> kernel handles the interrupt with interrupt handler
* An interrupt is physically produced signal by hardware controller which delivered to interrupt controller pin (particular pin assigned to this hardware device), then interrupt controller multiplex the lines into single line to process. After interrupt controller received interrupt, it (interrupt controller) sends it to processor. Processor (CPU) detects the interrupt and make decision to notify or not the kernel, kernel receives the interrupt and invokes the appropriate interrupt handler
* Each device has it unique interrupt number. It (unique interrupt number) enables the kernel to differentiate "who" caused the interrupt and invoke appropriate interrupt handler
* Unique interrupt number is called IRQ line - interrupt request line. Main interrupt has assigned numbers, PC bus has dynamic interrupt numbers
* Kernel know the device by interrupt number (IRQ line)
* Exceptions occurs synchronously, exceptions are generated by processor during execution of instructions in response to programming error (divide by 0) or abnormal condition which requires attention by kernel (page fault)
* Exceptions are synchronous interrupt. In many archs, exception and interrupt are handled similar by processor as well as by kernel hence exception/interrupt handling is handled in same way in kernel
* Kernel in response to the interrupt executes: interrupt handler or interrupt service routine (ISR). Each device generate the interrupt which is associated with specific interrupt handler
* ISR is the C function which has standard way of passing info
* ISR is executed in the special context: "interrupt context"/"atomic context"
* Code executing during ISR which is "interrupt context"/"atomic context" is not possible to block
* It is imperative that interrupt handler executes ASAP to return the execution to interrupted process
* Job of ISR is to ack and perform some job (ex: network interrupt handler will need to copy the packets from hardware to the memory with 10-gigabit cards - this is a lot of work)
* ISR aim is conflicting: do as quickly as possible and do a lot of work. To achieve this interrupt handler split in two parts (halves) - "top half" and "bottom half"
* "Top half": Time-critical part runs immediately after invocation of interrupt handler (ack the receipt of interrupt or resetting the hardware)
* "Bottom half": Runs in future
* Example of "top half" and "bottom half": Once packets are delivered to the network card, network cards generates physical signal (interrupt) from card controller to the interrupt controller, then interrupt controller delivers the interrupt to the processor. Processor detects the interrupt from the network card and notifies the kernel about the interrupt from network card. In response to interrupt, kernel invokes IRQ (interrupt handler) for network card which is part of device driver codebase. Packets are copied from network card to memory since network card limited queue and these must be copied before bufferover flow. Then kernel returns the previous executing instructions. At this point "top half" is completed. Next: processing and handling of packets are executed later during "bottom half"
* Each IRQ registered with kernel by number, interrupt handler (ISR) is part of device driver and registered with "request_irq"
* Interrupt handlers (ISR) are the responsibility of the device driver. Each device registers the ISR for the device. It is done via declaring the "int request_irq(<irq>, <handler>, <flags>, <name>, <dev>)" where <irq> the number of the interrupt line; <handler> the handler (function) to be executed upon receiving this interrupt; <flags> the flags to be set; <name> the name of the interrupt in ASCII (/proc/irq); <dev> used for shared interrupt lines
* "int request_irq(...)" flags can be: IRQF_DISABLED - if set then during execution of interrupt handler (function), it is not possible to break it;  IRQF_SAMPLE_RANDOM - if this interrupt can be used for entropy pool for truly random numbers; IRQF_TIMER - if this is handler processes interrupt for system timer; IRQF_SHARED - if interrupt can be shared among other interrupts
* Interrupt handler (ISR) must be registered ("int  request_irq") before device is fully initialized
* When device driver is unloaded, interrupt handler must be disabled ("free_irq()")
* Role of interrupt handler depends entirely on the device
* ISR needs to minimum ack the interrupt; more complex logic would be do <action> and push extended work to "bottom half"
* Interrupt handler during execution disables the corresponding interrupt line hence other interrupt from the same device cannot happen, but other interrupts are served. Hence nested interrupt handling are not supported
* Shared handlers has IRQF_SHARED as flag set; "dev" must be unique
* Shared interrupt handler (ISR) must be able to distinguish the device by "dev"
* Kernel upon receiving the interrupt sequentially invokes all registered handlers on the line. Interrupt handler must be capable of distinguishing the device generated interrupt. If not - then quickly exist
* Interrupt context is not associated with the process
* Interrupt handler can interrupt another interrupt handler on the different line
* Interrupt context has no associated "current" marco. Since there is no association then interrupt context cannot sleep (how you can map it back after sleep?)
* Interrupt context cannot call certain functions (like: sleep)
* Interrupt context is time-critical and must not have heavy loops; must be as simple as possible
* In interrupt context as much work must be pushed in the bottom half which runs at a more convenient time
* Interrupt handler stack is configurable option: 16K on 64-bit, 8K on 32-bit
* Device issues the interrupt by sending electric signal to the interrupt controller. Interrupt controller sends the interrupt to the processor by utilizing special pin. Processor stops whatever it is doing, disables the interrupt system, jumps to predefined location in the memory and executes the code. The predefined location is set by kernel and this is the entry point for interrupt handlers
* Interrupt handler starts execution in kernel with predefined entry point
* For each interrupt handler processor jumps to unique location in memory and executes code located there. Hence kernel knows the IRQ number of incoming interrupt
* Upon the interrupt entry point, current register values of interrupted task are stored on the stack. Then kernel calls do_IRQ()
* /proc is procfs virtual filesystem which exists in kernerl memory and shows {interrupt line, number of interrupts since boot, interrupt controller handling the interrupt, device associated with interrupt (who causes the interrupt)
  * Bottom half job is to the interrupt-related work which was not done by top half
