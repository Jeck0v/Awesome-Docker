# Lesson 04 — The Docker Engine

> **In one sentence:** The Docker Engine is not one big program but a small team — `dockerd`, `containerd`, the `containerd-shim` and `runc` — where each member has one clear job, and together they turn an image into a running container.

---

## What you will learn

- what is inside the **Docker Engine**;
- the job of each team member: **dockerd**, **containerd**, **containerd-shim**, **runc**;
- why Docker was split into several pieces instead of one;
- what **BuildKit**, the **storage driver** and the **network** part do;
- how to see all of this on your own computer.

---

## 1. Opening the kitchen door

In [Lesson 03](./03-docker-architecture.md), we said the **daemon** is the kitchen. Now we open the door and discover that the kitchen has a **team**, a bit like a building site:

| Building site | Docker Engine | Job |
|---|---|---|
| The **project manager** who talks to the client | **dockerd** | Receives your requests, handles images, builds, networks and volumes |
| The **site foreman** who organises the workers | **containerd** | Manages the life of containers: download images, create, start, stop |
| The **site supervisor** who stays on site | **containerd-shim** | Stays beside the container(s) it manages for their whole life |
| The **worker** who actually lays the bricks | **runc** | Asks the Linux kernel to create the container, then leaves |
| The **ground** everyone builds on | **Linux kernel** | Provides the isolation (namespaces, cgroups…) |

```
   docker CLI   (you)
       |
       |  Docker Engine API (HTTP)
       v
   +-----------+
   |  dockerd  |   project manager: API, images, builds, networks, volumes
   +-----------+
       |
       |  gRPC (a fast internal "telephone")
       v
   +------------+
   | containerd |   foreman: lifecycle of containers, image storage
   +------------+
       |
       |  shims (typically one per container with Docker)
       v
   +------------------+     +------------------+
   | containerd-shim  |     | containerd-shim  |   supervisors (stay)
   +------------------+     +------------------+
       |                          |
       v                          v
     runc                       runc              workers (create, then leave)
       |                          |
       v                          v
   [container 1]             [container 2]
   ================================================
               LINUX KERNEL (namespaces, cgroups, ...)
```

---

## 2. Each team member in detail

### 2.1 `dockerd` — the project manager

- Listens to the **Docker API** (your `docker` commands arrive here).
- Manages **images**, **builds**, **networks**, **volumes**, logs, authentication with registries.
- Does **not** create containers itself any more: it passes that job to **containerd**.

### 2.2 `containerd` — the foreman

- A separate program (and an independent open-source project, part of the CNCF).
- Manages the **lifecycle** of containers: create, start, pause, stop, delete.
- Handles **image storage and transfer** (pulling and unpacking layers).
- It is so good at its job that **Kubernetes** can use containerd directly, without Docker.

### 2.3 `containerd-shim` — the supervisor that stays

Once runc has created a container, it **leaves**. Someone has to stay with the container to:

- keep its input and output open (so that `docker logs` works);
- report when it stops and with which **exit code**;
- keep it alive even if `dockerd` or `containerd` is restarted.

That "someone" is the **shim**. A shim process manages the lifecycle of one or more container tasks, depending on the runtime implementation. With Docker's usual setup, you will typically see one shim per container.

### 2.4 `runc` — the worker

- A small, low-level tool that follows the **OCI runtime specification** (see [Lesson 01](./01-containerization.md)).
- It talks directly to the **Linux kernel** to set up the "blinkers" (namespaces), the "budget" (cgroups) and the filesystem, then starts your program inside.
- Once the container is running, runc **exits**. Its work is done.

Other runtimes can replace runc (for example `crun`, or `gVisor` and `Kata Containers` for stronger isolation), because they all follow the same OCI rules.

---

## 3. Why split the engine into several pieces?

In the very early days, Docker was **one big program** that did everything. That caused problems. Imagine a building site where the project manager also lays every brick: if the manager goes home for lunch, **the whole site stops**.

Splitting the engine gave several benefits:

| Benefit | Simple explanation |
|---|---|
| **Containers survive a daemon restart** | Because the shims keep containers running independently of `dockerd`, you can restart or upgrade `dockerd` without killing your containers (with the `live-restore` option enabled). |
| **Each piece does one job well** | Easier to test, fix and improve. |
| **Open standards** | runc and containerd follow OCI rules, so other tools (Kubernetes, Podman, nerdctl…) can reuse them. |
| **Choice** | You can swap runc for another runtime without changing the rest. |

---

## 4. The other important parts of the engine

### 4.1 BuildKit — the image builder

When you run `docker build`, the work is done by **BuildKit**, the modern builder (default since Docker Engine 23). BuildKit is a **separate project**: Docker integrates it, but it can also be used on its own, without Docker Engine.

What makes it good:

- it builds independent steps **in parallel**;
- it has a smart **cache**: it skips steps that have not changed (the "transparent sheets" from Lesson 01);
- it can build images for **other processor types** (for example ARM for a Raspberry Pi) with `docker buildx`.

### 4.2 The storage driver — how layers are stacked

Remember the transparent sheets from [Lesson 01](./01-containerization.md)? Something has to physically **stack those layers** and present them as a single folder to the container.

That is the job of the **storage driver** (or, in the newer design, the containerd **snapshotter**). On modern Linux, it is almost always **overlay2**, based on the Linux feature **OverlayFS** — which we explain in [Lesson 05](./05-linux-kernel-primitives.md).

> Newer Docker versions are moving to the **containerd image store**, where containerd stores the images itself using **snapshotters** (including an OverlayFS snapshotter). Depending on your version and settings, `docker info` may show `overlay2` or `overlayfs` with a containerd snapshotter. The two are **not** strictly equivalent: they have different storage architectures, even though both typically rely on OverlayFS on Linux.

### 4.3 Networking

The engine also creates the "corridors" between containers:

- by default, containers are connected to a virtual network called **`bridge`**;
- each container receives its own private IP address;
- when you "publish a port" (`-p`), the engine sets up rules so that traffic arriving on your computer is forwarded to the container.

Under the hood, all of this uses Linux networking features (virtual network cards, bridges, firewall rules). You will see in [Lesson 05](./05-linux-kernel-primitives.md) why this is a big reason to learn Linux.

---

## 5. The journey of a container, through the engine

Let us redo `docker run nginx`, but this time **inside** the engine:

```
 1. docker CLI   --> "run nginx"                               --> dockerd
 2. dockerd      --> checks the image, prepares network/volumes
 3. dockerd      --> "create and start this container"         --> containerd
 4. containerd   --> pulls/unpacks the image if needed, prepares the filesystem
 5. containerd   --> starts a containerd-shim for this container
 6. shim         --> calls runc with an OCI "bundle" (config + filesystem)
 7. runc         --> asks the Linux kernel: namespaces, cgroups, mounts
 8. runc         --> starts nginx inside, then EXITS
 9. shim         --> stays, watches nginx, keeps its logs flowing
10. you          --> "docker logs", "docker stop"... go back down the same chain
```

---

## 6. See it on your own computer

You do not have to take this lesson on trust. Two commands show the engine's components:

- `docker version` — shows the **Client** and the **Server** separately. In the Server part you will find the versions of **Engine**, **containerd**, **runc** and **docker-init**.
- `docker info` — shows details such as the **Storage Driver**, the **Cgroup Version**, the **Kernel Version**, the default **Runtime**, and the number of containers and images.

You will use them in the first exercise. Read the output slowly: everything in this lesson is visible there.

> On Docker Desktop for Windows, the "Kernel Version" shown by `docker info` is a **Linux** kernel version (with "microsoft" or "WSL" in its name). That is the kernel of the small Linux VM from [Lesson 02](./02-containers-vs-virtual-machines.md). It proves that your containers run on Linux, even on a Windows PC.

---

## Remember

- The Docker Engine is a **team**: `dockerd` → `containerd` → `containerd-shim` → `runc` → Linux kernel.
- **dockerd** = API, images, builds, networks, volumes.
- **containerd** = container lifecycle and image storage.
- **shim** = stays with the container(s) it manages (logs, exit code, survives daemon restarts).
- **runc** = creates the container using the kernel, then leaves.
- **BuildKit** builds images; the **storage driver** (overlay2) or a containerd **snapshotter** stacks layers; the engine also manages **networking**.
- `docker version` and `docker info` let you see all of this.

---

## Quick quiz

1. Which component actually talks to the Linux kernel to create the container?
2. Why is a shim needed once runc has exited?
3. What happens to running containers if `dockerd` is restarted with `live-restore` enabled?
4. Which component builds images when you run `docker build`?
5. Which command shows the versions of containerd and runc?

<details>
<summary>Show the answers</summary>

1. **runc** (the OCI runtime).
2. To keep the container's input/output open, collect its exit code and keep it independent from `dockerd` and `containerd`, so it survives their restart.
3. They **keep running**, because the shims keep them alive.
4. **BuildKit**.
5. `docker version` (in the Server section).

</details>

---

**Previous lesson:** [03 — Docker architecture](./03-docker-architecture.md) · **Next lesson:** [05 — Linux kernel primitives](./05-linux-kernel-primitives.md)
