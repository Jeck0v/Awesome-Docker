# Review — Exercise 05: Seeing Linux at work

> **Read this only after you have done the exercise yourself.** This is a code review, not an answer sheet.

Labels used: **[GOOD]** keep it · **[BLOCKER]** must fix · **[SHOULD FIX]** bad habit · **[SUGGESTION]** nicer way · **[LEARNING]** important to understand.

---

## The submission under review

```bash
# Part A
docker run alpine uname -a
docker run ubuntu uname -a
docker run debian uname -a
docker run alpine cat /etc/os-release
docker run ubuntu cat /etc/os-release
docker run debian cat /etc/os-release

# Part B
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker exec web1 hostname
docker exec web2 hostname
docker inspect web1          # searched for "IPAddress" by hand
docker inspect web2
docker exec web1 ps
# error: exec: "ps": executable file not found in $PATH
docker exec web1 apt-get update
docker exec web1 apt-get install -y procps
docker exec web1 ps aux
docker top web1

# Part C
docker run -d --name limited --memory 64 --cpus 0.5 --pids-limit 50 nginx
# error: Minimum memory limit allowed is 6MB
docker run -d --name limited --memory 64m --cpus 0.5 --pids-limit 50 nginx
docker exec limited cat /sys/fs/cgroup/memory.max
docker exec limited cat /sys/fs/cgroup/cpu.max
docker exec limited cat /sys/fs/cgroup/pids.max
docker stats

# Bonus
docker run --memory 10m alpine tail /dev/zero
docker ps -a       # Exited (137)

# Clean up
docker rm -f web1 web2 limited
```

This is the most "Linux" exercise of the fundamentals, and you clearly connected it with the theory. The review focuses on three things: **leftover containers** (again), **modifying a running container** — which is a real anti-pattern — and **reading the numbers** properly, because the numbers *are* the lesson.

---

## Part A — one shared kernel

> **[GOOD]** `uname` and `/etc/os-release` are exactly the right sources.

> **[SHOULD FIX]** Six `docker run` without `--rm` = six dead containers.

Every one-off command here should have been `docker run --rm ...`. After Exercises 01 and 03 this should be automatic. I will keep pointing it out until it is.

> **[LEARNING]** What the results prove (question 1).

| Container | `/etc/os-release` says | `uname -r` says |
|---|---|---|
| `alpine` | Alpine Linux 3.x | e.g. `6.6.87.2-microsoft-standard-WSL2` |
| `ubuntu` | Ubuntu 24.04 LTS | **the same** |
| `debian` | Debian GNU/Linux 12 | **the same** |
| `docker info` → Kernel Version | — | **the same** |

Three different distributions. One kernel. The kernel of the host (here, Docker Desktop's Linux VM).

So what does an "ubuntu" image contain? **Ubuntu's userland**: its files, its libraries, its tools (`bash`, `apt`…), its `/etc/os-release`. **Not** Ubuntu's kernel. There is no kernel in any image. That one fact explains:

- why containers start in milliseconds (nothing to boot — Lesson 02);
- why Linux containers need a Linux kernel, hence the VM on Windows/macOS (Lesson 02, section 5);
- why a vulnerability in the shared kernel can potentially affect the host and compromise isolation between containers (Lesson 02, isolation).

> **[SUGGESTION]** `uname -a` prints a lot (it includes the hostname, which changes every time and adds noise). `uname -r` gives just the kernel release — the only thing you wanted to compare. Ask for exactly what you need.

---

## Part B — hostnames and IP addresses

> **[GOOD]** `docker exec <name> hostname` is correct.

> **[LEARNING]** Each container's hostname is its **short container ID** — different for `web1` and `web2`, although they come from the same image. That is the **UTS namespace**. Same image, different identity.

> **[SHOULD FIX]** Searching the whole `docker inspect` output by hand for the IP address.

As in Exercise 04: ask for the field. The IP lives under the container's networks:

```bash
docker container inspect web1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker container inspect web2 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Typical result: `172.17.0.2` and `172.17.0.3`. Each container has its **own network namespace**, its own virtual network card (one end of a **veth pair**), attached to the default **bridge** network (Lesson 05, section 7). Those addresses are private: they exist inside Docker's network, not on your home network.

(The `range` is needed because a container can be attached to several networks.)

---

## Part B — `ps` not found, then `apt-get install` inside a running container

> **[LEARNING]** Why was `ps` missing?

The default variant of the official `nginx` image is based on a **slim** Debian (other official variants exist, such as Alpine-based tags — the base depends on the tag you choose). Mainly to keep the image small, and also to reduce what an attacker could use, it does not include `procps`, the package that provides `ps`. Exercise 03 told you: **never assume a tool exists in an image.**

> **[BLOCKER]** Installing packages inside a running container to debug it.

It "worked", but in a real environment it is a serious anti-pattern:

| Problem | Why it matters |
|---|---|
| **The change is invisible** | Nobody reading the Dockerfile knows that container now has extra software. It no longer matches its image |
| **It is temporary anyway** | It lives in the writable layer. The next time the container is recreated, it is gone. You "fixed" nothing |
| **It changes what you are debugging** | Installing packages modifies files, uses memory and network. You are no longer observing the original situation |
| **It may be impossible** | Many production images run as non-root, have read-only filesystems, have no package manager or no internet access |
| **Security** | You just downloaded and ran new software inside a service, by hand |

Remember Lesson 01: containers are **paper plates**. You do not repair them; you replace them. And you do not modify them to look inside.

### Better ways to see processes — from best to acceptable

**1. From the host side, with Docker** (this was task 6):

```bash
docker container top web1
```

**2. With a temporary "debugging" container that joins the target's PID namespace** — the professional trick:

```bash
docker run --rm -it --pid container:web1 alpine ps
```

This starts a throwaway `alpine` container (which has `ps`) **inside web1's PID namespace**. It sees web1's processes without changing a single file in web1. It is also a beautiful demonstration of Lesson 05: namespaces are **separate building blocks** — you can share one (PID) while keeping the others (filesystem) separate.

**3. Without any tool at all**, because Linux exposes processes as folders in `/proc`:

```bash
docker container exec web1 ls /proc
```

Every number in the list is a PID. That is what `ps` reads anyway.

(Docker Desktop also offers `docker debug` for the same purpose, depending on your subscription and version.)

---

## Part B — comparing PIDs (question 3)

> **[LEARNING]** Here is what you should have seen:

| Seen from | nginx master process | nginx worker processes |
|---|---|---|
| Inside the container (`--pid container:web1 ... ps`) | PID **1** | 2, 3, … (small numbers) |
| Docker's host-side view (`docker container top web1`) | A large number, e.g. 4038 | Other large numbers |

It is **the same process**, not a copy. One process, two names — one per PID namespace. Inside, nginx thinks it is the first program on the machine. The kernel knows better.

> **[LEARNING]** On Docker Desktop, the "host" PIDs shown by `docker top` are the PIDs **inside the Linux VM**, not Windows process IDs. You will not find nginx in the Windows Task Manager — another reminder that your containers live in the VM.

---

## Part C — the limits

> **[GOOD]** You read the error and fixed the unit.

> **[LEARNING]** Why `--memory 64` failed: without a unit, Docker reads the number as **bytes**. 64 bytes is absurd, and Docker refuses anything below 6 MB. Units matter: `b`, `k`, `m`, `g`. **Always** check units in resource limits — a missing `m` or an extra `g` is a classic production incident.

> **[GOOD]** The three cgroup files are the right ones. Let us read the values properly (question in task 8):

| File | Value you should see | What it means |
|---|---|---|
| `memory.max` | `67108864` | 64 × 1024 × 1024 bytes = **64 MiB**. When the limit is reached, the kernel attempts to reclaim memory. If it cannot reclaim enough, a cgroup OOM event may occur and the kernel may kill one or more processes |
| `cpu.max` | `50000 100000` | "In every period of **100 000 µs** (0.1 s), this group may use **50 000 µs** of CPU time" = **half a CPU** |
| `pids.max` | `50` | At most 50 processes/threads in this container. The 51st fails to start |

The essential point: **Docker only wrote these numbers into kernel files.** The kernel enforces them. This is Lesson 05, section 4, seen with your own eyes.

> **[SUGGESTION]** You can confirm from the outside too, which is how you would check a container you are not allowed to enter:

```bash
docker container inspect limited --format 'mem={{.HostConfig.Memory}} cpus(nano)={{.HostConfig.NanoCpus}} pids={{.HostConfig.PidsLimit}}'
# mem=67108864 cpus(nano)=500000000 pids=50
```

> **[LEARNING]** Why each flag, and what the alternatives would do:

| Flag | What it controls | Alternative | Difference / consequence |
|---|---|---|---|
| `--memory 64m` | Hard memory ceiling | *(none)* | No limit: the container can use **all** the memory of the host (or VM) and push other services into trouble |
| `--cpus 0.5` | Hard CPU ceiling (a **quota**) | `--cpu-shares 512` | Shares are only a **relative priority** when the CPU is busy; when the CPU is idle, the container can still use everything |
| `--cpus 0.5` | | `--cpuset-cpus 0` | Pins the container to **specific cores** (CPU affinity), rather than limiting the amount (quota). `--cpuset-cpus 0` allows it to run only on logical CPU 0, but does not guarantee it 100 % of that CPU: other processes can use it too |
| `--pids-limit 50` | Maximum processes | *(none)* | A runaway program (or a "fork bomb") can create processes until the whole machine freezes |

---

## Task 9 — `docker stats`

> **[GOOD]** Right command.

> **[SUGGESTION]** `docker stats` keeps refreshing until you press `Ctrl+C`. For a snapshot you can copy into your notes, use:

```bash
docker container stats --no-stream
```

> **[LEARNING]** Look at the `MEM USAGE / LIMIT` column:

- `limited` shows `... / 64MiB` — your cgroup limit;
- `web1` and `web2` show `... / 7.6GiB` (or similar) — that is **not** a limit you set. It is the total memory available to the Linux VM (or the host on Linux). In other words: **no limit** (question 5).

On a shared server, a container without a memory limit can consume everything. When the whole machine runs out of memory, the kernel's OOM killer picks a victim — and it might not be the container that caused the problem. That is why limits are standard practice in production (and mandatory in many Kubernetes clusters).

---

## Bonus — hitting the memory limit

> **[GOOD]** Good choice of command: `tail /dev/zero` reads an endless stream with no line breaks, so `tail` keeps buffering until memory runs out.

> **[SHOULD FIX]** Two gaps in the evidence:

**1. You only saw `Exited (137)`, you did not prove *why*.** 137 = 128 + 9 = `SIGKILL`. Many things send `SIGKILL` (including `docker kill`). Docker records whether it was the OOM killer:

```bash
docker container inspect <container> --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}'
# exit=137 oom=true
```

Never stop at "it looks like X". **Prove X.** That is the difference between guessing and debugging.

**2. Swap can blur the experiment.** When you set `--memory` but not `--memory-swap`, Docker allows the container to use an equal amount of **swap** on top (if the host has swap). The container then slows down before being killed. For a clean experiment, make both equal, which means "no swap":

```bash
docker run --name oom-test --memory 10m --memory-swap 10m alpine tail /dev/zero
docker container inspect oom-test --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}'
docker container rm oom-test
```

> **[LEARNING]** Question 4 — CPU versus memory, and who decides.

| Limit reached | What happens | Why |
|---|---|---|
| **CPU** | The process is **throttled**: it waits for the next period. Slower, but alive | CPU time is renewable. Waiting costs nothing but time |
| **Memory** | The kernel's **OOM killer** sends `SIGKILL` → exit code 137 | Memory cannot be "waited for". The kernel must free it **now** |

And it is a **kernel** decision, not a Docker decision. Docker wrote `memory.max`; the kernel killed the process. Docker only *reported* what happened.

---

## Clean-up

> **[SHOULD FIX]** `docker rm -f` again, and incomplete: the six Part A containers and the bonus container are still there.

```bash
docker container ls -a                       # identify the leftovers and note their IDs
docker container stop web1 web2 limited      # still running: stop them politely
docker container rm web1 web2 limited
docker container rm <id1> <id2> <id3> <id4> <id5> <id6> <bonus_id>   # Part A + bonus, already stopped
```

Remove the leftovers **explicitly, by ID**. `docker container prune` is not a targeted clean-up: it removes **every** stopped container on the Docker Engine, not only the ones from this exercise — looking at the list first does not make it selective.

---

## The reviewed version

```bash
# --- Part A: one shared kernel ---
docker run --rm alpine uname -r
docker run --rm ubuntu uname -r
docker run --rm debian uname -r
docker info --format '{{.KernelVersion}}'
docker run --rm alpine cat /etc/os-release
docker run --rm ubuntu cat /etc/os-release
docker run --rm debian cat /etc/os-release

# --- Part B: namespaces ---
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker container exec web1 hostname
docker container exec web2 hostname
docker container inspect web1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker container inspect web2 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker run --rm --pid container:web1 alpine ps     # inside view: nginx is PID 1
docker container top web1                          # host view: large PIDs

# --- Part C: cgroups ---
docker run -d --name limited --memory 64m --cpus 0.5 --pids-limit 50 nginx
docker container exec limited cat /sys/fs/cgroup/memory.max   # 67108864
docker container exec limited cat /sys/fs/cgroup/cpu.max      # 50000 100000
docker container exec limited cat /sys/fs/cgroup/pids.max     # 50
docker container stats --no-stream

# --- Bonus: the OOM killer, proven ---
docker run --name oom-test --memory 10m --memory-swap 10m alpine tail /dev/zero
docker container inspect oom-test --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}'

# --- Clean up ---
docker container stop web1 web2 limited
docker container rm web1 web2 limited oom-test
docker container ls -a
```

---

## Question 6 — why Linux matters (a model answer)

Two examples of a good answer:

- **Volumes and permissions.** When a container writes into a folder mounted from the host, Linux compares **UID/GID numbers**, not user names. Without understanding users, groups and `chmod`/`chown`, the error "Permission denied" — or files on your laptop suddenly owned by root — is a mystery.
- **Graceful shutdown.** `docker stop` sends the configured stop signal to the container's main process. By default this is `SIGTERM`, but the image can set another one with `STOPSIGNAL` (the official nginx image uses `SIGQUIT`), and `--stop-signal` can override it. If the process does not exit within the grace period (10 seconds by default on Linux), Docker sends `SIGKILL`. If the program is PID 1 and does not handle its stop signal (or a shell script in front of it does not forward it), the container waits for the whole grace period and is then killed with `SIGKILL` — possibly in the middle of writing data. Without knowing signals and the special role of PID 1, you cannot even see that this is happening.

Any two topics from the table in Lesson 05, section 9, explained with the same "Docker symptom → Linux cause" logic, is a good answer.

---

## Key takeaways

1. **There is no kernel in an image.** Every container on a host shares one. Say it until it is obvious.
2. **Never modify a running container to debug it.** Look from outside (`top`, `inspect`, `stats`) or bring a throwaway debugging container that shares the right namespace.
3. **Limits are numbers in kernel files.** Docker writes them; the kernel enforces them. Check your **units**.
4. **CPU limit → slow. Memory limit → killed (137).** And prove an OOM kill with `OOMKilled`, do not assume it.
5. **Everything you observed today is Linux.** Namespaces, cgroups, `/proc`, signals, exit codes. This is why the next step after the fundamentals is not "more Docker commands" — it is **more Linux**. See Lesson 05, section 9, for the learning path.

**Previous review:** [Exercise 04](./04-explanations.md) · **Back to:** [Exercises](../README.md)
