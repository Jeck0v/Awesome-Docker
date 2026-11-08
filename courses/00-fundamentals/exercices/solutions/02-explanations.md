# Review — Exercise 02: Your first web server

> **Read this only after you have done the exercise yourself.** This is a code review, not an answer sheet.

Labels used: **[GOOD]** keep it · **[BLOCKER]** must fix · **[SHOULD FIX]** bad habit · **[SUGGESTION]** nicer way · **[LEARNING]** important to understand.

---

## The submission under review

```bash
# Task 1 - first try
docker run nginx
# (terminal stuck, lots of text... pressed Ctrl+C)

# second try
docker run -d --name my-web -p 80:8080 nginx
# browser: "This site can't be reached"

# third try
docker run -d --name my-web -p 8080:80 nginx
# error: Conflict. The container name "/my-web" is already in use

docker rm -f my-web
docker run -d --name my-web -p 8080:80 nginx
# browser OK

# Task 3 and 4
docker logs my-web
docker logs -f my-web

# Task 5 and 6
docker kill my-web
docker start my-web

# Task 7
docker run -d --name my-web2 -p 8080:80 nginx
# error: port is already allocated

# Task 8
docker rm my-web2
docker run -d --name my-web2 -p 8081:80 nginx

# Task 9
docker rm -f my-web my-web2
```

Honestly, this is a very typical — and very good — learning trail. You hit three classic walls (a blocked terminal, a reversed port mapping, a name conflict) and you got past all three. Let us go through it so that next time you get it right the first time, and above all so that you understand *why*.

---

## Task 1, first try — `docker run nginx`

> **[LEARNING]** Why was your terminal "stuck"?

Without options, `docker run` runs the container in the **foreground** ("attached"): your terminal is connected to the container's output, and the command does not return until the container's main process ends. nginx is a **server**: it is designed to run for ever. So the command never returns.

Pressing `Ctrl+C` sent an interrupt signal to nginx, which stopped — and the container with it.

| Mode | Option | Your terminal | Typical use |
|---|---|---|---|
| Foreground (attached) | *(none)* | Blocked, shows the output | Short commands, or watching a program start up |
| Background (detached) | `-d` / `--detach` | Free immediately; prints the container ID | Services: web servers, databases, APIs |

> **[SHOULD FIX]** This first try **created a container that you never removed**. It has a random name and sits in the `Exited` state. `docker container ls -a` will show it. Clean it up, or next time add `--rm` to experiments.

---

## Task 1, second try — `-p 80:8080`

> **[BLOCKER]** The two port numbers are reversed.

This is *the* most common mistake with Docker. The rule is:

```
   -p  HOST_PORT : CONTAINER_PORT
        your PC      inside the container
        (outside)    (inside)
```

A way to remember it: **"outside : inside"** — you read it from your browser's point of view, going **into** the container.

What you wrote, `-p 80:8080`, means: "connect port **80 of my PC** to port **8080 of the container**." But nginx listens on port **80** inside the container (the image page on Docker Hub says so, and so does the `EXPOSE 80` instruction in its image). Nothing listens on 8080 inside, so the traffic arrives at a closed door. And on your side, the browser was looking at 8080 on your PC, where nothing was published at all.

Correct:

```bash
docker run -d --name my-web -p 8080:80 nginx
```

> **[LEARNING]** How could you have found the container's port without guessing? `docker image inspect nginx --format '{{json .Config.ExposedPorts}}'` returns `{"80/tcp":{}}`. Reading the image's documentation or metadata before running it is a good reflex.

---

## Task 1, third try — "The container name is already in use"

> **[LEARNING]** A failed *idea* still leaves a real container.

Your second try did not "fail" from Docker's point of view: the container was created and started correctly; it simply had the wrong port mapping. It was still there, still called `my-web`. Container names are **unique**, so the third try was refused.

> **[SHOULD FIX]** `docker rm -f` as a reflex.

`-f` (`--force`) removes a running container in one step, killing it (with `SIGKILL` by default) without giving it the graceful shutdown period that `docker stop` allows. On a web server with no data, the consequence is nil. On a database, an abrupt termination can interrupt in-flight operations and require recovery on restart. Never make forced removal your default: use the polite path and keep `-f` for when you *know* it is safe:

```bash
docker container stop my-web
docker container rm my-web
```

---

## Tasks 3 and 4 — logs

> **[GOOD]** `docker logs` and `docker logs -f` are exactly right.

> **[SUGGESTION]** Add `--tail` and, when useful, `--timestamps`.

On a long-running service, `docker logs` prints **everything since the start**. After a few days, that is thousands of lines.

```bash
docker container logs --tail 20 my-web          # only the last 20 lines
docker container logs -f --tail 20 my-web       # last 20 lines, then follow live
docker container logs --since 5m -t my-web      # last 5 minutes, with timestamps
```

> **[LEARNING]** Where do these logs come from? nginx inside the image is configured to write its access log to **standard output** and its error log to **standard error**. Docker captures those two streams (thanks to the shim from Lesson 04). This is the Docker convention: **applications log to stdout/stderr, not to files inside the container.** If an application writes logs to a file, `docker logs` shows nothing.

To leave the live view: `Ctrl+C`. That stops the `logs` command, **not** the container — because you are attached to the log stream, not to the container's main process. Compare with task 1, first try, and make sure you understand the difference.

---

## Tasks 5 and 6 — `docker kill` then `docker start`

> **[SHOULD FIX]** `kill` is not `stop`.

| Command | Behaviour | What happens | Exit code |
|---|---|---|---|
| `docker stop` | Sends the configured stop signal (usually `SIGTERM`), **waits** (10 s by default), then `SIGKILL` if still running | The program can **finish its work**: close connections, flush data to disk | Depends on the process |
| `docker kill` | `SIGKILL` by default | The kernel ends the process **instantly**. No clean-up possible | Generally `137` |
| `docker rm -f` | Kills the container (like `kill`), then removes it | Instant and permanent | — (container gone) |

The exit code is chosen by the **process** and the signal it actually received, not guaranteed by the Docker command. As a rule of thumb, 143 usually corresponds to termination by `SIGTERM` and 137 to `SIGKILL`.

An everyday comparison: `stop` is saying "the shop closes in ten minutes, please finish serving your customers". `kill` is cutting the electricity.

> **[LEARNING]** A nice detail for nginx: its official image declares `STOPSIGNAL SIGQUIT`. For nginx, `SIGQUIT` means "graceful shutdown: finish the current requests, then exit". `docker stop` respects that signal; `docker kill` does not. This is the kind of thing image authors do on purpose — `kill` throws that work away. It also means that after `docker stop`, nginx can exit cleanly with a code other than 143 (typically `0`).

So the right pair for tasks 5 and 6:

```bash
docker container stop my-web
docker container start my-web
```

> **[LEARNING]** What survived the stop/start? **Everything about the container**: its name, its port mapping, its configuration and its writable layer. `start` restarts the *same* container. `run` would create a *new* one. That is the key difference between stopping and removing:

| | Stopped container | Removed container |
|---|---|---|
| Process running? | No | No |
| Configuration (name, ports…) | Kept | Gone |
| Writable layer (changed files) | Kept | Gone |
| Logs | Kept | Gone |
| Can be restarted with `start`? | Yes | No — you must `run` a new one |

---

## Task 7 — "port is already allocated"

> **[GOOD]** You read the error and understood it.

> **[LEARNING]** Why can both containers use port 80 *inside*, but not 8080 *outside*?

Each container has its **own network namespace** (Lesson 05, section 3): its own little network world, with its own port 80. Ten containers can each have a port 80 without seeing each other.

Your PC, however, has **one** port 8080. When the first container published `8080:80`, Docker reserved 8080 on your PC for it. A second reservation of the same host port is impossible — just as two shops cannot have the same street address.

> **[SHOULD FIX]** In this scenario, the failed `docker run` in task 7 left the container `my-web2` behind (in the `Created` state). This is not systematic: depending on when the startup error occurs, Docker may leave a container behind in the `Created` state. Always check `docker container ls -a` after a failed `docker run` before retrying. You correctly removed it before retrying — good.

---

## Task 8 — second container on port 8081

> **[GOOD]** `-p 8081:80` is right: different outside port, same inside port.

---

## Task 9 — `docker rm -f my-web my-web2`

> **[SUGGESTION]** Fine for a demo with no data. For the reasons above, prefer the polite route as a habit:

```bash
docker container stop my-web my-web2
docker container rm my-web my-web2
```

---

## The bonus question — who can reach your web server?

> **[BLOCKER]** (in a real project) — publishing on all network interfaces by accident.

`-p 8080:80` actually means `-p 0.0.0.0:8080:80`: "publish on **every** network interface of the host". On your home network, that means **other devices** (your flatmate's laptop, a smart TV…) can reach `http://<your-ip>:8080`. On a server connected to the internet, it means **the whole internet** — and on Linux, Docker's port publishing writes its own firewall rules, which can bypass a firewall front end such as `ufw` that you thought was protecting you.

To publish **only for your own computer**, add the loopback address in front:

```bash
docker run -d --name my-web -p 127.0.0.1:8080:80 nginx
```

| What you write | Who can reach it | Use it for |
|---|---|---|
| `-p 8080:80` | Every network the host is connected to | A service that is **meant** to be public |
| `-p 127.0.0.1:8080:80` | Only the host itself | Local development, admin interfaces, databases |
| `-P` (capital P) | Every network; Docker picks **random** high host ports for every `EXPOSE`d port | Quick tests when you do not care about the port number (find it with `docker port`) |
| `--network host` | The container uses the host's network directly; no isolation, no mapping | Special cases on Linux. Behaves differently on Docker Desktop — avoid as a beginner |
| *(no `-p` at all)* | Only other containers on the same Docker network | Internal services (for example a database used only by your API) |

The habit to take: **publish as little as possible, as narrowly as possible.** A database container published on `0.0.0.0` with a default password is one of the most common real-world security incidents.

---

## Other points worth mentioning

> **[SHOULD FIX]** No image tag: `nginx` means `nginx:latest`.

Today it gives you one version of nginx; in three months, another. For learning it is acceptable. In a project, **pin the version** (`nginx:1.27` or whichever is current) so that what you tested is what you run. Exercise 04 goes deeper.

> **[SUGGESTION]** Look at the container's details instead of guessing.

```bash
docker container ls --filter "name=my-web"      # status and ports at a glance
docker container port my-web                    # 80/tcp -> 0.0.0.0:8080
```

---

## The reviewed version

```bash
# 1. Background, named, published only on this computer, pinned version
docker run -d --name my-web -p 127.0.0.1:8080:80 nginx:1.27
docker container port my-web

# 2. Browser: http://localhost:8080

# 3 and 4. Logs: recent lines, then live
docker container logs --tail 20 my-web
docker container logs -f --tail 20 my-web      # Ctrl+C stops following, not the container

# 5 and 6. Polite stop, restart the SAME container
docker container stop my-web
docker container start my-web

# 7. Expected to fail: host port 8080 is already taken
docker run -d --name my-web2 -p 127.0.0.1:8080:80 nginx:1.27
docker container ls -a --filter "name=my-web2" # did the failed attempt leave a container?
docker container rm my-web2                    # if so, remove it

# 8. Different host port, same container port
docker run -d --name my-web2 -p 127.0.0.1:8081:80 nginx:1.27

# 9. Clean up politely
docker container stop my-web my-web2
docker container rm my-web my-web2
```

(Use whichever nginx version is current on Docker Hub when you read this; the point is to choose one deliberately.)

---

## Key takeaways

1. **`-p` is "outside : inside".** Write it on a sticky note until it is automatic.
2. **Services run detached (`-d`)**; one-off commands run attached, often with `--rm`.
3. **`stop` before `kill`.** Give programs the chance to shut down cleanly. Exit code 137 in your logs should always be a surprise, never a habit.
4. **Every published port is a door.** Open it only to the people who need it (`127.0.0.1` by default for local work).
5. **A failed `docker run` may leave a container behind**, depending on when the error occurs. Check `docker container ls -a` before retrying.

**Previous review:** [Exercise 01](./01-explanations.md) · **Next review:** [Exercise 03 — Life inside a container](./03-explanations.md)
