---
layout: post
title:  "Interrupts"
date:   2023-09-01 03:43:00 +0530
---
In event-driven systems, SDKs frequently use terms like "Handlers" or "Loopers". While such anthropomorphic naming might appear intuitive and user-friendly from a system designer's perspective, it has often felt obfuscated to me. Let's take an example of an app initiating a network call. Here is what I have found transpires at the bare metal level.

{% highlight ruby %}
+--------------------------------------+
|              Application             |
|                                      |
|  Main Thread                         |
|   [Handler Creation]                 |
|                                      |
|  Background Thread                   |
|   [Network Request Made]             |
|                                      |
+--------------------------------------+
                |
                V
+--------------------------+   +----------------------------+
|      Network Interface   |   |       Memory/Buffer        |
|          Card            |   |                            |
|    ______                |   | [Interrupt Data Written]   |
|   | Data |               |   |                            |
|   |Packet|---[Packet     |   +----------------------------+
|   |______|    Arrives]   |                ^
|       |                  |                |
|       +---[Hardware      +----------------+
|           Interrupt]     |                |
+--------------------------+                |
                |                           |
                V                           |
+--------------------------+                |
|          Processor       |----------------+
|                          |
|  Cores:    ____     ____ |
|           | C1 |   | C2 ||
|           |____|   |____||
|                          |
|  Kernel `epoll` &        |
|  Scheduler               |
|   [epoll_wait: BLOCKED]  |
|   [Handling Mechanism]   |
+--------------------------+
{% endhighlight ruby %}


The process:

1. The app's background thread sends a network request.
2. It then calls epoll_wait and is put to sleep by the CPU scheduler.
3. The network card, upon receiving a data packet, signals a hardware interrupt to the CPU.
4. The CPU executes an interrupt service routine (ISR), which writes data about the packet to a specific memory location.
5. This change in memory is what "wakes up" the previously sleeping thread (actor is the CPU scheduler), allowing it to process the response.

{% highlight python %}

# MAIN THREAD
handler = createHandler()

# BACKGROUND THREAD/CORE
sendNetworkRequest()
# Wait for network response, thread is BLOCKED here.
response = epoll_wait() # Waits until interrupt data is written to memory.
handleNetworkResponse(response)

# KERNEL LEVEL
def interrupt_service_routine():
    write_data_to_memory()
    wake_epoll()
{% endhighlight python %}
