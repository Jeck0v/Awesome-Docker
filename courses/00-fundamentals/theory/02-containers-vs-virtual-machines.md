# Lesson 02 — Containers vs Virtual Machines

> **In one sentence:** A virtual machine pretends to be a *whole computer* with its own operating system, whereas a container is just an *isolated program* that shares the operating system's kernel with its neighbours.

---

## What you will learn

- what a **virtual machine (VM)** is and what a **hypervisor** does;
- the one key difference between a VM and a container: **who owns the kernel**;
- a detailed **comparison table**;
- when to choose a VM, when to choose a container, and why we often use **both**;
- why Docker Desktop on Windows and macOS secretly uses a small VM.

---

## 1. First, two words you need: "operating system" and "kernel"

An **operating system** (Windows, macOS, Linux…) is the big piece of software that makes a computer usable.

At its very centre is the **kernel**. The kernel is the **boss of the computer**. It is the only program allowed to talk directly to the hardware: the processor, the memory, the disk, the network card. Every other program has to *ask the kernel* when it wants something.

```
   +-------------------------------------+
   |  Your programs (browser, game...)   |   "Please, may I write this file?"
   +-------------------------------------+
   |  KERNEL  (the boss)                 |   "Yes / no, and here is how."
   +-------------------------------------+
   |  HARDWARE (CPU, RAM, disk, network) |
   +-------------------------------------+
```

Keep this picture in mind. The whole difference between VMs and containers is about **the kernel**.

---

## 2. What is a virtual machine?

A **virtual machine** is a **pretend computer** that runs inside your real computer.

It has its own pretend processor, pretend memory, pretend disk… and on top of that, its **own complete operating system, with its own kernel**.

The software that creates and manages these pretend computers is called a **hypervisor** (for example VirtualBox, VMware, Hyper-V, KVM).

### The houses analogy

Imagine a large field (your real computer).

- **Virtual machines** are like building **separate houses** on the field. Each house has its **own foundations, own plumbing, own electricity, own boiler, own roof**. Very independent, very solid… but each house is expensive and slow to build.
- **Containers** are like **flats in one block**. All flats share the **same foundations, plumbing and boiler** (the kernel), but each flat has its **own front door, own furniture and own lock**. Much cheaper and faster to set up, and you can fit many more on the same land.

```
           VIRTUAL MACHINES                          CONTAINERS
  +-----------+ +-----------+ +-----------+   +-------+ +-------+ +-------+
  |  App A    | |  App B    | |  App C    |   | App A | | App B | | App C |
  | libraries | | libraries | | libraries |   | libs  | | libs  | | libs  |
  |-----------| |-----------| |-----------|   +-------+ +-------+ +-------+
  | Guest OS  | | Guest OS  | | Guest OS  |   +-------------------------------+
  | + KERNEL  | | + KERNEL  | | + KERNEL  |   |  Container engine (Docker)    |
  +-----------+ +-----------+ +-----------+   +-------------------------------+
  +-------------------------------------+     +-------------------------------+
  |            HYPERVISOR               |     |  Host OS + ONE SHARED KERNEL  |
  +-------------------------------------+     +-------------------------------+
  +-------------------------------------+     +-------------------------------+
  |            HARDWARE                 |     |           HARDWARE            |
  +-------------------------------------+     +-------------------------------+

   3 apps = 3 full operating systems           3 apps = 1 kernel, shared
```

Look at the left side: **three full guest operating systems**, each with its own kernel. On the right side: **one kernel**, shared by everyone.

### Two kinds of hypervisor (for your general knowledge)

| Type | Where it runs | Examples |
|---|---|---|
| **Type 1** ("bare metal") | Directly on the hardware, no normal OS underneath | VMware ESXi, Microsoft Hyper-V, Xen, KVM |
| **Type 2** ("hosted") | As an application on top of your normal OS | VirtualBox, VMware Workstation, Parallels |

This classification is a simplification. KVM is a module inside the Linux kernel, and Hyper-V runs underneath Windows rather than on top of it; both are usually classed as type 1, even though they look like part of an ordinary operating system.

---

## 3. The big comparison table

| Criterion | Virtual machine | Container |
|---|---|---|
| **What is it?** | A complete pretend computer | An isolated process (a program with blinkers) |
| **Analogy** | A separate house with its own foundations | A flat in a shared block |
| **Kernel** | Each VM has **its own kernel** | All containers **share the host's kernel** |
| **What is inside** | A full operating system + libraries + application | Only the application + its libraries (no kernel) |
| **Managed by** | A hypervisor (Hyper-V, VirtualBox, KVM, VMware…) | A container engine (Docker, containerd, Podman…) |
| **Typical size** | Gigabytes (GB) | Depends on the image: from a few MB to several GB. With no guest OS, usually much smaller than a VM |
| **Start-up time** | Tens of seconds to minutes (it has to boot) | Usually under a second (it just starts a process) |
| **Memory use** | High — every VM reserves memory for a whole OS | Low — only what the application really uses |
| **How many on a laptop** | A handful | Dozens, even hundreds |
| **Isolation strength** | Generally stronger — separated at the hardware level | Generally weaker — a kernel flaw can affect every container |
| **Security boundary** | Harder to escape, but not invulnerable; depends on configuration | Depends heavily on configuration: privileges, runtime, attack surface |
| **Different operating systems** | Yes — a Windows VM can run on a Linux host and vice versa | No — Linux containers need a **Linux kernel** |
| **Portability** | Heavy disk images, harder to move | Light images, shared easily through registries |
| **Repeatability** | Possible, but snapshots are large and slow | Excellent — a Dockerfile rebuilds the same image every time |
| **Performance** | Some overhead due to virtualisation | Close to running the program directly for CPU work; filesystem, network or heavy I/O can have noticeable overhead |
| **Typical use** | Running different OSes, strong isolation, legacy systems | Packaging applications, microservices, CI/CD, development environments |
| **Updating** | Log in and patch the OS inside the VM | Build a new image, replace the container |
| **Unit of thinking** | "A server" | "An application" |

---

## 4. Explaining the table with simple examples

### "Start-up time"

- **VM:** like turning on an old desktop computer. You wait for the logo, the loading screen, the login… Easily 30 seconds or more.
- **Container:** like opening an app on your phone. Tap, and it is there.

Why? A VM has to **boot a whole operating system**. A container only **starts a program**, because the kernel is already running.

### "Size"

- A VM with Ubuntu inside: several **gigabytes**.
- A container has no size of its own: it uses an **image**, and image sizes vary a lot. An `alpine` image (a tiny Linux distribution) is about **8 megabytes**; an image with a large application and its libraries can weigh several **gigabytes**. Small size is a property of a well-chosen image, not of containers in general.

### "Different operating systems"

Imagine a block of flats built with **French electrical sockets**. Every flat uses French sockets — you cannot have a flat with British sockets in the same building, because the wiring (the kernel) is shared.

- With **VMs**, each house has its own wiring, so one can be "Windows" and another "Linux".
- With **containers**, every flat uses the host's kernel. A Linux container needs a Linux kernel.

### "Isolation strength"

- **VMs:** separate houses. If a burglar gets into one house, the others are still safe behind their own walls.
- **Containers:** flats in one block. Strong front doors, but if someone breaks the **shared foundations** (a bug in the kernel), every flat may be in danger.

Neither is absolute: a badly configured VM can be broken into, and a well-configured container can be quite safe. The real level of security depends on configuration, privileges, the runtime and how much is exposed. That is why, in later courses, we will learn to make containers safer: not running as `root`, removing unnecessary permissions, and so on.

---

## 5. The secret of Docker Desktop on Windows and macOS

You might be wondering: *"If Linux containers need a Linux kernel, how does Docker work on my Windows PC?"*

Excellent question. The answer: **Docker Desktop quietly runs a small Linux virtual machine** for you.

- On **Windows**, it normally uses **WSL 2** (Windows Subsystem for Linux), which runs a real Linux kernel in a lightweight VM managed by Hyper-V technology.
- On **macOS**, it uses Apple's virtualisation framework to run a small Linux VM.

Your containers run **inside that Linux VM**, sharing **its** kernel.

```
  +---------------------------------------------------------+
  |                 YOUR WINDOWS PC                         |
  |                                                         |
  |   docker CLI  ----talks to---->  +-------------------+  |
  |   (PowerShell)                   | small Linux VM    |  |
  |                                  | (WSL 2)           |  |
  |                                  |  +----+ +----+    |  |
  |                                  |  | C1 | | C2 |    |  |
  |                                  |  +----+ +----+    |  |
  |                                  |  Linux kernel     |  |
  |                                  +-------------------+  |
  +---------------------------------------------------------+
```

> **Key idea:** VMs and containers are not enemies. On your own PC, you are **already using both**: a VM to provide a Linux kernel, and containers inside it.

(Windows also has its own "Windows containers", which share a Windows kernel. They exist, but they are much less common and are outside the scope of this course.)

---

## 6. So which one should I use?

| Choose a **virtual machine** when… | Choose a **container** when… |
|---|---|
| You need a **different operating system** (for example Windows on a Linux server) | You want to **package an application** and run it the same everywhere |
| You need **very strong isolation** (untrusted code, separate customers) | You want things to **start fast** and use **few resources** |
| You must run **old software** that needs a full, specific OS | You build **microservices** (many small cooperating programs) |
| You need to control the **kernel itself** (custom kernel modules, kernel versions) | You want **repeatable** builds, tests and development environments |

### Very often: both

In the real world, cloud providers give you **VMs**, and inside those VMs, you run **containers**. You get the strong walls of the VM *and* the lightness of containers.

```
   Cloud data centre
   +---------------------------------------------------+
   |  VM (customer A)            VM (customer B)       |
   |  +---------------------+    +------------------+  |
   |  | [web] [api] [db]    |    | [shop] [cache]   |  |
   |  |   containers        |    |   containers     |  |
   |  +---------------------+    +------------------+  |
   +---------------------------------------------------+
```

---

## Remember

- The **kernel** is the boss of the computer that talks to the hardware.
- A **VM** carries its **own kernel**; a **container** **shares** the host's kernel.
- Containers are **usually smaller, faster and lighter**; VMs are **generally more isolated** and can run **other operating systems**.
- Docker Desktop on Windows and macOS uses a **small Linux VM** to provide the Linux kernel.
- In real life we often use **containers inside VMs**.

---

## Quick quiz

1. What is the single most important difference between a VM and a container?
2. Why does a container start faster than a VM?
3. Can you run a Linux container directly on a Windows kernel? How does Docker Desktop deal with this?
4. Give one situation where a VM is a better choice than a container.
5. Why is the isolation of containers considered weaker than that of VMs?

<details>
<summary>Show the answers</summary>

1. A VM has **its own kernel**; containers **share the host's kernel**.
2. A VM must boot an entire operating system. A container only starts a process, because the kernel is already running.
3. No. Docker Desktop runs a small **Linux VM** (WSL 2 on Windows) and the containers run inside it.
4. For example: running Windows on a Linux server, running untrusted code from strangers, or needing a specific kernel version.
5. Because all containers share one kernel. A flaw in that kernel could let a container affect the host or other containers.

</details>

---

**Previous lesson:** [01 — Containerisation](./01-containerization.md) · **Next lesson:** [03 — Docker architecture](./03-docker-architecture.md)
