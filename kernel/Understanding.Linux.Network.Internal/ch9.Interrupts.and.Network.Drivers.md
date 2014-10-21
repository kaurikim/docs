
# CH9: Interrupts and Network Drivers

The previous chapters gave an overview of how the initialization of core
components in the networking code is taken care of. The remainder of the book
offers a feature-by-feature or subsystem-by-subsystem analysis of how networking
is implemented, why features were introduced, and, when meaningful, how they
interact with each other.

This chapter begins an explanation of how packets travel between the L2 or
driver layer and the IP or network layer described indetail in Part V. I’ll be
referring a lot to the data structures introduced in Chapters 2 and 8, so you
should be ready to turn back to those chapters as needed.

Even before the kernel is ready to handle the frame that is coming from or going
to the L2 layer, it must deal with the subtle and complex system of interrupts
set up to make the handling of thousands of frames per second possible. That is
the subject of this chapter.

A couple of other general issues affect the discussion in this chapter:

* When the Linux kernel is compiled with support for symmetric multiprocessing 
(SMP) and runs on a multiprocessor system, the code for receiving and transmitting
packets takes full advantage of that power. The data structures involved are
designed with that goal in mind. In this chapter, we will look at one aspect of
SMP support in particular: the differences between the new softirq queuesand the
old backlog queue.
* When talking about the ingress path, I will cover both the old interface, 
whichis still used by most network drivers, and the new interface, called NAPI, 
which can significantly increase performance under medium to high loads.

In this chapter, you will be given an overview on both bottom half handlers and
kernel synchronization mechanisms. However, for a more detailed discussion, you 
can refer to the other two O’Reilly books, Understanding the Linux Kernel and 
Linux Device Drivers.

## Decisions and Traffic Direction

The paths taken by packets through the network stack differ for received, transmitted,
and forwarded packets (see Figure 9-1). Differences in processing also depend on
the features compiled into the kernel and how they are configured.  Finally, the
devices involved can make a difference because different devices support different
features.

TODO: Figure 9-1. Traffic directions

Virtual devices, such as the familiar loopback interface (lo), tend to use 
shortcuts inside the network stack. These devices are software only. For 
instance, the loopback interface is not associated with any piece of hardware, 
but bonding interfaces are associated indirectly with one or more network cards. 
Some virtual interfaces can therefore dispense with some of the limitations
found with hardware (such as the Maximum Transmission Unit, or MTU) and thus 
speed up performance.

Figure 9-2 gives an idea of the big picture. It is certainly very sketchy; for
instance, it does not show all of the conditions that can lead to dropping a 
frame.* The figure includes extra details about the ingress path; you can find
more detailed graphs about the egress path in Parts V, VI, and VII. We will go 
through all the links that should be part of the graph in the rest of this 
chapter.

## Notifying Drivers When Frames Are Received

In Chapter 5, I mentioned that devices and the kernel can use two main techniques
for exchanging data: polling and interrupts. I also said that a combination of
the two is also a valid option. This section offers a brief overview of the most
common ways for a driver to notify the kernel about the reception of a frame,
along with the main pros and cons for each one. Some approaches depend on the
availability of specific features on the devices (such as ad hoc timers), and some
need changes to the driver, the operating system, or both.

This discussion could theoretically apply to any device type, but it best 
describes those devices like network cards that can generate a high number of 
interactions (that is, the reception of frames).

### Polling

With this technique, the kernel constantly keeps checking whether the device has
anything to say. It can do that by continually reading a memory register on the
device, for instance, or returning to check it when a timer expires. As you can 
imagine, this approach can easily waste quite a lot of system resources, and is
rarely employed if the operating system and device can use other techniques such
as interrupts. Still, there are cases where polling is the best approach. We will
come back to this point later.

### Interrupts

Here the device driver, on behalf of the kernel, instructs the device to generate
a hardware interrupt when specific events occur. The kernel, interrupted from its
other activities, will then invoke a handler registered by the driver to take care
of the device’s needs. When the event is the reception of a frame, the handler
queues the frame somewhere and notifies the kernel about it. This technique, which
is quite common, still represents the best option under low traffic loads. 
Unfortunately, it does not perform well under high traffic loads: forcing an
interrupt for each frame received can easily make the CPU waste all of its time
handling interrupts.


The code that takes care of an input frame is split into two parts: first the
driver copies the frame into an input queue accessible by the kernel, and then
the kernel processes it (usually passing it to a handler dedicated to the
associated protocol such as IP). The first part is executed in interrupt context
and can preempt the execution of the second part. This means that the code that
accepts input frames and copies them into the queue has higher priority than the
code that actually processes the frames.

Under a high traffic load, the interrupt code would keep preempting the processing
code. The consequence is obvious: at some point the input queue will be full, but
since the code that is supposed to dequeue and process those frames does not have
a chance to run due to its lower priority, the system collapses. New frames cannot
be queued since there is no space, and old frames cannot be processed because
there is no CPU available for them. This condition is called receive-livelock in 
the literature.

In summary, this technique has the advantage of very low latency between the 
reception of the frame and its processing, but does not work well under high
loads. Most network drivers use interrupts, and a large section later in this 
chapter will discuss how they work.

### Processing Multiple Frames During an Interrupt

This approach is used by quite a few Linux device drivers. When an interrupt is
notified and the driver handler is executed, the latter keeps downloading frames
and queuing them to the kernel input queue, up to a maximum number of frames (or
a window of time). Of course, it would be possible to keep doing that until the 
queue gets empty, but let’s remember that device drivers should behave as good 
citizens. They have to share the CPU with other subsystems and IRQ lines with
other devices. Polite behavior is especially important because interrupts are
disabled while the driver handler is running.

Storage limitations also apply, as they did in the previous section. Each device
has a limited amount of memory, and therefore the number of frames it can store
is limited. If the driver does not process them in a timely manner, the buffers 
can get full and new frames (or old ones, depending on the driver policies) could 
be dropped. If a loaded device kept processing incoming frames until its queue 
emptied out, this form of starvation could happen to other devices.

This technique does not require any change to the operating system; it is implemented 
entirely within the device driver.

There could be other variations to this approach. Instead of keeping all interrupts
disabled and having the driver queue frames for the kernel to handle, a driver 
could disable interrupts only for a device that has frames in its ingress queue 
and delegate the task of polling the driver’s queue to a kernel handler. This is
exactly what Linux does with its new interface, NAPI. However, unlike the approach
described in this section, NAPI requires changes to the kernel.


### Timer-Driven Interrupts

This technique is an enhancement to the previous ones. Instead of having the device
asynchronously notify the driver about frame receptions, the driver instructs the 
device to generate an interrupt at regular intervals. The handler will then check 
if any frames have arrived since the previous interrupt, and handles all of them 
in one shot. Even better would be to have the driver generate interrupts at intervals, 
but only if it has something to say.

Based on the granularity of the timer (which is implemented in hardware by the device 
itself; it is not a kernel timer), the frames that are received by the device will
experience different levels of latency. For instance, if the device generated an 
interrupt every 100 ms, the notification of the reception of a frame would have an 
average delay of 50 ms and a maximum one of 100 ms. This delay may or may not be 
acceptable depending on the applications running on top of the network connections 
using the device.*

The granularity available to a driver depends on what the device has to offer, since
the timer is implemented in hardware. Only a few devices provide this capability
currently, so this solution is not available for all the drivers in the Linux kernel.
One could simulate that capability by disabling interrupts for the device and using 
a kernel timer instead. However, one would not have the support of the hardware, and 
the CPU cannot spend as much of its resources as the device can on handling timers, 
so one would not be able to schedule the timers nearly as often. This workaround 
would, in the end, become a polling approach.

### Combinations

Each approach described in the previous sections has some advantages and disadvantages. 
Sometimes, it is possible to combine them and obtain something even better. We said
that under low load, the pure interrupt model guarantees a low latency, but that
under high load it performs terribly. On the other hand, the timer-driven interrupt 
may introduce too much latency and waste too much CPU time under low load, but it 
helps a lot in reducing the CPU usage and solving the receive-livelock problem under
high load. A good combination would use the interrupt technique under low load and
switch to the timer-driven interrupt under high load. The tulip driver included in 
the Linux kernel, for instance, can do this (see drivers/net/tulip/interrupt.c*).

### Example

A balanced approach to processing multiple frames is shown in the following piece 
of code, taken from the drivers/net/3c59x.c Ethernet driver. It is a selection of 
key lines from vortex_interrupt, the function registered by the driver as the handler
of interrupts from devices in 3Com’s Vortex family:

```c
static irqreturn_t vortex_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
    int work_done = max_interrupt_work;
    ioaddr = dev->base_addr;
    ... ... ...
    status = inw(ioaddr + EL3_STATUS);
    do {
        ... ... ...
        if (status & RxComplete)
            vortex_rx(dev);
        if (--work_done < 0) {
            /* Disable all pending interrupts. */
            ... ... ...
            /* The timer will re-enable interrupts. */
            mod_timer(&vp->timer, jiffies + 1*HZ);
            break;
        }
        ... ... ...
    } while ((status = inw(ioaddr + EL3_STATUS)) & (IntLatch | RxComplete));
    ... ... ...
}
```

Other drivers that follow the same model will have something very similar. They 
probably will call the EL3_STATUS and RxComplete symbols something different, and 
their implementation of an xxx_rx function may be different, but the skeleton will
be very close to the one shown here.

In vortex_interrupt, the driver reads from the device the reasons for the interrupt
and stores it into status. Network devices can generate an interrupt for different 
reasons, and several reasons can be grouped together in a single interrupt. If RxComplete
(a symbol specially defined by this driver to mean a new frame has been received) 
is among those reasons, the code invokes vortex_rx.* During its execution, interrupts
are disabled for the device. However, the driver can read a hardware register on 
the card and find out if in the meantime, a new interrupt was posted. The IntLatch
flag is true when a new interrupt has been posted (and it is cleared by the driver
when it is done processing it).

vortex_interrupt keeps processing incoming frames as long as the register says there
is an interrupt pending (IntLatch) and that it is due to the reception of a frame
(RxComplete). This also means that only multiple occurrences of RxComplete interrupts 
can be handled in one shot. Other types of interrupts, which are much less frequent, 
can wait.

Finally -- here is where good citizenship enters -- he loop terminates if it reaches 
the maximum number of input frames that can be processed, stored in work_done. This 
driver uses a default value of 32 and allows that value to be tuned at module load 
time.

## Interrupt Handlers

A good deal of the frame handling we discuss in this chapter takes place in response
to interrupts from network hardware. The scheduling of functions triggered by interrupts
is a complicated topic and deserves some study, even though it doesn’t concern networking
in particular. Therefore, in this section, we discuss the various ways that interrupts
are handled by different network drivers and introduce the concepts of bottom halves
and softirqs.

In Chapter 5, we saw how device drivers register their handlers with an IRQ number,
but we did not see how hardware interrupts delegate frame processing to software
interrupt handlers. This section will describe how an interrupt request associated
with the reception of a frame is handled all the way to the point where protocol 
handlers discussed in Chapter 13 receive their packets. We will see the relationship 
between hardware IRQs and software IRQs and why the latter category is needed. We 
will briefly see how interrupts were handled with the old kernels and then compare 
the old approach to the new one introduced with kernel version 2.4. This discussion 
will show the advantages of the new model over the old one, especially in the area 
of performance.

Before launching into softirqs, we need a small introduction to the concept of bottom 
half handlers. However, I will not go into much detail about them because they are 
documented in other resources, notably Understanding the Linux Kernel and Linux Device
Drivers.

### Reasons for Bottom Half Handlers

Whenever a CPU receives an interrupt notification, it invokes the handler associated 
with that interrupt, which is identified by a number. During the handler’s execution 
in which the kernel code is said to be in interrupt context—interrupts are disabled 
for the CPU serving the interrupt. This means that if a CPU is busy serving one interrupt,
it cannot receive other interrupts, whether of the same type or of different types.* 
Nor can the CPU execute any other process: it belongs totally to the interrupt handler 
and cannot be preempted.

In the simplest situation, these are the main events touched off by an interrupt:

1. The device generates an interrupt and the hardware notifies the kernel.
1. If the kernel is not serving another interrupt (and if interrupts are not disabled 
for other reasons) it will see the notification.
1. The kernel disables interrupts for the local CPU and executes the handler associated 
with the interrupt type received.
1. The kernel exits the interrupt handler and reenables interrupts for the local CPU.

In short, interrupt handlers are nonpreemptible and non-reentrant. (A function is 
defined as non-reentrant when it cannot be interrupted by another invocation of 
itself. In the case of interrupt handlers, it simply means that they are executed 
with interrupts disabled.) This design choice helps reduce the likelihood of race 
conditions. However, because the CPU is so limited in what it can do, the nonpreemptible 
design has potentially serious effects on performance by the kernel as well as the 
processes waiting to be served by the CPU.


Therefore, the work done by interrupt handlers should be as quick as possible. The 
amount of processing needed by the interrupt handlers during interrupt context depends 
on the type of event. A keyboard, for instance, may simply send an interrupt every 
time a key is pressed, which requires very little effort to be handled: the handler 
simply needs to store the code of the key somewhere, and run a few times per second 
at most. At other times, the actions required to handle an interrupt are not trivial 
and their executions could require much CPU time. Network devices, for instance, 
have a relatively complex job: they need to allocate a buffer (sk_buff), copy the 
received data into it, initialize a few parameters within the buffer structure 
(protocol) to tell the higher-layer protocol handlers what kind of data is coming 
from the driver, and so on.

Here is where the concept of a bottom half handler comes into play. Even if the action 
triggered by an interrupt needs a lot of CPU time, most of this action can usually 
wait. Interrupts are allowed to preempt the CPU in the first place because if the 
operating system makes the hardware wait too long, it may lose data. This is obviously 
true of realtime streaming data, but also is true of any hardware that has to store 
incoming data in fixed-size buffers. And if the hardware loses data, there is usually 
no way to get it back.

On the other hand, if the kernel or a userspace process has to be delayed or preempted, 
no data will be lost (with the exception of real-time systems, which entail a completely 
different way of handling processes as well as interrupts). In light of these considerations, 
modern interrupt handlers are divided into a top half and a bottom half. The top half 
consists of everything that has to be executed before releasing the CPU, to preserve 
data. The bottom half contains everything that can be done at relaive leisure.

One can define a bottom half as an asynchronous request to execute a particular 
function. Normally, when you want to execute a function, you do not have to request 
anything—you simply invoke it. When an interrupt arrives, you have a lot to do and 
don’t want to do it right away. Thus, you package most of the work into a function 
that you submit as a bottom half.

The following model allows the kernel to keep interrupts disabled for much less time 
than the simple model shown previously:

1. The device signals the CPU to notify it of the interrupt.
1. The CPU executes the associated top half, disabling further interrupt notifications 
until this handler has finished its job.
1. Typically, a top half performs the following:
    a. It saves somewhere in RAM all the information that the kernel will need later 
    to process the interrupt event.
    a. It marks a flag somewhere (or triggers something using another kernel mechanism) 
    to make sure the kernel will know about the interrupt and will use the data saved 
    by the handler to complete the event processing.
    a. Before terminating, it reenables the interrupt notifications for the local 
    CPU.
1. At some later point, when the kernel is free of more pressing matters, it checks 
the flag set by the interrupt handler (signaling the presence of data to be processed) 
and calls the associated bottom half handler. It also clears the flag so that it 
can later recognize when the interrupt handler sets the flag again.

Over time, Linux developers have tried different types of bottom halves, which obey 
different rules. Networking has played a large role in the development of new implementations, 
because of networking’s need for low latency—that is, a minimal amount of time between 
the reception of a frame and its delivery. Low latency is more important for network 
device drivers than for other types of devices because of the high number of tasks 
involved in reception and transmission. As described earlier in the section “Interrupts,” 
it can be disastrous to let a large number of frames build up while waiting to be 
handled. Sound cards are another example of devices requiring fast response.

Bottom Halves Solutions

The kernel provides different mechanism for implementing bottom halves and for deferring
work in general. These mechanisms differ mainly with regard to the following points:

* Running context:
    * Interrupts are seen by the kernel as having a different running context from 
    userspace processes or other kernel code. When the function executed by a bottom 
    half is capable of going to sleep, it is restricted to mechanisms allowed in 
    process context, as opposed to interrupt context.

* Concurrency and locking
    * When a mechanism can take advantage of SMP, this has implications for how 
    serialization is enforced (if necessary) and how locking influences scalability.

In this chapter, we will look only at those mechanisms that do not need a process 
context—namely, softirqs and tasklets. In the next section, we will briefly see their 
implications for concurrency and locking.

When you need to defer the execution of a function that may sleep, you need to use 
a dedicated kernel thread or work queues. A work queue is simply a queue where you 
can queue a request to execute a function, and a kernel thread will take care of 
it. In this case, the function would be executed in the context of a kernel thread, 
and therefore sleeping is allowed. Since the networking code mainly uses softirq 
and tasklets, we will not look at work queues.

### Concurrency and Locking

Before launching into the code that network drivers use to handle bottom halves, we need some background on concurrency, which refers to functions that can interfere with each other either because they are scheduled on different CPUs or because one is suspended by the kernel to run another. Related topics are locks and the disabling of interrupts. (Concurrency is discussed in detail in both Understanding the Linux Kernel and Linux Device Drivers.)


