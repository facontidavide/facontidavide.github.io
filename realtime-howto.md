# WAIT!!

Stop right now and watch this video: [Introduction to Realtime Linux](https://www.youtube.com/watch?v=BKkX9WASfpI).

Once you did, continue reading...

# Introduction

In terms of real-time linux, you might now that the two most common options
are often [Xenomai](https://xenomai.org/) and [RT-Preempt](https://rt.wiki.kernel.org/index.php/Main_Page)

I will not go into the technical details and the actual differences between these
two approaches, but it is worth mentioning that:

1. Xenomai has its own API and IPC mechanism, RT-Preempt instead use mostly the Posix API.

2. With Xenomai you can achieve latencies up to 20-30 usec, whilst RT-PREEMPT
would usually achieve 60-90 usec if well configured.

3. More and more code or RT-Reempt was moved into mailine linux in the last years, 
this means that it is easier to use in recent kernels.

4. X86 is very well supported by any Linux Kernel. ARM platforms instead very often use
custom linux kernels with mmultiple patches. These patches might collide with the RT extensions.

5. A rule of thumb is that you can have a reliable control loop if it's latency doesn't exceed
10% of the loop period.

6. Ask yourself: *"What would happen in I miss a deadline in my control loop once every minute / hour / day"?*


For these reasons, if you control loop doesn't exceed 1000 Hz, RT-Preempt will 
work just fine.

You might tempted to say "I want the best, let's use Xenomai". **Don't**.

Write your application using RT-Preempt and switch to Xenomai only if you detect
serious issues directly related to latencies.

# How to get started with RT-Preempt

The best place to learn how to build and use RT-Preempt is [this](https://rt.wiki.kernel.org/index.php/RT_PREEMPT_HOWTO)

If you are using Ubuntu, [this project](https://bitbucket.org/thismaechler/ubuntustudio-14.04-realtimeaudio/src)
 is also very usefull.
 
## My howto for Ubuntu 14.04

Install required packages

    sudo apt-get update
    sudo apt-get dist-upgrade
    sudo apt-get install kernel-package fakeroot build-essential libncurses5-dev \
                          wget bzip2 xz-utils libssl-dev bc git libqt4-dev pkg-config

Get sources. Chose a kernel number and a [RT Patch here](https://www.kernel.org/pub/linux/kernel/projects/rt/).
For example:

    wget -c https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.47.tar.gz
    wget -c https://www.kernel.org/pub/linux/kernel/projects/rt/4.4/patch-4.4.47-rt59.patch.xz

Extract and apply the patch:

    tar xvf linux-4.4.47.tar.gz 
    cd linux-4.4.47/
    xzcat ../patch-4.4.47-rt59.patch.xz | patch -p1

Recicle the kenerl configuration on your system and cross your fingers, hoping that it will work:


    cp /boot/config-$(uname -r) .config
    make xconfig

In the GUI, use CTRL-F to find the word "preempt" and check the option **Fully Preemptible Kernel (RT)**

Be sure also that **High Resolution Timer** is enabled.

To build the kernel:

    make-kpkg clean
    fakeroot make-kpkg --initrd --append-to-version=-patched kernel-image kernel-headers -j $(getconf _NPROCESSORS_ONLN)
    
If you get any error at the end pf the process (during the creation of the debian files), you might want
 to follow [these instructions](https://bugs.launchpad.net/ubuntu/+source/kernel-package/+bug/1308183).
 
 In short, you should install 
 
 [kernel-package_13.003_all.deb](https://launchpad.net/ubuntu/+source/kernel-package/13.003/+build/5980712/+files/kernel-package_13.003_all.deb)


Finally, you can install the kernel using the commands:

    sudo dpkg -i ../linux-{headers,image}-4.4.47-rt59*.deb

## First steps and Hello World

A good way to start understanding how to write a real-time process/thread is to real this
[example](https://rt.wiki.kernel.org/index.php/RT_PREEMPT_HOWTO#A_Realtime_.22Hello_World.22_Example)

I also suggest to read the source code of [Cyclictest](https://rt.wiki.kernel.org/index.php/Cyclictest).

## Tip and tricks

1. Remember that context switching is a relatively slow operation. Don't use multiple threads if you can serialize
your operations on a single one.

* You assign RT priority to __threads__ not processes. This means that in a single process
normal and RT threads can coexist.

* RT threads shall __not allocate dynamic memory__, if not during the initialization. 

   * If you really need to use dynamic data structure, consider the use of a memory pool.
   * Prefer data structures like std::vector and boost::circular_buffer.
   * std::string allocates memory. So does std::iostream.
   * Use **Valgrind** or other tools such as **MemTrack** or **libcwd** to check that you are not allocating memory.
   
* Avoid algorithm that use iterative algorithms that are not time bounded. An example? Inverse-kinematic
 using the pseudo inverse of a jacobian.
 
   * If you use them, always use a timeout to be sure that you won't use too much time budget.
   
* Optional: learn the main principles of **cache friendly programming**.

* There are stacks of the OS that are inerently not RT, like USB. In theory even the Ethernet
task is not, but you will get good determinism if you **avoid using a switch or router**.


* Whenever your application need to do non-deterministic and non-RT tasks, "delegate" the action to another
non-RT thread. Take a look to the "asynchronous execution" design patters.

* The best inter-thread communication are circular buffers (protected by mutexes) or the 
data structures in [boost::lockfree](http://www.boost.org/doc/libs/1_63_0/doc/html/lockfree.html).

   * In general, even if mutexes *should* be safe on a RT-OS (take few minutes to read about "priority inversion"),
    prefer lockfree data structures if you can.
    
   
   

