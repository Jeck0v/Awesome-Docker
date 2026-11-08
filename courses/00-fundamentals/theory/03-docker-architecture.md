# Lesson 03 — Docker Architecture

> **In one sentence:** Docker is built like a restaurant: you (the customer) give orders to a waiter (the **Docker client**), who passes them to the kitchen (the **Docker daemon**), which fetches ingredients from a supplier (the **registry**) and cooks dishes (the **containers**).

---

## What you will learn

- the **client–server** design of Docker;
- the role of each piece: **client**, **daemon**, **registry**, **objects**;
- how the client and the daemon talk to each other (the **API** and the **socket**);
- exactly what happens, step by step, when you type `docker run hello-world`;
- what **Docker Desktop** adds on top.

---

## 1. The big picture

When you type a `docker` command, you are **not** talking to the part that does the work. You are talking to a messenger.

Docker is made of three main parts:

```
   +-------------------+        +----------------------------------+        +------------------+
   |   DOCKER CLIENT   |        |        DOCKER HOST               |        |    REGISTRY      |
   |                   |        |                                  |        |                  |
   |  docker build     |  API   |   +--------------------------+   |  pull  |  Docker Hub      |
   |  docker pull  ----+------->|   |  Docker daemon (dockerd) |---+------->|  or a private    |
   |  docker run       |        |   +--------------------------+   |<-------+  registry        |
   |                   |        |          |            |          |  push  |                  |
   |  (you type here)  |        |          v            v          |        |  nginx, alpine,  |
   +-------------------+        |     +---------+  +-----------+   |        |  postgres, ...   |
                                |     | IMAGES  |  | CONTAINERS|   |        +------------------+
                                |     +---------+  +-----------+   |
                                |     +---------+  +-----------+   |
                                |     | VOLUMES |  | NETWORKS  |   |
                                |     +---------+  +-----------+   |
                                +----------------------------------+
```

1. the **client** — where you type commands;
2. the **host**, where the **daemon** lives and does the real work;
3. the **registry** — where images are stored and shared.

---

## 2. The restaurant analogy

| Restaurant | Docker | What it does |
|---|---|---|
| You, the customer | **You** | You decide what you want |
| The waiter | **Docker client** (`docker` command) | Takes your order and carries it to the kitchen. Does not cook. |
| The order slip | **Docker Engine API** (an HTTP API) | The standard format the order is written in |
| The kitchen and the chef | **Docker daemon** (`dockerd`) | Does all the real work |
| The recipe book | **Images** | Instructions/ingredients ready to be used |
| The dishes being served | **Containers** | Images brought to life |
| The cold room / pantry | **Volumes** | Where things are kept, even after the dish is eaten |
| The corridors between tables | **Networks** | How dishes and staff move around |
| The food supplier | **Registry** (Docker Hub) | Supplies ingredients (images) the kitchen does not have yet |

The most important lesson of this analogy:

> **The waiter does not cook.** The `docker` command you type does almost nothing on its own. It just sends a message to the daemon, which does the work.

This is why, if the daemon is not running (for example, Docker Desktop is closed), you get an error like:

```
Cannot connect to the Docker daemon at ... Is the docker daemon running?
```

The waiter is there, but the kitchen is closed.

---

## 3. Each piece in more detail

### 3.1 The Docker client

- It is the `docker` program you use in your terminal (PowerShell, Bash…).
- It turns your command into a request for the daemon.
- It can talk to the daemon on **your own computer**, or to a daemon on **another computer** (a server). The client and the daemon do not have to be on the same machine.

Other tools can be clients too: Docker Desktop's graphical interface, Docker Compose, editor extensions… They all speak to the same daemon.

### 3.2 The Docker daemon (`dockerd`)

"Daemon" is an old computing word for **a program that runs quietly in the background and waits for work** (like a helpful house elf, always on duty).

The Docker daemon:

- listens for requests from clients;
- **builds** images;
- **downloads** (pulls) and **uploads** (pushes) images;
- **creates, starts, stops and deletes** containers;
- manages **networks** and **volumes**.

In reality, the daemon delegates part of the work to other helpers (`containerd` and `runc`). We open the kitchen door and meet them in [Lesson 04 — Docker Engine](./04-docker-engine.md).

### 3.3 The Docker Engine API and the socket

The client and the daemon talk using the **Docker Engine API**: an API exposed over **HTTP**, made of well-defined endpoints, like a standard order slip. It is often described as a REST API; some operations (such as logs or attaching to a container) also use **streams** that stay open.

These messages travel through a **socket**, a sort of private telephone line:

- on Linux: a file called `/var/run/docker.sock`;
- on Windows: a "named pipe" called `//./pipe/docker_engine`;
- or, if configured, over the network (TCP), ideally protected with TLS.

> **Security note for later:** whoever can talk to the Docker socket can ask the daemon to do *anything*, including things that give full control of the host. Treat access to the socket like the **master key of the building**.

### 3.4 Registries

A **registry** is a storage and distribution service for images.

- **Docker Hub** is the default public registry: when you ask for `nginx`, Docker actually looks for `docker.io/library/nginx:latest`.
- Companies often use **private registries** (GitHub Container Registry, GitLab, Amazon ECR, Google Artifact Registry, Azure Container Registry, Harbor…).

An image name is made of several parts:

```
   docker.io / library / nginx : 1.27
   \_______/   \_____/   \___/   \__/
   registry   namespace  name    tag  (the version label)
```

If you leave parts out, Docker fills in defaults: the registry `docker.io`, the namespace `library` (official images), and the tag `latest`.

> **Watch out:** `latest` is just a **label**, chosen by default. It does **not** guarantee that you get the newest version. We will come back to this in the exercises.

### 3.5 Docker objects

These are the "things" the daemon manages. You will meet each of them in detail in later courses.

| Object | Simple description | Analogy |
|---|---|---|
| **Image** | A read-only template to create containers | A frozen ready meal |
| **Container** | A running (or stopped) instance of an image | The meal on the plate |
| **Volume** | Storage that survives when containers are deleted | A lunchbox you keep |
| **Network** | Lets containers talk to each other and to the outside world | Corridors and telephones |

---

## 4. Step by step: what happens when you type `docker run hello-world`

This is the very first command most people run. Let us follow it through the restaurant.

```
  You                Client               Daemon                 Registry (Docker Hub)
   |                   |                    |                           |
   |-- docker run ---->|                    |                           |
   |   hello-world     |-- API request ---->|                           |
   |                   |   "create & start  |                           |
   |                   |    hello-world"    |                           |
   |                   |                    |-- Do I have the image? -- |
   |                   |                    |   No.                     |
   |                   |                    |------ pull request ------>|
   |                   |                    |<----- image layers -------|
   |                   |                    |                           |
   |                   |                    |-- create container        |
   |                   |                    |-- start container         |
   |                   |                    |   (program prints text)   |
   |                   |<-- output stream --|                           |
   |<-- text on screen-|                    |                           |
   |                   |                    |-- program ends,           |
   |                   |                    |   container stops         |
```

1. **You** type `docker run hello-world`.
2. The **client** sends a request to the **daemon** through the socket.
3. The daemon checks: *"Do I already have the image `hello-world:latest`?"*
4. The first time, the answer is no, so the daemon **pulls** it from **Docker Hub**. (You will see `Unable to find image 'hello-world:latest' locally` followed by `Pulling from library/hello-world`.)
5. The daemon **creates** a container from the image (adds the writable layer, prepares the isolation).
6. The daemon **starts** the container: the small program inside it runs.
7. The program prints a message. The daemon **streams** that output back to the client, which shows it in your terminal.
8. The program finishes, so the container **stops** (it still exists, in the "Exited" state, until you delete it).

The second time you run the same command, step 4 is skipped: the image is already on your computer.

> **Fun fact:** the `hello-world` program explains these same steps in its own message. Read it carefully the first time you run it.

---

## 5. Docker Desktop: the full package

On Windows and macOS you usually install **Docker Desktop**. It is not just Docker Engine with a graphical interface: it is a complete environment that provides a Linux VM, integration with your operating system (files, network, credentials), resource management and various tools. It contains, among other things:

| Component | Role |
|---|---|
| The **Docker client** (`docker` CLI) | What you type in the terminal |
| The **Docker Engine** (daemon) | Runs inside a small **Linux VM** (WSL 2 on Windows) — see [Lesson 02](./02-containers-vs-virtual-machines.md) |
| **Docker Compose** | Runs several containers together from a single file |
| **Docker Buildx / BuildKit** | The modern image builder |
| A **graphical dashboard** | See and manage containers, images and volumes with the mouse |
| Extra tools | Image scanning, Kubernetes, extensions… |

On a Linux server, you usually install **Docker Engine** alone, without the desktop application. The daemon then runs directly on the machine.

---

## Remember

- Docker uses a **client–server** architecture.
- The **client** (`docker`) only **sends orders**; the **daemon** (`dockerd`) **does the work**.
- They talk through the **Docker Engine API** (HTTP), usually over a **Unix socket** or a **named pipe**. Access to the socket = master key.
- **Registries** store and share images; Docker Hub is the default.
- An image name = `registry/namespace/name:tag`; `latest` is **just a label**.
- Docker objects: **images, containers, volumes, networks**.

---

## Quick quiz

1. When you type `docker ps`, which part actually finds the list of containers?
2. You see "Cannot connect to the Docker daemon". In restaurant terms, what is going on?
3. What is the full name of the image `nginx` if you write it without any defaults?
4. Why is the first `docker run hello-world` slower than the second one?
5. Why should access to the Docker socket be protected?

<details>
<summary>Show the answers</summary>

1. The **daemon**. The client only asks for it and displays the answer.
2. The waiter (client) is there, but the kitchen (daemon) is closed — for example, Docker Desktop is not started.
3. `docker.io/library/nginx:latest`.
4. The first time, the daemon must **download** the image from the registry. The second time it is already stored locally.
5. Anyone who can talk to the daemon can make it do anything, including taking control of the host machine.

</details>

---

**Previous lesson:** [02 — Containers vs virtual machines](./02-containers-vs-virtual-machines.md) · **Next lesson:** [04 — Docker Engine](./04-docker-engine.md)
