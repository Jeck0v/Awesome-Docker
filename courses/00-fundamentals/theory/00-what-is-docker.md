# Lesson 00 — What Is Docker?

> **In one sentence:** Docker is a tool that packs a program *and everything it needs* into a little box, so that the program runs the same way on every computer.

---

## What you will learn

By the end of this lesson you will be able to:

- explain the problem Docker solves, using an everyday example;
- say what an **image**, a **container**, a **Dockerfile** and a **registry** are;
- describe what Docker is **not**, so you do not get confused later.

---

## 1. The problem: "But it works on my machine!"

Imagine you bake a delicious chocolate cake at home. Your friend asks for the recipe, so you write it down and send it.

Your friend tries it… and the cake is a disaster.

Why? Because:

- your oven heats to 180 °C, but theirs runs a bit hotter;
- you used a special brand of flour they cannot find;
- you forgot to write down that you added a pinch of salt;
- they live high up in the mountains, where water boils differently.

The **recipe** was the same, but the **kitchen** was different.

Software has *exactly* the same problem. A program written on one computer often breaks on another computer because:

| In the kitchen | On a computer |
|---|---|
| A different oven | A different operating system (Windows, macOS, Linux) |
| A different brand of flour | A different version of a library (for example Python 3.9 instead of 3.12) |
| A forgotten pinch of salt | A setting or file that only exists on the developer's laptop |
| Living in the mountains | A server that is configured differently |

Developers have a famous, slightly embarrassed phrase for this:

> *"But it works on my machine!"*

**Docker was created to make that sentence disappear.**

---

## 2. The solution: ship the whole kitchen, not just the recipe

What if, instead of sending your friend the recipe, you could send them a **magic lunchbox** that contains:

- the cake recipe,
- the exact flour,
- the exact oven,
- the exact pinch of salt,
- and even the right mountain air?

Your friend opens the lunchbox, presses a button, and gets **exactly the same cake as you**. Every time. In every house.

That magic lunchbox is what Docker gives you for software.

```
   WITHOUT DOCKER                         WITH DOCKER
 +------------------+               +--------------------------+
 |  your program    |               |  a "container"           |
 +------------------+               |  +--------------------+  |
   + "please install                |  | your program       |  |
     Python 3.12,                   |  | + Python 3.12      |  |
     these 14 libs,                 |  | + the 14 libraries |  |
     set this var..."               |  | + the settings     |  |
                                    |  +--------------------+  |
   -> might break                   +--------------------------+
                                      -> runs the same everywhere
```

---

## 3. Why is it called "Docker"?

Think of a big port by the sea, where huge ships are loaded and unloaded. The people who work there are called **dockers** (or dock workers).

Before the 1950s, dockers had to load every sack, barrel and crate *one by one*. Every object had a different shape. It was slow, and things got broken or lost.

Then someone invented the **shipping container**: a standard metal box, always the same size. It does not matter whether it holds bananas, cars or teddy bears — every crane, ship, lorry and train in the world knows how to carry it.

```
       +---------+ +---------+ +---------+
       | bananas | |  cars   | | teddies |   <- different contents
       +---------+ +---------+ +---------+   <- SAME box shape
   ~~~~~~~~~~~~~~~~~~~ ship ~~~~~~~~~~~~~~~~~~~
```

Docker does the same thing for programs:

- the **contents** can be anything (a website, a database, a game server…);
- the **box** is always the same, so any computer running Docker knows how to start it.

---

## 4. The four key words

There are four words you will hear **all the time**. Learn them now and the rest of the course will be much easier.

### 1. Dockerfile — *the recipe*

A **Dockerfile** is a simple text file with step-by-step instructions, like a recipe:

> "Start from a small Linux. Install Python. Copy my program inside. When someone starts the box, run my program."

You write it once.

### 2. Image — *the frozen ready meal*

When Docker follows the recipe, it produces an **image**.

An image is like a **frozen ready meal** in the freezer: everything is prepared and packed, but nobody is eating it yet. It does not *do* anything on its own. It just waits.

An important detail: an image is **read-only**. You cannot change a frozen meal once it is packed — you can only make a new one.

### 3. Container — *the meal on the plate*

When you **start** an image, you get a **container**. It is the meal, heated up and on the plate, actually being eaten.

From **one** image you can start **many** containers, just as one recipe can feed many people.

```
            Dockerfile   (the recipe)
                |
                |  "docker build"  = follow the recipe
                v
              Image      (frozen, read-only, can be shared)
                |
                |  "docker run"    = heat it up and serve it
        +-------+-------+
        v       v       v
   Container Container Container   (running, alive)
```

### 4. Registry — *the supermarket*

Where do you get images from? From a **registry**. A registry is like a **supermarket of images**: you can take images off the shelf (**pull**) or put your own on the shelf (**push**).

The biggest public registry is **Docker Hub** (<https://hub.docker.com>). There you will find official images for almost everything: `nginx` (a web server), `postgres` (a database), `python`, `node`, `alpine` (a tiny Linux)…

---

## 5. A tiny bit of history

| Year | What happened |
|---|---|
| 2008 | Linux gains **LXC**, a way to run "Linux containers". Powerful, but hard to use. |
| 2013 | A company called **dotCloud** shows **Docker** to the world at a conference (PyCon). It makes containers *easy*. Developers love it. |
| 2015 | Big companies create the **Open Container Initiative (OCI)** so that containers follow shared, open rules — not just Docker's own rules. |
| Today | Docker is one of the most common tools in software development. Containers run everywhere: laptops, data centres, the cloud. |

> **Key idea:** Docker did not invent containers. It made them **simple** and **friendly** for everyone. That is why it became so popular.

---

## 6. What Docker is **not**

Beginners often mix things up. Let us clear up the confusion straight away.

| Myth | Reality |
|---|---|
| "Docker is a virtual machine." | No. A container is much lighter than a virtual machine. We compare them carefully in [Lesson 02](./02-containers-vs-virtual-machines.md). |
| "Docker is a programming language." | No. You do not *write programs* in Docker. You *package and run* programs written in any language. |
| "Docker makes my program faster." | Not really. It makes your program **predictable** and **easy to move**, not magically faster. |
| "Everything inside a container is saved for ever." | No. When you delete a container, the changes you made inside it are gone. We will see how to keep data later (volumes). |
| "Docker only works on Linux." | Containers *need* a Linux kernel, but **Docker Desktop** lets you use them on Windows and macOS too, by quietly running a small Linux in the background. More on this in [Lesson 05](./05-linux-kernel-primitives.md). |

---

## 7. Why people love Docker

- **Same everywhere** — it runs the same on your laptop, your colleague's laptop and the server.
- **Fast** — a container usually starts in less than a second.
- **Light** — you can run dozens of containers on an ordinary laptop.
- **Clean** — try a database, then delete it. Nothing is left lying around on your computer.
- **Building blocks** — a website, its database and its cache can each live in their own container and talk to each other.
- **Easy to share** — push an image to a registry, and anyone can run it with one command.

---

## Remember

- Docker solves the **"it works on my machine"** problem.
- **Dockerfile** = recipe · **Image** = frozen meal · **Container** = meal on the plate · **Registry** = supermarket.
- One image → many containers.
- Docker is **not** a virtual machine.

---

## Quick quiz

1. In your own words, what problem does Docker solve?
2. What is the difference between an image and a container?
3. Can you start three containers from the same image?
4. Where would you go to find an official image for a web server?

<details>
<summary>Show the answers</summary>

1. A program that works on one computer often breaks on another because the environment is different. Docker packs the program **together with** its environment, so it runs the same everywhere.
2. An **image** is the packed, read-only template (the frozen meal). A **container** is a running instance of that image (the meal on the plate).
3. Yes. One image can produce as many containers as you like.
4. A registry, for example **Docker Hub** (<https://hub.docker.com>).

</details>

---

**Next lesson:** [01 — Containerisation](./01-containerization.md)
