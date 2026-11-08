# 00 — Docker Fundamentals

Welcome to the very first course of **Awesome-Docker**.

This course is for complete beginners. You do not need to know anything about Docker, servers or Linux. Everything is explained in plain English, with everyday comparisons (lunchboxes, restaurants, blocks of flats, pocket money…), and every new word is defined before it is used.

By the end, you will not just *use* Docker — you will understand **what it really is, how it works inside, and why it behaves the way it does**.

---

## What you will be able to do

- explain what Docker is and which problem it solves;
- tell the difference between an **image**, a **container**, a **Dockerfile** and a **registry**;
- compare **containers** with **virtual machines**, and know when to use which;
- describe Docker's **architecture** (client, daemon, registry) and the **engine's** components (`dockerd`, `containerd`, `runc`);
- explain the **Linux kernel features** that make containers possible (namespaces, cgroups, OverlayFS, capabilities, seccomp);
- run, inspect, stop and clean up containers and images **safely**, with good habits from day one.

---

## Prerequisites

- A computer with **Docker installed**:
  - Windows or macOS: [Docker Desktop](https://docs.docker.com/desktop/)
  - Linux: [Docker Engine](https://docs.docker.com/engine/install/)
- A terminal (PowerShell on Windows, Terminal on macOS/Linux).
- Curiosity. That is all.

---

## How the course is organised

```
00-fundamentals/
├── README.md        <- you are here
├── theory/          <- the lessons: read them in order
├── exercices/       <- hands-on exercises (no answers inside)
│   └── solutions/   <- code reviews of typical attempts
├── examples/
└── diagrams/
```

### 1. Theory — read in order

| # | Lesson | In one sentence |
|---|---|---|
| 00 | [What is Docker?](./theory/00-what-is-docker.md) | A tool that packs a program with everything it needs, so it runs the same everywhere |
| 01 | [Containerisation](./theory/01-containerization.md) | Packaging, isolation, portability — and images built from layers |
| 02 | [Containers vs virtual machines](./theory/02-containers-vs-virtual-machines.md) | Separate houses versus flats in one block — with a full comparison table |
| 03 | [Docker architecture](./theory/03-docker-architecture.md) | A restaurant: the client takes orders, the daemon cooks, the registry supplies |
| 04 | [The Docker Engine](./theory/04-docker-engine.md) | A team: `dockerd`, `containerd`, the shim and `runc` |
| 05 | [Linux kernel primitives](./theory/05-linux-kernel-primitives.md) | A container is a Linux process with blinkers, a budget and a reduced set of keys — and why learning Linux matters |

Each lesson ends with a **"Remember"** summary and a short **quiz** (answers hidden until you click).

### 2. Exercises — practise

The [exercises](./exercices/README.md) are designed for someone who has **just installed Docker**. They tell you **what** to achieve, never **how**. Hints point you to the theory and to the official Docker documentation, so that you learn to find answers yourself — the most valuable skill of all.

### 3. Solutions — compare and learn

The [solutions](./exercices/solutions/) are **not** answer sheets. Each one is a **code review** of a realistic attempt, explaining:

- **why** one option is used rather than another;
- **why** it matters;
- **what can go wrong** if you choose something else.

Do the exercise first. Then read the review. You will learn far more from comparing than from copying.

---

## Suggested pace

| Step | Time (approximate) |
|---|---|
| Lessons 00 to 02 | 1 hour |
| Lessons 03 and 04 | 1 hour |
| Lesson 05 | 1 hour (take your time, it is the most important) |
| Exercises 01 to 05, with their reviews | 3 to 5 hours |

There is no rush. Understanding beats speed.

---

## Useful links

- Official Docker documentation: <https://docs.docker.com/>
- Docker CLI reference: <https://docs.docker.com/reference/cli/docker/>
- Docker Hub (public images): <https://hub.docker.com/>
- Open Container Initiative: <https://opencontainers.org/>

---

**Start here:** [Lesson 00 — What is Docker?](./theory/00-what-is-docker.md)
