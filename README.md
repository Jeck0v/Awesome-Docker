# Awesome-Docker

Docker learning from beginner to advanced, with practical examples, tips, explanations, diagrams, and AI-powered development workflows.

---

## About this course

**Awesome-Docker** is a free, step-by-step course for anyone who wants to learn Docker properly, not just copy commands, but understand **what Docker really is, how it works inside, and why it behaves the way it does**.

It starts from zero. You do not need to know anything about Docker, servers or Linux. Every new word is defined before it is used, and ideas are explained with everyday comparisons (lunchboxes, restaurants, blocks of flats…) before going into the technical details.

The course is built around three principles:

- **Understand before you type.** Each topic starts with theory, explained simply, then goes deeper.
- **Learn by doing.** Exercises tell you *what* to achieve, never *how*. You find the commands yourself, with hints pointing to the lessons and the official documentation.
- **Learn from reviews, not answer sheets.** Solutions are written as **code reviews** of realistic attempts: why one option is better, why it matters, and what goes wrong otherwise.

---

## What you will learn

By the end of the currently available module, you will be able to:

- explain what Docker is and which problem it solves;
- tell the difference between an **image**, a **container**, a **Dockerfile** and a **registry**;
- compare **containers** with **virtual machines**, and know when to use which;
- describe Docker's **architecture** (client, daemon, registry) and the **engine's** components (`dockerd`, `containerd`, `runc`);
- explain the **Linux kernel features** that make containers possible (namespaces, cgroups, OverlayFS, capabilities, seccomp);
- run, inspect, stop and clean up containers and images **safely**, with good habits from day one.

---

## What is available now

### [00 - Docker Fundamentals](./courses/00-fundamentals/README.md)

The first module, for complete beginners.

**Theory: 6 lessons, each ending with a "Remember" summary and a quiz**

| # | Lesson |
|---|---|
| 00 | [What is Docker?](./courses/00-fundamentals/theory/00-what-is-docker.md) |
| 01 | [Containerisation](./courses/00-fundamentals/theory/01-containerization.md) |
| 02 | [Containers vs virtual machines](./courses/00-fundamentals/theory/02-containers-vs-virtual-machines.md) |
| 03 | [Docker architecture](./courses/00-fundamentals/theory/03-docker-architecture.md) |
| 04 | [The Docker Engine](./courses/00-fundamentals/theory/04-docker-engine.md) |
| 05 | [Linux kernel primitives](./courses/00-fundamentals/theory/05-linux-kernel-primitives.md) |

**Exercises: 5 hands-on exercises, with a code review for each**

| # | Exercise | Difficulty |
|---|---|---|
| 01 | First contact: check your installation | Very easy |
| 02 | Your first web server | Easy |
| 03 | Life inside a container | Easy |
| 04 | Images, tags and spring cleaning | Easy to medium |
| 05 | Seeing Linux at work | Medium |

→ [Exercises](./courses/00-fundamentals/exercices/README.md) · [Solutions (code reviews)](./courses/00-fundamentals/exercices/solutions/)

**Estimated time:** about 4 to 8 hours, lessons and exercises included.

More modules will follow, going from these fundamentals to advanced topics and AI-powered development workflows.

---

## Repository structure

```
Awesome-Docker/
├── README.md                  <- you are here
└── courses/
    └── 00-fundamentals/
        ├── README.md          <- module overview and suggested pace
        ├── theory/            <- the lessons, to read in order
        └── exercices/         <- hands-on exercises (no answers inside)
            └── solutions/     <- code reviews of typical attempts
```

---

## Prerequisites

- A computer with **Docker installed**:
  - Windows or macOS: [Docker Desktop](https://docs.docker.com/desktop/)
  - Linux: [Docker Engine](https://docs.docker.com/engine/install/)
- A terminal (PowerShell on Windows, Terminal on macOS/Linux).
- Curiosity. That is all.

---

## Getting started

1. Read the [Fundamentals overview](./courses/00-fundamentals/README.md).
2. Start with [Lesson 00 - What is Docker?](./courses/00-fundamentals/theory/00-what-is-docker.md) and read the lessons in order.
3. Do the [exercises](./courses/00-fundamentals/exercices/README.md) on your own.
4. Only then, compare your work with the [solutions](./courses/00-fundamentals/exercices/solutions/).

---

## License

See [LICENSE](./LICENSE).
