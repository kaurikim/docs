
# CH5 Network Device Initializaiton

The flexibility of modern operating systems introduces complexity into 
initialization. First, a device driver can be loaded as either a module or 
a static component of the kernel. Furthermore, devices can be present at boot 
time or inserted (and removed) at runtime: the latter type of device, called 
a hot-pluggable device, includes USB, PCI CardBus, IEEE 1394 (also called 
FireWire by Apple), and others. We'll see how hot-plugging affects what happens
in both the kernel and the user space.

In this first chapter, we will cover:

* A piece of the core networking code initialization.
* The initialization of an NIC.
* How an NIC uses interrupts, and how IRQ handlers can be allocated and 
released. We will also look at how drivers can share IRQs.
* How the user can provide configuration parameters to device drivers loaded 
as modules.
* Interaction between user space and kernel during device initialization and 
configuration. We will look at how the kernel can run a user-space helper to 
either load the correct device driver for an NIC or apply a user-space 
configuration. In particular, we will look at the Hotplug feature.
* How virtual devices differ from real ones with regard to configuration and[...]’


/* 생략 */

## Hardware Interrupts

You do not need to know the low-level background about how hardware interrupts
are handled. However, there are details worth mentioning because they can make 
it easier to understand how NIC device drivers are written, and therefore how 
they interact with the upper networking layers.

Every interrupt runs a function called an interrupt handler, which must be
tailored to the device and therefore is installed by the device driver. 
Typically, when a device driver registers an NIC, it requests and assigns an 
IRQ. It then registers and (if the driver is unloaded) unregisters a handler 
for a given IRQ with the following two architecture-dependent functions. They 
are defined in kernel/irq/manage.c and are overridden by architecture-specific
functions in arch/ XXX /kernel/irq.c, where XXX is the architecture-specific
directory:

```c
int request_irq(unsigned int irq, void (*handler)(int, void*, struct pt_regs*), 
        unsigned long irqflags, const char * devname, void *dev_id)
void free_irq(unsigned_int irq, void *dev_id)
```

When the kernel receives an interrupt notification, it uses the IRQ number to
find out the driver’s handler and then executes this handler. To find handlers,
the kernel stores the associations between IRQ numbers and function handlers
in a global table. The association can be either one-to-one or one-to-many,
because the Linux kernel allows multiple devices to use the same IRQ, a feature
described in the later section “Interrupt sharing.”

In the following sections, you will see common examples of the information
exchanged between devices and drivers by means of interrupts, and how an IRQ
can be shared by multiple devices under some conditions.

### Interrupt Type

With an interrupt, an NIC can tell its driver several different things. Among
them are:

Reception of a frame:
    This is the most common and standard situation.

Transmission failure:
    This kind of notification is generated on Ethernet devices only after a
    feature called exponential binary backoff has failed (this feature is
    implemented at the hardware level by the NIC). Note that the driver will
    not relay this notification to higher network layers; they will come to
    know about the failure by other means (timer timeouts, negative ACKs, etc.).

DMA transfer has completed successfully:
    Given a frame to send, the buffer that holds it is released by the driver
    once the frame has been uploaded into the NIC’s memory for transmission on
    the medium. With synchronous transmissions (no DMA), the driver knows right
    away when the frame has been uploaded on the NIC. But with DMA, which uses
    asynchronous transmissions, the device driver needs to wait for an explicit
    interrupt from the NIC. You can find an example of each case at points where
    dev_ kfree_skb* is called within the driver code drivers/net/3c59x.c (DMA)
    and drivers/ net/3c509.c (non-DMA).

Device has enough memory to handle a new transmission:
    It is common for an NIC device driver to disable transmissions by stopping
    the egress queue when that queue does not have sufficient free space to hold
    a frame of maximum size (e.g., 1,536 bytes for an Ethernet NIC). The queue
    is then reenabled when memory becomes available. The rest of this section
    goes into this case in more detail.


The final case in the previous list covers a sophisticated way of throttling
transmissions in a manner that can improve efficiency if done properly. In this
system, a device driver disables transmissions for lack of queuing space, asks
the NIC to issue an interrupt when the available memory is bigger than a given
amount (typically the device’s Maximum Transmission Unit, or MTU), and then 
reenables transmissions when the interrupt comes.


A device driver can also disable the egress queue before a transmission (to 
prevent the kernel from generating another transmission request on the device),
and reenable it only if there is enough free memory on the NIC; if not, the
device asks for an interrupt that allows it to resume transmission at a later
time. Here is an example of this logic, taken from the el3_start_xmit routine,
which the drivers/net/3c509.c driver installs as its hard_start_xmit† function
in its net_device structure:


```c
static int
el3_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    ... ... ...
    netif_stop_queue (dev);
    ... ... ...
    if (inw(ioaddr + TX_FREE) > 1536)
        netif_start_queue(dev);
    else
        outw(SetTxThreshold + 1536, ioaddr + EL3_CMD);
    ... ... ... 
}
```
