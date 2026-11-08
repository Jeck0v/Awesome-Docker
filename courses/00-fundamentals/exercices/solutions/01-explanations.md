# Review — Exercise 01: First contact

> **Read this only after you have done the exercise yourself.** This is not an answer sheet. It is a code review: explaining what is good, what is risky, and *why*.

---

## How to read this review

Each comment has a label:

| Label | Meaning |
|---|---|
| **[GOOD]** | Correct and well done. Keep doing it. |
| **[BLOCKER]** | Must be fixed. It is wrong, or it can cause real damage. |
| **[SHOULD FIX]** | Works, but it is a bad habit that will hurt you later. |
| **[SUGGESTION]** | A nicer or more modern way. Optional, but worth learning. |
| **[LEARNING]** | No problem here, just something important to understand. |

---

## The submission under review

```bash
# Task 1 - version
docker -v

# Task 2 - info
docker info

# Task 3 and 4
docker run hello-world
docker run hello-world

# Task 5 and 6
docker ps
docker ps -a

# Task 7
docker images

# Task 8 - clean up
docker rm $(docker ps -aq)
```

Overall: it does what was asked, and it is a good first contact. But there are two things I would not let into a real project, and a few habits worth correcting now, while they are still cheap to change.

---

## Task 1 — `docker -v`

> **[SHOULD FIX]** `docker -v` does not prove that Docker works.

`docker -v` (or `docker --version`) only prints the version of the **client** — the `docker` program on your computer. It **never talks to the daemon**. You could have Docker Desktop completely closed, and this command would still answer happily.

Remember the restaurant from Lesson 03: `docker -v` is like asking the waiter his name. It tells you nothing about whether the kitchen is open.

What the task asked for was **two parts: client and server**. That is `docker version`:

```bash
docker version
```

It returns a `Client` section **and** a `Server` section. The `Server` section only appears if the client managed to reach the daemon. So this command is a real **health check**.

| Command | Talks to the daemon? | What it proves | When to use it |
|---|---|---|---|
| `docker -v` / `docker --version` | No | The CLI is installed | In a script that only needs the client version |
| `docker version` | Yes | Client **and** daemon work, and they agree on an API version | "Does Docker actually work?" |
| `docker info` | Yes | The daemon works, and how it is configured | Diagnosing configuration (storage, cgroups, kernel…) |

> **[LEARNING]** In the `Server` section you will find `Engine`, `containerd`, `runc` and `docker-init`. These are exactly the team members from Lesson 04. When someone tells you "Docker is just one program", you now have proof that it is not.

---

## Task 2 — `docker info`

> **[GOOD]** Correct command.

> **[SUGGESTION]** Do not just run it, **read** it. And learn to extract only what you need.

`docker info` prints a lot. In a real investigation, you rarely want all of it. Docker supports Go templates with `--format`:

```bash
docker info --format '{{.Driver}} | cgroup v{{.CgroupVersion}} | {{.KernelVersion}} | {{.OperatingSystem}}'
```

Example output on Docker Desktop for Windows:

```
overlayfs | cgroup v2 | 6.6.87.2-microsoft-standard-WSL2 | Docker Desktop
```

(Your numbers will differ. Depending on your version and settings, the driver may also read `overlay2`.)

> **[LEARNING]** Look at that kernel version: `...microsoft-standard-WSL2`. You are on Windows, yet the kernel is **Linux**. That is the small Linux VM from Lesson 02. Your containers live there. This one line justifies an entire lesson of theory — this is why I want you to *read* outputs, not just produce them.

| Field | What it tells you | Lesson |
|---|---|---|
| `Storage Driver` | How image layers are stacked (overlay2 / overlayfs) | 01, 04, 05 |
| `Cgroup Version` | Which version of the "budget" system is used | 05 |
| `Kernel Version` | The kernel **shared** by all your containers | 02, 05 |
| `Operating System` | Where the daemon runs (Docker Desktop, Ubuntu, …) | 02 |
| `Containers: Running / Paused / Stopped` | A quick inventory | 03 |

---

## Tasks 3 and 4 — `docker run hello-world` (twice)

> **[GOOD]** Correct, and running it twice was the point.

> **[LEARNING]** What changed between the two runs:

First run:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
```

Second run: straight to `Hello from Docker!`.

Why? On the first run, the daemon did not have the image, so it **pulled** it from Docker Hub. On the second run, the image was already stored locally. This is the caching behaviour from Lesson 03, section 4.

Also notice the name it looked for: `hello-world:latest`, pulled from `library/hello-world`. You wrote `hello-world`; Docker **filled in the defaults**: registry `docker.io`, namespace `library`, tag `latest`.

> **[SHOULD FIX]** Each `docker run` created a **new container**. You now have two stopped containers that you did not need.

`docker run` = **create** a new container **+** start it. It never reuses an old one. Running a short, one-off command twice leaves two corpses behind.

The habit to build from day one is the `--rm` option: "remove the container automatically when it exits":

```bash
docker run --rm hello-world
```

| Option | What happens after the program ends | When to use it |
|---|---|---|
| *(none)* | The container stays, in the `Exited` state | When you want to inspect it afterwards (logs, exit code, files) |
| `--rm` | The container is deleted automatically | One-off commands, tests, experiments — **most of what you do while learning** |

Consequence of not using it: after a few weeks of learning, `docker ps -a` shows dozens of dead containers with random names like `quirky_hopper`. They use disk space and make it hard to find the containers that matter.

---

## Tasks 5 and 6 — `docker ps` and `docker ps -a`

> **[GOOD]** Correct, and you understood the difference.

> **[SUGGESTION]** Prefer the newer "management command" spelling: `docker container ls`.

```bash
docker container ls        # running containers
docker container ls -a     # all containers, including stopped ones
```

`docker ps` is the historic short form, and it still works. But Docker reorganised its commands by **object**: `docker container ...`, `docker image ...`, `docker volume ...`, `docker network ...`. The long form is easier to read and easier to guess:

| Old style | Management command style |
|---|---|
| `docker ps` | `docker container ls` |
| `docker rm` | `docker container rm` |
| `docker images` | `docker image ls` |
| `docker rmi` | `docker image rm` |

You will see both in the wild. Know both, write the clear one.

> **[LEARNING]** Why did task 5 show nothing?

The `hello-world` program prints its message and **ends**. When the main process of a container ends, the container **stops**. A container is **not** a little computer that stays switched on (Lesson 01, section 6): it lives exactly as long as its main process.

Its status reads something like `Exited (0) 2 minutes ago`. The number in brackets is the **exit code** of the program:

| Exit code | Usual meaning |
|---|---|
| `0` | Success — the program finished normally |
| `1`, `2`, … | The program reported an error |
| `137` | Killed with `SIGKILL` (128 + 9) — often a forced stop or out of memory |
| `143` | Ended after `SIGTERM` (128 + 15) — a stop request; not proof on its own that the shutdown was clean |

You will meet 137 and 143 again in Exercises 02 and 05.

---

## Task 7 — `docker images`

> **[GOOD]** Correct. Same remark as above: `docker image ls` is the clearer form.

> **[LEARNING]** Look at the size of `hello-world`: a few **kilobytes**. Compare with what a virtual machine would need (Lesson 02). It is a single tiny program — no operating system, no kernel.

---

## Task 8 — `docker rm $(docker ps -aq)`

> **[BLOCKER]** This removes **every** stopped container on your machine — not just the `hello-world` ones.

The task said: *remove the containers created by `hello-world`.* Your command says: *remove every container Docker knows about.* On a fresh install, the result is the same. On a real machine, this is how people lose a container they were still debugging, together with its logs and the files in its writable layer.

The danger is that the command **looks** precise, but it is not. The right reflex is: **target exactly what you mean, nothing more.**

Better options, from most to least precise:

```bash
# 1. By name or ID, explicitly identified (copy them from 'docker container ls -a')
docker container rm quirky_hopper elated_turing

# 2. Containers created from the hello-world image
docker container ls -a --filter "ancestor=hello-world"         # look first
docker container rm $(docker container ls -aq --filter "ancestor=hello-world")

# 3. All stopped containers, with a confirmation prompt
docker container prune
```

The `ancestor` filter is convenient, but it matches containers based on the given image **or on images built from it**. For a beginner's demonstration this is not a problem; when you need maximum precision, removing explicitly identified IDs remains the safest option.

| Approach | What it removes | Risk |
|---|---|---|
| `docker rm $(docker ps -aq)` | Every stopped container. Without `-f`, Docker refuses to remove running containers, so the command can **partly fail** while still removing the stopped ones | High — removes things you did not mean to target |
| `docker rm -f $(docker ps -aq)` | **Every** container, **including running ones** | Very high — kills running services |
| By name/ID | Exactly what you listed | Lowest |
| `--filter "ancestor=hello-world"` | Containers made from that image (or from images built on it) | Low — targeted |
| `docker container prune` | All stopped containers, after asking for confirmation | Medium — at least it asks |

> **[LEARNING]** That partial failure matters: `docker rm` with several containers is **not atomic**. It handles them one by one, so some can be removed while others fail. After an error, always check what is actually left with `docker container ls -a`.

> **[LEARNING]** "Look before you delete." Run the listing command first, read the list, *then* delete. It costs you three seconds and has saved many careers.

> **[LEARNING]** Bash and PowerShell both interpret `$( ... )` as a command substitution, so this command runs in both — but the two shells do not follow exactly the same rules (splitting of the output, quoting), so behaviour can differ in edge cases. But if the inner command returns nothing, `docker container rm` complains that it needs at least one argument. That error is harmless, but it is a sign you did not check first.

And of course: if you had used `--rm` in tasks 3 and 4, task 8 would have had nothing to do.

---

## Question 6 — running a command with Docker Desktop closed

A good answer looks like this:

```
Cannot connect to the Docker daemon at npipe:////./pipe/docker_engine. Is the docker daemon running?
```

(On Linux the address is `unix:///var/run/docker.sock`.)

In restaurant terms: the **waiter** (client) took your order and walked to the kitchen door, but the **kitchen** (daemon) is closed. The client does nothing by itself. This error also tells you *where* the client is trying to reach the daemon: through a **named pipe** on Windows and a **Unix socket** on Linux — exactly as described in Lesson 03, section 3.3.

---

## The reviewed version

During this exercise, we intentionally keep the containers to inspect their lifecycle. In your own projects, prefer `--rm` for disposable one-off containers.

```bash
# 1. Real health check: client AND server
docker version

# 2. Configuration, then only the useful fields
docker info
docker info --format '{{.Driver}} | cgroup v{{.CgroupVersion}} | {{.KernelVersion}} | {{.OperatingSystem}}'

# 3 and 4. Run the test image (the first time without --rm, to have something to list)
docker run hello-world
docker run hello-world

# 5 and 6. Running containers, then all containers
docker container ls
docker container ls -a

# 7. Local images
docker image ls

# 8. Look, then remove only what we created
docker container ls -a --filter "ancestor=hello-world"
docker container rm $(docker container ls -aq --filter "ancestor=hello-world")

# From now on, for one-off runs:
docker run --rm hello-world
```

---

## Key takeaways

1. **A command that "works" is not the same as a command that proves something.** `docker -v` works; `docker version` proves. Always ask: *what does this output actually demonstrate?*
2. **Read outputs.** The kernel version in `docker info` contained the whole of Lesson 02. Most beginners scroll straight past it.
3. **Clean up as you go** with `--rm` for disposable containers, instead of cleaning up in bulk later — and leave it out when you deliberately want to inspect a container afterwards.
4. **Target precisely when deleting.** Broad delete commands are fine on a fresh laptop and dangerous everywhere else. Build the precise habit now, while the stakes are low.

**Next review:** [Exercise 02 — Your first web server](./02-explanations.md)
