# Review — Exercise 03: Life inside a container

> **Read this only after you have done the exercise yourself.** This is a code review, not an answer sheet.

Labels used: **[GOOD]** keep it · **[BLOCKER]** must fix · **[SHOULD FIX]** bad habit · **[SUGGESTION]** nicer way · **[LEARNING]** important to understand.

---

## The submission under review

```bash
# Task 1
docker run -it --name playground alpine bash
# error: exec: "bash": executable file not found in $PATH

docker run -it --name playground alpine sh
# error: Conflict. The container name "/playground" is already in use

docker rm playground
docker run -it --name playground alpine sh

# Task 2 (inside)
whoami          # root
hostname        # 3f2a9c1e7b4d
ps              # PID 1 = sh, and ps
echo "Docker is fun" > /root/hello.txt
exit

# Task 3
docker ps -a    # playground: Exited (0)

# Task 4
docker start playground
# prints "playground" and returns... no shell?
docker exec -it playground sh
cat /root/hello.txt   # Docker is fun
exit

# Task 5 and 6
docker attach playground
# pressed Ctrl+P then Ctrl+Q -> back on my PC, container still running
docker exec -it playground sh
ps
exit

# Task 7
docker ps       # playground: Up

# Task 8
docker run -it alpine sh
ls /root        # empty
exit

# Task 9
docker run --rm -it alpine sh
exit

# Task 10
docker run -it ubuntu bash
exit

# Task 11
docker rm -f playground
```

There is real understanding in this attempt — you discovered `exec`, you found the detach keys, and you used `--rm`. Let me go through the bumps, because each one hides an important idea.

---

## Task 1 — `bash` not found, then a name conflict

> **[LEARNING]** Why is there no `bash` in `alpine`?

Alpine Linux is a **minimal** distribution designed to be tiny (a few megabytes). Its shell is `sh`, provided by **BusyBox** — a single small program that plays the role of `sh`, `ls`, `ps`, `cat` and many other commands. `bash` is simply not installed.

This is your first contact with a big Lesson 05 idea: **an image contains a distribution's userland (its tools and files), and different images contain different tools.** Never assume a command exists inside a container.

Typical official base images (not guaranteed for every variant or custom image):

| Image | Default shell | Has `bash`? | Package manager |
|---|---|---|---|
| `alpine` | `sh` (BusyBox) | No | `apk` |
| `debian`, `ubuntu` | `bash` (and `sh`) | Yes | `apt` |
| "distroless" / `scratch` images | **none** | No | none |

> **[LEARNING]** Why did the second attempt say the name was in use?

Even though `bash` failed to start, Docker had already **created** the container `playground` before trying to run `bash` in it. It was left in the `Created` state. So the name was taken. (You saw the same effect with ports in Exercise 02.)

> **[GOOD]** You read the error, removed the leftover, and retried. That is exactly the right reaction.

---

## Task 1 — the options `-i` and `-t`

> **[GOOD]** `-it` is correct for an interactive shell. But do you know what each letter does?

| Option | Long form | What it does | Without it… |
|---|---|---|---|
| `-i` | `--interactive` | Keeps the container's **standard input** open, so what you type reaches the program | The process cannot receive interactive input through Docker's stdin. Depending on the program, it may exit immediately when stdin is unavailable or closed (a shell typically does) |
| `-t` | `--tty` | Gives the container a **pseudo-terminal**: a prompt, line editing, colours | You can type, but there is no prompt, and the output looks raw and confusing |
| `-it` | both | A normal interactive terminal | — |

When do you use only one of them? `-i` alone is used when you **pipe** data into a container (for example `cat file.sql | docker run -i postgres psql`) — a TTY would get in the way there. For a human typing in a shell: `-it`.

---

## Task 2 — inside the container

> **[GOOD]** All four commands are right. Now let us look at what the answers *mean*.

**`whoami` → `root`.** You are root inside the container. As Lesson 05, section 6 explains, without user namespace remapping, UID 0 inside the container maps to **UID 0 on the host**. However, this does not grant unrestricted host root privileges: capabilities, namespaces, seccomp and other security mechanisms still constrain the process. That is why, later, you will learn to make images run as a **non-root user**.

**`hostname` → `3f2a9c1e7b4d`.** It is the **short ID of the container** (the first 12 characters). Run `docker container ls` and you will see the same value in the `CONTAINER ID` column. It comes from the **UTS namespace**: each container has its own hostname.

**`ps` → PID 1 is `sh`.** Inside the container, your shell is process number **1**, the first process of its own little world. That is the **PID namespace**. On the host, the very same `sh` has a completely different number.

> **[LEARNING]** The rule that explains the rest of this exercise: **a container's lifecycle is tied to its main process. When that process exits, Docker considers the container stopped** and marks it as `Exited`. The main process is usually PID 1 inside the container. It is not necessarily a shell — it can be nginx, python, node, tini… In this exercise, it is the shell.

---

## Task 3 — `exit` stopped the container

> **[LEARNING]** Why?

Your shell **was** the main process (PID 1). Typing `exit` ended the shell; the runtime saw the main process end, and Docker considered the container stopped. Status `Exited (0)` — 0 because the shell ended normally.

---

## Task 4 — `docker start` "did nothing"

> **[SHOULD FIX]** `docker start` does not attach your terminal by default.

`docker start playground` **did** start the container — a new `sh` process was running, as PID 1, in the background. But your terminal was not connected to it, so it looked like nothing happened.

| Command | What it does |
|---|---|
| `docker start playground` | Starts the container in the background, terminal not connected |
| `docker start -ai playground` | Starts a new instance of the container's original command **and** **a**ttaches your terminal **i**nteractively. The previous shell process is gone, but the container's writable layer is preserved |

You then used `docker exec`, which worked — but for a different reason (see below). The clean answer to task 4 was:

```bash
docker container start -ai playground
cat /root/hello.txt      # Docker is fun
```

> **[LEARNING]** Why did the file survive? Because `start` restarts the **same** container, with the **same writable layer** (Lesson 01, section 3). Stopping a container does not touch its writable layer. Only **removing** the container deletes it. Note the distinction: the container's **files** persist, but not the **process** — the memory of the previous shell (its variables, its history) is gone.

---

## Tasks 5, 6 and 7 — attach, detach, exec

> **[GOOD]** `Ctrl+P` then `Ctrl+Q` is exactly the right way to leave without stopping. Well found.

This is the most important distinction of the exercise, so let us make it very clear.

| | `docker attach` | `docker exec -it ... sh` |
|---|---|---|
| What it does | Connects your terminal to the **main process's standard input/output streams**. It starts nothing new | Starts a **new, extra** process inside the running container |
| Number of shells after it | Still one (you are connected to the main process's streams) | One more (the original main process is still there) |
| If you type `exit` | The main process (the shell) ends → **the container stops** | You end only **your extra shell** → the container **keeps running** |
| To leave without stopping | `Ctrl+P`, `Ctrl+Q` | Just `exit` |
| Analogy | Taking the driver's seat of a car that is already moving | Getting into the passenger seat |

That is why in task 7 the container was still `Up`: you left an **exec** shell, not the main process.

And look at the `ps` output in task 6: there are **two** `sh` processes now — PID 1 (the original shell) and your exec shell with a higher number. That is the proof that `exec` adds a process.

> **[SUGGESTION]** `docker exec` is the tool you will use most in real life, to inspect a running service. `docker attach` is rarely useful on services: it connects you to the server's streams, and when attached, pressing `Ctrl+C` can send `SIGINT` to the container's main process and stop it. The exact behaviour depends on signal proxying (`--sig-proxy`) and on the TTY configuration. Use `Ctrl+P` followed by `Ctrl+Q` to detach without stopping the container. Learn attach, but **reach for exec**.

---

## Task 8 — the new container has no file

> **[GOOD]** Correct observation.

> **[SHOULD FIX]** This container has **no name** and **no `--rm`**. It is now sitting in `Exited` state with a random name like `vigilant_curie`. You never cleaned it up (task 11 only removes `playground`).

> **[LEARNING]** Why is the file missing? `docker run` created a **new** container from the **image**. The image is read-only and was never changed; your file only ever existed in `playground`'s private writable layer.

```
   alpine image (read-only, unchanged)
        |                         |
   playground                 new container
   writable layer:            writable layer:
   /root/hello.txt            (empty)
```

The consequence for real life (question 6): **anything precious written inside a container is one `docker rm` away from disappearing.** A database container must store its data in a **volume** (or a bind mount), which lives outside the container's writable layer and survives its removal. Volumes have their own course; for now, remember Lesson 01: *program inside, precious data outside*.

---

## Task 9 — `--rm`

> **[GOOD]** Correct.

> **[SUGGESTION]** You did not *check* that it left nothing behind, which was part of the task. Always verify:

```bash
docker container ls -a      # the --rm container must not appear
```

| Option | After exit | Good for |
|---|---|---|
| *(none)* | Container stays (`Exited`) | When you want to restart it or inspect it afterwards (like `playground`) |
| `--rm` | Container deleted automatically | Experiments and one-off commands |
| `--name` | Gives a predictable name | Anything you will refer to again |

Note that `--rm` and `--name` work fine together. In this exercise, `playground` was deliberately kept (no `--rm`) because we needed to restart it.

---

## Task 10 — `bash` in `ubuntu`

> **[GOOD]** It works in `ubuntu`, as expected.

> **[SHOULD FIX]** Again no `--rm`. One more leftover container.

> **[LEARNING]** Run `cat /etc/os-release` in both — they report different distributions. Then run `uname -r` in both: **the same kernel**. Ubuntu's tools, Alpine's tools, one shared kernel. Exercise 05 builds on exactly this.

---

## Task 11 — clean-up

> **[SHOULD FIX]** The clean-up is incomplete, and uses `-f` by reflex.

`docker rm -f playground` removed only `playground`. The anonymous containers from tasks 8 and 10 are still there. A complete clean-up starts by **looking**:

```bash
docker container ls -a                    # identify the real IDs/names of your leftovers
docker container stop playground          # still running: stop it politely first
docker container rm playground <container_id> <container_id>
```

The containers from tasks 8 and 10 are already stopped, so they need no `stop` before `rm`. Use the IDs or names actually shown by `docker container ls -a` — they are random, so never copy them from someone else's notes.

Alternatively, `docker container prune` removes **every** stopped container on your machine — not only the ones from this exercise — after asking for confirmation. Only use it if you have checked that list. The real fix, though, is upstream: **`--rm` on every throwaway container**, and there is nothing left to clean.

---

## The reviewed version

```bash
# 1. Interactive shell. Alpine has sh, not bash.
docker run -it --name playground alpine sh

# 2. Inside
whoami                                   # root (UID 0)
hostname                                 # short container ID (UTS namespace)
ps                                       # PID 1 = sh (PID namespace)
echo "Docker is fun" > /root/hello.txt
exit                                     # main process ends -> container stops

# 3. Status
docker container ls -a --filter "name=playground"     # Exited (0)

# 4. Same container, same writable layer, new shell process, terminal attached
docker container start -ai playground
cat /root/hello.txt                      # still there

# 5. Leave WITHOUT stopping: Ctrl+P, Ctrl+Q

# 6. A second, extra shell in the same container
docker container exec -it playground sh
ps                                       # two sh processes now
exit                                     # ends only the exec shell

# 7. Still running
docker container ls --filter "name=playground"        # Up

# 8. A brand-new container: no file, and cleaned automatically
docker run --rm -it alpine sh -c "ls -la /root"

# 9. Throwaway container, then verify
docker run --rm -it alpine sh
docker container ls -a

# 10. bash exists in ubuntu, not in alpine
docker run --rm -it ubuntu bash

# 11. Clean up politely
docker container stop playground
docker container rm playground
docker container ls -a                   # verify: nothing left
```

---

## Key takeaways

1. **A container's lifecycle is tied to its main process.** When it exits, Docker considers the container stopped. Almost every "why did my container stop?" question comes back to this.
2. **`exec` adds a process; `attach` connects your terminal to the main process's streams.** Use `exec` to look inside running services.
3. **stop ≠ remove.** Stopping keeps the writable layer; removing deletes it. Precious data goes in volumes.
4. **Do not assume tools exist in an image.** `bash`, `ps`, `curl`, `vim`… minimal images often do not have them, and that is a *good* thing for size and security.
5. **`--rm` is a habit, not an option.** Every container you start without a reason to keep it should have it.

**Previous review:** [Exercise 02](./02-explanations.md) · **Next review:** [Exercise 04 — Images, tags and spring cleaning](./04-explanations.md)
