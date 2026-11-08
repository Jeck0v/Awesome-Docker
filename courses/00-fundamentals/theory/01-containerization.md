# Lesson 01 — Containerisation

> **In one sentence:** Containerisation means packing a program with everything it needs into an isolated box (a *container*) that behaves the same wherever you run it.

---

## What you will learn

- what "containerisation" really means;
- the **three big promises** of a container: *packaging*, *isolation* and *portability*;
- how an image is built from **layers**, like a stack of transparent sheets;
- why containers are **disposable**, and why that is a good thing;
- the limits of containers (what they cannot do).

---

## 1. The word itself

"Containerisation" is just a long word for **"putting things into containers"**.

In the real world, a port that "containerises" its goods puts everything into standard metal boxes. In software, we do the same with programs.

A **container** is:

1. **a program** (your website, your game server, your script…);
2. **plus everything that program needs to run** (libraries, tools, settings, files);
3. **running in its own private space**, separated from the rest of the computer.

---

## 2. The three big promises

### Promise 1 — Packaging: "Everything is in the box"

Think of a **school trip bag**. Before the trip, your parents pack everything you need: sandwiches, a water bottle, a raincoat, your pencil case. When you arrive, you do not need to ask anyone for anything — it is all in the bag.

A container is the same. Your program does not need to "ask" the computer for Python or for a library. **It brought its own.**

### Promise 2 — Isolation: "Everyone has their own bedroom"

Imagine three children sharing **one** bedroom. One wants to sleep, one wants to play music, one wants the light on. Chaos!

Now give each child **their own bedroom**. They live in the same house, but they cannot mess up each other's things.

Containers give each program its own "bedroom":

- its **own view of the files** — by default, a container cannot see the files of another (unless you deliberately share some, for example with a volume or a bind mount);
- its **own list of running programs** — it thinks it is alone on the computer;
- its **own network address**;
- its **own share** of memory and processor, if you decide to set limits.

So you can run a program that needs Python 3.8 and another that needs Python 3.13 **side by side**, and they will never fight.

```
                    YOUR COMPUTER
 +---------------------------------------------------+
 |  +-------------+ +-------------+ +-------------+  |
 |  | Container A | | Container B | | Container C |  |
 |  | Python 3.8  | | Python 3.13 | | Node.js 22  |  |
 |  | old app     | | new app     | | website     |  |
 |  +-------------+ +-------------+ +-------------+  |
 |   cannot see      cannot see      cannot see      |
 |   B or C          A or C          A or B          |
 +---------------------------------------------------+
```

### Promise 3 — Portability: "It runs everywhere"

Because everything is inside the box, you can move the box:

- from your laptop,
- to your colleague's laptop,
- to a test server,
- to the cloud,

…and it still works the same way. Just like a shipping container moves from a ship to a lorry to a train without anyone opening it.

There are conditions, though. The new machine must provide a compatible **kernel** (Linux, for Linux containers) and a compatible **processor architecture**: an image built for ARM64 does not run on an AMD64 machine without emulation. And anything the program uses *outside* the box (a database, a network service, a mounted folder) must also be available there.

---

## 3. Images are made of layers

This is one of the cleverest ideas in Docker, so take your time.

### The transparent sheets analogy

Imagine you draw a picture using **transparent plastic sheets**, one on top of another:

- **Sheet 1:** you draw the sky and the grass.
- **Sheet 2:** you draw a house.
- **Sheet 3:** you draw a cat in the window.

Put them on top of each other and you see the full picture. Each sheet only contains **what changed** compared with the sheets underneath.

```
   Sheet 3:  the cat               <- your program
   Sheet 2:  the house             <- Python installed
   Sheet 1:  the sky and the grass <- a small Linux (for example Debian)
   ---------------------------------
   Looking from above: the complete picture = the image
```

A Docker **image** works exactly like this. Each instruction in the Dockerfile adds a new **layer** on top of the previous ones.

### Why is this so useful?

**1. Sharing saves space.** Suppose you have ten images that all start with the same "sky and grass" sheet (the same base Linux). Docker stores that sheet **only once** and all ten images share it.

**2. Rebuilding is fast.** If you only change the cat, Docker does not need to redraw the sky and the house. It reuses them from its **build cache** and only redraws the cat. But be careful: if you change the **house**, the cat must be redrawn too. A change in one layer invalidates the cache for that layer **and every layer above it**. That is why the order of the instructions in a Dockerfile matters.

**3. Downloading is fast.** When you download (pull) an image, Docker only downloads the layers you do not already have.

### The writable layer of a container

Image layers are **read-only** — nobody is allowed to scribble on them.

So what happens when a running container needs to write a file? Docker puts **one extra, empty, writable sheet** on top, just for that container.

```
   +-------------------------------+
   | writable layer (container A)  |  <- private to container A, can change
   +-------------------------------+
   | layer 3: your program         |  \
   | layer 2: Python               |   > read-only, shared by every
   | layer 1: base Linux           |  /  container made from this image
   +-------------------------------+
```

If container A changes a file that belongs to a lower layer, Docker first **copies** that file into the writable layer, then changes the copy. The original stays untouched. This trick is called **copy-on-write**.

> **Consequence to remember:** when you delete the container, its writable sheet goes in the bin. Everything written there is **lost**.

---

## 4. Containers are disposable — and that is a feature

Imagine you are using **paper plates** at a picnic. When a plate is dirty, you do not wash it. You throw it away and take a fresh one. Fast, clean, no fuss.

Containers are meant to be treated like **paper plates**, not like your grandmother's precious china:

- something is broken? Delete the container and start a fresh one from the image;
- need a new version? Build a new image and replace the old containers;
- never "repair by hand" inside a running container and hope you remember what you did.

This idea has a name: containers should be **ephemeral** (short-lived) and **immutable** (you replace them; you do not change them).

### "But what about my important data?"

Good question. If containers are thrown away, where do you keep data that must survive — for example, the contents of a database?

You store it **outside** the container, in a **volume**. A volume is like a **lunchbox you keep** while the paper plates are thrown away. You will learn about volumes in a later course; for now, remember:

> **Program inside the container. Precious data outside the container.**

---

## 5. One container, one job

A good habit: **each container has one clear responsibility.**

This is a good practice, not an absolute rule. A container can run several closely related processes (for example with a small process supervisor). What matters is that each container has a clear role.

Think of a restaurant kitchen. There is a chef for the sauces, a chef for the desserts and someone washing up. If one person did everything at once, it would be a mess, and if they were ill, the entire kitchen would stop.

For a typical website you would have:

```
 +-----------+     +-----------+     +-----------+
 |  website  | --> |    API    | --> | database  |
 | container |     | container |     | container |
 +-----------+     +-----------+     +-----------+
```

Benefits:

- you can update the website without touching the database;
- you can restart one piece if it crashes;
- you can run more copies of the busy piece only.

---

## 6. A container is just a process (a sneak peek)

Here is a secret that surprises many people:

> **A container is not a tiny computer. It is a normal program running on your computer, wearing special "blinkers" so it can only see its own little world.**

Blinkers are the leather flaps that horses wear next to their eyes so they only look straight ahead. The Linux kernel can put "blinkers" on a program so that it:

- only sees its own files,
- only sees its own processes,
- only sees its own network,
- and is only allowed to use a certain amount of memory and CPU.

The Linux features that do this are called **namespaces** and **cgroups**. They are so important that they have their own lesson: [Lesson 05 — Linux kernel primitives](./05-linux-kernel-primitives.md).

---

## 7. Standards: the OCI

If every company invented its own kind of container, we would be back to the pre-1950s port with boxes of all shapes.

That is why the **Open Container Initiative (OCI)** exists. It writes the official rules for:

- **what an image looks like** (the *image specification*);
- **how to run a container** (the *runtime specification*);
- **how registries share images** (the *distribution specification*).

Thanks to the OCI, an image built with Docker can be run by other tools (such as Podman or Kubernetes), and the other way round.

Official website: <https://opencontainers.org>

---

## 8. What containers are good at — and what they are not

| Good at | Not good at (or needs care) |
|---|---|
| Running the same program identically on any compatible host | Running a program for a **different operating-system kernel** (for example, a Windows program in a Linux container) |
| Starting quickly | Keeping data, unless you use volumes |
| Running many programs side by side without conflicts | Perfect security isolation — they share the host's kernel (see [Lesson 02](./02-containers-vs-virtual-machines.md)) |
| Trying software without installing it on your computer | Graphical desktop applications (possible, but awkward) |
| Making builds and tests repeatable | Replacing good habits: a badly written program is still badly written inside a container |

---

## Remember

- **Containerisation** = packing a program with all it needs into an isolated, portable box.
- Three promises: **packaging**, **isolation**, **portability**.
- Images are built from **read-only layers**; each container gets one **writable layer** on top.
- Containers are **disposable**: replace them, do not repair them.
- Precious data lives **outside** the container, in volumes.
- A container is really a **normal process with blinkers**.

---

## Quick quiz

1. Name the three big promises of a container.
2. Ten images share the same base layer. How many times is that layer stored on disk?
3. You create a file inside a container, then delete the container. What happens to the file?
4. Why is "one container, one job" a good habit?
5. What is the OCI for?

<details>
<summary>Show the answers</summary>

1. Packaging, isolation, portability.
2. Only **once**. Layers are shared.
3. It is **lost**. It lived in the container's writable layer, which is deleted with the container.
4. Each piece can be updated, restarted or scaled on its own, and a problem in one piece does not bring down everything else.
5. It defines open standards for images, runtimes and registries, so that containers work with many tools, not just Docker.

</details>

---

**Previous lesson:** [00 — What is Docker?](./00-what-is-docker.md) · **Next lesson:** [02 — Containers vs virtual machines](./02-containers-vs-virtual-machines.md)
