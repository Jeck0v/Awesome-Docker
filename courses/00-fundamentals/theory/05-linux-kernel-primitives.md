# Lesson 05 — Linux Kernel Primitives

> **In one sentence:** Docker does not create containers by magic — it asks the **Linux kernel** to do it, using a handful of built-in features (namespaces, cgroups, layered filesystems and security filters). Understand those features and you understand Docker.

---

## What you will learn

- why the **Linux kernel** is the real engine behind every container;
- the four families of kernel features ("primitives") that make a container:
  1. **namespaces** — what a container can **see**;
  2. **cgroups** — what a container can **use**;
  3. **layered filesystems (OverlayFS)** — what a container's **files** look like;
  4. **security features** (users, capabilities, seccomp, AppArmor/SELinux) — what a container is **allowed to do**;
- how these features shape **the logic of Docker** (why things behave the way they do);
- why **learning Linux** is the key to understanding the advanced parts of Docker.

> A "primitive" is a basic building block, like a single LEGO brick. Docker is the instruction booklet that assembles the bricks; the bricks themselves belong to Linux.

---

## 1. The big reveal: "a container" does not exist

This is the most important sentence of the whole fundamentals course:

> **There is no such thing as a "container" inside the Linux kernel.**

The kernel has no "container" object. What it has is a set of separate features. When Docker "creates a container", it really does this:

1. start a **normal Linux process** (your program);
2. ask the kernel to give that process **its own private view** of the system (namespaces);
3. ask the kernel to put that process on a **budget** (cgroups);
4. give it a **root folder** built from stacked image layers (OverlayFS);
5. **remove dangerous powers** from it (capabilities, seccomp, AppArmor/SELinux).

The result *looks* like a separate little machine. But underneath, it is **just a process with blinkers, a budget and a reduced set of keys**.

```
   What you think you have:            What you really have:

   +-------------------------+         A normal Linux process...
   |   "a little computer"   |           + namespaces   (blinkers:  what it SEES)
   |   with its own OS       |           + cgroups      (budget:    what it USES)
   +-------------------------+           + OverlayFS    (files:     what it STANDS ON)
                                         + capabilities / seccomp / LSM
                                                        (keys:      what it is ALLOWED to do)
```

Remember the team from [Lesson 04](./04-docker-engine.md)? **runc** is the worker that makes these requests to the kernel. Docker, containerd and runc **organise**; the **kernel does the actual isolating**.

---

## 2. A quick refresher: the kernel and system calls

The **kernel** is the boss of the computer (see [Lesson 02](./02-containers-vs-virtual-machines.md)). Programs cannot touch the hardware directly. They must ask the kernel, using **system calls** ("syscalls") — a bit like filling in an official request form:

- `open` — "please open this file";
- `write` — "please write these bytes";
- `clone` — "please create a new process";
- `mount` — "please attach this filesystem here";
- …and a few hundred more.

Because **every** container's request goes to the **same kernel**, the kernel is in the perfect position to decide **what each container may see, use and do**. That is exactly what the primitives below are for.

---

## 3. Namespaces — what a container can SEE

### The idea

A **namespace** gives a process its **own private version** of a part of the system. Inside it, the process believes it is alone.

**Analogy: virtual-reality headsets.** Imagine five children in the same living room, each wearing a VR headset. Each one sees a different world: one is on a beach, one is in space, one is in a castle. They are physically in the same room (the same computer), but **what each one sees** is completely different.

### The different namespaces

Linux has several kinds of namespace. Each one hides a different "part of the room".

| Namespace | What it isolates | Simple example inside a container |
|---|---|---|
| **PID** (process IDs) | The list of running programs | Your program thinks it is **process number 1**, the very first program. It cannot see the host's other programs. |
| **NET** (network) | Network cards, IP addresses, ports, routing | The container has its **own IP address**, and its **own port 80**, so ten containers can all use port 80 without clashing. |
| **MNT** (mount) | The list of mounted filesystems | The container sees **its own root folder `/`**, not your computer's files. |
| **UTS** (host and domain name) | The machine's name | The container has **its own hostname** (by default, a short random-looking ID). |
| **IPC** (inter-process communication) | Shared memory and message queues | Programs in different containers cannot secretly share memory. |
| **USER** (user IDs) | Who is "root" and who is not | "root" inside the container can be mapped to a **harmless normal user** on the host. (Not enabled by default in Docker — see section 6.) |
| **CGROUP** | The view of the cgroup tree | The container only sees its own resource budget, not everyone else's. |
| **TIME** | Some system clocks | Exists in recent kernels; Docker does not use it by default. |

### What it looks like

```
   HOST view (sees everything)                 CONTAINER view (PID namespace)

   PID 1     systemd                            PID 1   nginx: master   <- thinks it is first!
   PID 812   sshd                               PID 7   nginx: worker
   PID 4021  containerd-shim
   PID 4038  nginx: master   <---- same process, two different numbers ---->
   PID 4062  nginx: worker
```

The **same** nginx process has **two numbers**: 4038 for the host, 1 inside the container. It is not a copy. It is one process, seen through two different "headsets".

### Why it matters for Docker's logic

- **PID 1 is special in Linux.** The first process has extra responsibilities: it receives stop signals and must clean up finished child processes. Inside a container, *your* program becomes PID 1, often without being designed for it. This is why some containers take 10 seconds to stop (see section 8).
- **Network namespaces** explain why you must **publish ports** (`-p`) to reach a container from your browser: the container's port 80 lives in *its* network world, not yours.
- **Mount namespaces** explain why a container cannot see your files unless you explicitly **mount** them in.

---

## 4. Cgroups — what a container can USE

### The idea

**Control groups** ("cgroups") put a group of processes on a **budget**: how much **memory**, how much **processor time**, how many **processes**, how much **disk input/output** they may use. They also **measure** what is used (this is how `docker stats` gets its numbers).

**Analogy: pocket money.** Each child gets a fixed amount of pocket money per week. They can spend it however they like, but they cannot spend their sibling's money, and they cannot spend more than they have. Without pocket money rules, one greedy child could empty the family's bank account.

### Main resources controlled

| Resource | What happens at the limit | Docker option (examples) |
|---|---|---|
| **Memory** | The kernel first tries to reclaim memory (and may use swap); if that is not enough, the **OOM killer** ("Out Of Memory") **kills** a process | `--memory` |
| **CPU** | The process is **slowed down** (it waits its turn), not killed | `--cpus`, `--cpu-shares` |
| **Number of processes** | New processes cannot be created | `--pids-limit` |
| **Disk input/output** | Reading/writing is slowed down | `--device-read-bps`, `--blkio-weight` |

Notice the difference: running out of **CPU** makes you **slow**; running out of **memory** can get you **killed**. That is a kernel rule, not a Docker rule. It explains a common surprise: a container that suddenly stops with **exit code 137** (= 128 + signal 9, `SIGKILL`). Be careful, though: 137 only means the process received `SIGKILL`. The OOM killer is one possible cause; `docker kill` or a stop timeout are others. Docker records which one it was (the `OOMKilled` field of `docker inspect`).

### Why it matters

- **Without limits, a container can use all of the host's memory and CPU** — by default, Docker sets **no** memory or CPU limit. One buggy container can slow down or crash everything else on the machine.
- A program that creates processes endlessly (a "fork bomb") can freeze a machine — unless a PID limit is set.
- Modern systems use **cgroups v2**. `docker info` shows `Cgroup Version: 2`. Docker Desktop uses version 2 as well.

### See it yourself

Inside a container, the cgroup budget is visible as ordinary files. For example, with cgroups v2:

```bash
docker run --rm --memory 50m alpine cat /sys/fs/cgroup/memory.max
# 52428800      <- 50 MiB, written in bytes, enforced by the kernel
```

Docker simply **wrote a number into a kernel file**. The kernel does the rest.

---

## 5. Layered filesystems (OverlayFS) — what a container STANDS ON

### The idea

In [Lesson 01](./01-containerization.md) you learnt that an image is a stack of **read-only layers** with one **writable layer** on top for each container. The Linux feature that makes this stack look like **one normal folder** is called **OverlayFS** (used by Docker's `overlay2` driver).

**Analogy: tracing paper.** Several sheets of tracing paper on top of one another. Looking down from above, you see one complete drawing. You may only draw on the **top** sheet. If you want to "change" something drawn on a lower sheet, you redraw it on the top sheet so it hides the old version.

```
   merged view   ->  what the container sees as "/"     (looks like one folder)
        ^
   upperdir      ->  writable layer   (this container only)
   lowerdir(s)   ->  image layers     (read-only, shared)
```

- **Reading** a file: the kernel looks from the top down and shows the first version it finds.
- **Changing** a file from a lower layer: the kernel first **copies it up** into the writable layer (*copy-on-write*), then changes the copy.
- **Deleting** a file from a lower layer: the kernel cannot erase a read-only layer, so it places a "**whiteout**" marker on top that hides it.

### Then the container is "locked" into that folder

Using the mount namespace and a system call called `pivot_root` (a more secure relative of the old `chroot`), runc makes the merged folder become the container's `/`. From inside, there is no way to "walk up" into the host's files.

### Why it matters for Docker's logic

- This is why **images are small to share and fast to start**: layers are reused, nothing is copied until it changes.
- This is why **deleting a file in a later Dockerfile step does not make the image smaller**: the file still exists in the lower layer; it is only hidden by a whiteout.
- This is why **writing a lot of data inside a container is slow and temporary**: copy-on-write has a cost, and the writable layer disappears with the container. Heavy or precious data belongs in **volumes**, which bypass OverlayFS.

---

## 6. Security features — what a container is ALLOWED to do

Namespaces hide things; cgroups limit amounts. But a process could still **ask the kernel for dangerous things**. Linux has several extra locks.

### 6.1 Users and UIDs

Linux does not really know user *names*; it knows **numbers** (UIDs). `root` is UID **0**.

> **Important:** by default, Docker does **not** use the USER namespace. So **root inside a container is UID 0 on the host too.** That does not mean it automatically has every root power on the host: capabilities, seccomp, AppArmor/SELinux and the namespaces limit what it can do (see below). But it is still UID 0, so any gap in those protections is more serious.

That is why a common best practice is to run containers as a **non-root user**, and why options such as **rootless Docker** or **user namespace remapping** exist.

This also explains the famous "**Permission denied**" problem with mounted folders: the container's user and the host's files are compared **by number**, not by name.

### 6.2 Capabilities — a keyring instead of a master key

Traditionally, root has **every power**. Linux splits those powers into about forty separate **capabilities**, such as:

| Capability | Power |
|---|---|
| `CAP_NET_BIND_SERVICE` | Listen on ports below 1024 |
| `CAP_CHOWN` | Change who owns a file |
| `CAP_NET_ADMIN` | Change network settings |
| `CAP_SYS_ADMIN` | A huge collection of powerful operations (almost "full root") |
| `CAP_SYS_MODULE` | Load code into the kernel |

**Analogy:** instead of giving the caretaker the **master key** to the whole building, you give them a **small keyring** with only the keys they need.

Docker starts containers with a **reduced keyring** by default: it keeps a small set of harmless capabilities and removes dangerous ones like `CAP_SYS_ADMIN` and `CAP_SYS_MODULE`. You can remove even more (`--cap-drop`) or add some back (`--cap-add`). The option `--privileged` gives the container **almost all** the keys — which is why it is considered dangerous.

### 6.3 Seccomp — a list of forbidden requests

**Seccomp** ("secure computing") filters **system calls**. Docker applies a default profile that **blocks dozens of risky syscalls** (for example rebooting the machine or loading a new kernel).

**Analogy:** the official request form (syscall) has some boxes that are greyed out. The kernel refuses them, whoever asks.

### 6.4 AppArmor and SELinux

These are **Linux Security Modules (LSM)**: extra rule books that say which files and resources a program may touch. Ubuntu/Debian usually use **AppArmor**; Red Hat/Fedora usually use **SELinux**. Docker applies a default profile when they are available.

### Layers of protection

```
   Your program
      | wants to do something
      v
   [ namespace ]   can it even SEE the target?
   [ capability ]  does it have the right KEY?
   [ seccomp    ]  is this kind of REQUEST allowed?
   [ AppArmor / SELinux ] does the RULE BOOK allow it?
   [ cgroups    ]  is there enough BUDGET left?
      |
      v
   Linux kernel does it (or refuses)
```

---

## 7. Networking primitives — how containers talk

Networking is also built with ordinary Linux features:

| Linux feature | Analogy | Role in Docker |
|---|---|---|
| **Network namespace** | A separate house with its own telephone line | Each container gets its own network stack |
| **veth pair** (virtual Ethernet) | A cable with two ends | One end inside the container (`eth0`), the other on the host |
| **Bridge** (`docker0`) | A power strip / switch | Connects all those cable ends together, so containers can talk |
| **iptables / nftables (NAT)** | A receptionist redirecting calls | Forwards traffic from `host:8080` to `container:80` when you use `-p 8080:80` |

```
   Browser --> host port 8080 --(NAT rule)--> docker0 bridge --(veth)--> container eth0 :80 --> nginx
```

When "my container cannot reach the internet" or "I cannot reach my container", the answer is almost always somewhere in this table.

---

## 8. How Linux shapes the logic of Docker

Now we can explain many Docker behaviours that look strange at first. None of them are Docker "decisions" — they are **direct consequences of how Linux works**.

| What you observe in Docker | The Linux reason behind it |
|---|---|
| Linux containers need Linux; Docker Desktop runs a Linux VM on Windows/macOS | Containers **share the host kernel**; there is no kernel inside the image |
| Every container reports the **same kernel version**, whatever its distribution | Same reason: one shared kernel. An "ubuntu" image contains Ubuntu's **tools and files**, not Ubuntu's kernel |
| A container starts in a fraction of a second | Creating namespaces and starting a process is cheap; nothing needs to boot |
| `docker stop` sometimes waits 10 seconds | Docker sends `SIGTERM`, waits, then sends `SIGKILL`. If your program is PID 1 and ignores `SIGTERM` (or a shell in front of it swallows the signal), the kernel's PID 1 rules mean it is not stopped until the `SIGKILL` |
| Exit code **137** | 128 + 9: the process ended after receiving `SIGKILL`. Possible causes: the **OOM killer**, `docker kill`, a stop timeout… — it does not prove which one |
| Exit code **143** | 128 + 15: the process ended after receiving `SIGTERM`. This alone does not guarantee that the shutdown was clean |
| "Permission denied" on a mounted folder | Linux compares **UID/GID numbers** between the container and the host |
| `bash: not found` in an `alpine` container | Alpine is a minimal Linux userland that ships `sh` (BusyBox), not `bash` |
| A container can use all the RAM | By default, no cgroup limit is set |
| `--privileged` is risky | It hands back almost all **capabilities**, disables most filters and exposes host devices |
| Deleting files in a later image layer does not shrink the image | OverlayFS **whiteouts** hide files; the lower layer still contains them |
| Ten containers can all listen on port 80 | Each has its **own network namespace** |

> **Key idea:** Docker is a very good **interface** to Linux. When something behaves strangely, the explanation is almost always in the kernel, not in Docker itself.

---

## 9. Why learning Linux is essential for advanced Docker

At the beginning, you can use Docker by memorising commands. That is fine for a week. But very soon you meet problems where **the command is correct and things still do not work** — and the explanation is in Linux.

**Analogy:** you can drive a car without knowing anything about engines. But to become a **mechanic** — to diagnose a strange noise, to tune it, to make it safe — you must understand the engine. **Linux is Docker's engine.**

### What each advanced Docker topic really requires

| Advanced Docker topic | Linux knowledge you need | What goes wrong if you do not know it |
|---|---|---|
| **Writing good Dockerfiles** | Package managers (`apt`, `apk`), the filesystem layout (`/etc`, `/usr`, `/var`), shells | Huge images, broken installs, "command not found" |
| **Volumes and bind mounts** | Users, groups, **UID/GID**, file permissions (`chmod`, `chown`) | "Permission denied", files owned by root on your laptop |
| **Networking** | IP addresses, ports, DNS, bridges, NAT, `iptables`/`nftables` | Containers cannot talk to each other, ports unreachable, mysterious exposure to the internet |
| **Stopping and restarting cleanly** | Processes, **signals** (`SIGTERM`, `SIGKILL`), PID 1, zombie processes | Slow shutdowns, lost data, leftover zombie processes |
| **Resource limits and performance** | **cgroups**, memory, CPU scheduling, the OOM killer | Random crashes (exit 137), one container slowing the whole server |
| **Security and hardening** | root vs non-root, **capabilities**, **seccomp**, AppArmor/SELinux, user namespaces | Containers that can take over the host |
| **Debugging** | `ps`, `top`, `/proc`, logs, `strace`, `ip`, `ss`, `curl` | Guessing instead of diagnosing |
| **Image size and build speed** | OverlayFS, layers, copy-on-write | Slow builds, gigantic images, secrets left in old layers |
| **Orchestration (Kubernetes, Swarm)** | Everything above — at scale | Problems you cannot explain on many machines at once |

### A suggested Linux learning path for Docker users

You do not need to become a Linux system administrator. Learn these, roughly in this order:

1. **The command line:** moving around (`cd`, `ls`, `pwd`), reading files (`cat`, `less`), searching (`grep`, `find`).
2. **The filesystem layout:** what lives in `/etc`, `/usr`, `/var`, `/tmp`, `/proc`, `/sys`.
3. **Users and permissions:** UID/GID, `chmod`, `chown`, `sudo`, what `root` really means.
4. **Processes and signals:** `ps`, `top`, `kill`, PID 1, exit codes.
5. **Package managers:** `apt` (Debian/Ubuntu), `apk` (Alpine), `dnf` (Fedora/Red Hat).
6. **Networking basics:** IP, ports, DNS, `ip`, `ss`, `curl`.
7. **The primitives of this lesson:** namespaces, cgroups, OverlayFS, capabilities, seccomp.

> **Good news:** Docker is a fantastic **Linux playground**. With one command you get a fresh Linux system you can break without consequences. Use containers to learn Linux, and Linux to understand containers.

Useful references:

- Linux manual pages — namespaces: <https://man7.org/linux/man-pages/man7/namespaces.7.html>
- Linux manual pages — cgroups: <https://man7.org/linux/man-pages/man7/cgroups.7.html>
- Linux manual pages — capabilities: <https://man7.org/linux/man-pages/man7/capabilities.7.html>
- Kernel documentation — OverlayFS: <https://docs.kernel.org/filesystems/overlayfs.html>
- Docker documentation — Docker Engine security: <https://docs.docker.com/engine/security/>

---

## 10. Putting it all together

```
                          ONE CONTAINER
   +--------------------------------------------------------------+
   |  your program (a normal Linux process)                       |
   |                                                              |
   |  SEES      -> namespaces   (pid, net, mnt, uts, ipc, ...)    |
   |  USES      -> cgroups      (memory, cpu, pids, io)           |
   |  STANDS ON -> OverlayFS    (image layers + writable layer)   |
   |  ALLOWED   -> capabilities + seccomp + AppArmor/SELinux      |
   |  TALKS     -> veth + bridge + NAT                            |
   +--------------------------------------------------------------+
                              |
                  set up by runc, organised by
                  containerd and dockerd
                              |
   ==============================================================
                     THE ONE, SHARED LINUX KERNEL
   ==============================================================
```

---

## Remember

- The kernel has **no "container" object**. A container = **a process + namespaces + cgroups + layered filesystem + security filters**.
- **Namespaces** = what it **sees**. **Cgroups** = what it **uses**. **OverlayFS** = what it **stands on**. **Capabilities/seccomp/LSM** = what it is **allowed to do**.
- All containers **share one kernel** — this explains the speed, the lightness, the Linux-only rule and the weaker isolation.
- Many "strange" Docker behaviours (exit 137, slow stops, permission errors) are **Linux behaviours**.
- To master advanced Docker, **learn Linux**: processes, signals, users and permissions, networking, and the primitives above.

---

## Quick quiz

1. Does the Linux kernel have a "container" object? What does it have instead?
2. Which primitive controls what a container can **see**? Which one controls what it can **use**?
3. A container is limited to 256 MB of memory and tries to use 400 MB. What happens? And if it is limited to half a CPU and wants more?
4. Why do an `ubuntu` container and an `alpine` container report the same kernel version?
5. Why can "root inside a container" still be dangerous with Docker's default settings?
6. Give two advanced Docker topics where Linux knowledge is essential, and explain why.

<details>
<summary>Show the answers</summary>

1. No. It has separate features — namespaces, cgroups, layered filesystems, capabilities, seccomp, LSMs — that Docker combines.
2. **Namespaces** control what it sees; **cgroups** control what it uses.
3. Memory: the kernel tries to reclaim memory; if it cannot, the **OOM killer** kills the process (exit code 137 — check `OOMKilled` to be sure). CPU: the process is simply **slowed down**; it is not killed.
4. Because both share the **same host kernel**. An image contains a distribution's files and tools, never a kernel.
5. Because the USER namespace is not enabled by default, so container root is UID 0 on the host. Capabilities and seccomp reduce its powers, but a mistake (for example `--privileged` or a dangerous mount) can give it control of the host.
6. For example: **volumes** (UID/GID and permissions), **networking** (namespaces, bridges, NAT), **graceful shutdown** (signals and PID 1), **resource limits** (cgroups and the OOM killer), **security** (capabilities, seccomp, non-root users).

</details>

---

**Previous lesson:** [04 — Docker Engine](./04-docker-engine.md) · **Next step:** [Exercises](../exercices/README.md)
