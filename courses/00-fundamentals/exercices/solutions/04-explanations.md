# Review — Exercise 04: Images, tags and spring cleaning

> **Read this only after you have done the exercise yourself.** This is a code review, not an answer sheet.

Labels used: **[GOOD]** keep it · **[BLOCKER]** must fix · **[SHOULD FIX]** bad habit · **[SUGGESTION]** nicer way · **[LEARNING]** important to understand.

---

## The submission under review

```bash
# Task 1
docker pull alpine
docker pull alpine:3.20
docker pull debian:bookworm-slim
docker pull ubuntu

# Task 2 and 3
docker images

# Task 4
docker history nginx

# Task 5
docker inspect alpine
# (scrolled through ~150 lines of JSON looking for things)

# Task 6
docker images --digests

# Task 7
docker pull alpine
# "Image is up to date for alpine:latest"

# Task 8
docker rmi alpine:3.20
docker run --name keep-me nginx true
docker rmi nginx
# error: conflict: unable to remove repository reference "nginx" ... container is using its referenced image
docker rmi -f nginx

# Task 9 and 10
docker system prune -a --volumes
# "Are you sure you want to continue? [y/N]" -> y
```

The exploration in tasks 1 to 7 is good. The ending — tasks 8 to 10 — is where things go wrong, because it contains the two most expensive habits in this whole course: **forcing** and **mass-deleting without reading**. Let us go step by step.

---

## Task 1 — pulling images

> **[GOOD]** `alpine:3.20` and `debian:bookworm-slim`: you chose explicit tags. Good.

> **[SHOULD FIX]** `docker pull ubuntu` — no tag, so you got `ubuntu:latest`.

The task asked for a **specific version tag**. `ubuntu:latest` is whatever the Ubuntu maintainers currently point `latest` at. Today it is one release; after the next LTS comes out, it will silently become another — with a different version of every library inside.

```bash
docker pull ubuntu:24.04       # or the current LTS release listed on Docker Hub
```

> **[LEARNING]** `latest` is **only a label** (Lesson 03, section 3.4). It means "the tag used when nobody specifies one". It does **not** mean "newest", and it is not checked by anyone. Some projects even leave `latest` pointing to an old version.

> **[LEARNING]** `bookworm` is a **codename**: Debian 12. Debian's next release (Debian 13) is called `trixie`. Codename tags are a common convention. `-slim` means "with documentation and rarely used files removed".

---

## Tasks 2 and 3 — sizes and image IDs

> **[GOOD]** `docker images` is correct (the clearer modern form is `docker image ls`).

Typical order, smallest to largest (exact numbers vary by version and by whether your Docker shows compressed or uncompressed size):

| Image | Approximate size | Why |
|---|---|---|
| `alpine` | about 8 MB | musl libc + BusyBox. Nothing else. |
| `debian:bookworm-slim` | about 75 MB | Standard Debian base with GNU tools and `apt`, documentation removed |
| `ubuntu:24.04` | about 80 MB | Similar to Debian, a few more defaults |
| `nginx` | about 190 MB | A Debian base **plus** nginx and its modules |

> **[LEARNING]** Question 2 — what do you lose with Alpine?

Alpine is small because it uses **musl** instead of the more common **glibc** C library, and **BusyBox** instead of the full GNU tools. Consequences:

| Advantage of Alpine | Price you may pay |
|---|---|
| Tiny: fast to pull, little disk space | Some pre-compiled programs and libraries (built for glibc) do not work, or must be compiled — sometimes slowly |
| Fewer packages = fewer potential vulnerabilities | Fewer tools for debugging (`bash`, `curl`, `ps` options…) |
| Fast to start new containers | Small behaviour differences (DNS resolution, locales) that can cause hard-to-find bugs |

Neither is "the best". The sensible answer is: **choose the smallest base your application works reliably on, and test it.** "Slim" Debian images are often a safe middle ground.

> **[LEARNING]** Task 3 — same ID or not?

Your `alpine:latest` and `alpine:3.20` have **different IDs**, because `latest` currently points to a newer 3.x release than 3.20. If you had chosen the exact version that `latest` currently points to, the IDs would be **identical**: two labels on the same image. A tag is a **sticky note**; the ID identifies the actual content.

---

## Task 4 — history

> **[GOOD]** Correct command.

> **[SUGGESTION]** Add `--no-trunc` when you need to read the full instructions, and know how to read the output.

```bash
docker image history nginx
docker image history --no-trunc nginx
```

What to notice:

- each line is a **layer** or a **metadata step**, newest at the top;
- lines with a size of `0B` (such as `CMD`, `EXPOSE`, `ENV`, `STOPSIGNAL`) change **metadata**, not files;
- the big lines are the **base distribution** (the very bottom) and the **`RUN` step that installs nginx**.

This is the "transparent sheets" stack from Lesson 01, made visible. Later, when you write Dockerfiles, `history` is how you find out which step made your image fat.

---

## Task 5 — `docker inspect alpine`

> **[SHOULD FIX]** Scrolling through 150 lines of JSON is not a method. Ask for exactly what you need.

`docker inspect` accepts the same `--format` templates you saw with `docker info` in Exercise 01:

```bash
docker image inspect alpine --format '{{.Os}}/{{.Architecture}}'
docker image inspect alpine --format '{{json .Config.Cmd}}'
docker image inspect alpine --format '{{json .Config.Env}}'
```

Typical answers: `linux/amd64` (or `linux/arm64` on an Apple Silicon Mac), `["/bin/sh"]`, and a `PATH=...` variable.

| Approach | Good for |
|---|---|
| `docker inspect <thing>` | When you do not know what you are looking for yet: explore once |
| `docker image inspect <image> --format '...'` | Every time after that: precise, scriptable, readable |
| `docker image inspect` vs `docker inspect` | Prefer the specific one: `docker inspect alpine` could match a **container** called `alpine` as well |

> **[LEARNING]** The architecture is worth a second look: `linux/...`. Even on Windows. Images are built for **Linux** and for a **processor type** — another reminder of Lessons 02 and 05.

---

## Task 6 — digests

> **[GOOD]** `--digests` is the right option.

> **[LEARNING]** Tag versus digest — question 4.

| | Tag (`alpine:3.20`) | Digest (`alpine@sha256:4bcff6...`) |
|---|---|---|
| What it is | A human-friendly **name** | A **fingerprint** of the exact content |
| Can it change? | **Yes** — maintainers can move it to a new image (security patches, for example) | **Never** — different content means a different digest |
| Analogy | "Today's special" on a menu | A specific dish's photo with a serial number |
| Use it when | You want updates within a version line | You need to be **certain** to get exactly the same bytes (reproducible builds, audits) |

Both can be combined, which is the professional form in critical places:

```bash
docker pull alpine:3.20@sha256:<the digest you noted>
```

The tag tells humans what it is; the digest guarantees what it is. If the tag later moves, the digest still wins.

> **[LEARNING]** Question 5 — why pin versions? Because "it worked yesterday" is only useful if today you run **the same thing** as yesterday. Without pinning, a rebuild next month may pull a different base image, and a bug appears with no change in *your* code. This is the "it works on my machine" problem from Lesson 00 — coming back through the back door.

---

## Task 7 — pulling again

> **[GOOD]** And the answer `Image is up to date` is the point.

> **[LEARNING]** Docker compared the **digest** of your local `alpine:latest` with the registry's. Same digest → nothing to download. If the maintainers had moved `latest` in the meantime, Docker would have downloaded only the **layers you do not have yet**. That is where `Already exists` comes from (question 3): a layer with the same digest is already stored, possibly because another image shares it. Layers are shared and downloaded once (Lesson 01, section 3).

---

## Task 8 — removing images

> **[GOOD]** `docker rmi alpine:3.20` — targeted, by name and tag. Exactly right.

> **[BLOCKER]** `docker rmi -f nginx` — forcing past a safety check you did not understand.

Docker refused the first time **for a reason**: the container `keep-me` was created from that image. The error was a guard rail, not an obstacle.

What `-f` actually did here:

- it removed the **tag** `nginx:latest`;
- it could **not** delete the image content, because a container still needs it. The image is still on disk, now nameless (`<none>`);
- `docker container ls -a` now shows `keep-me` with a raw **image ID** instead of `nginx`.

So you freed **no space**, you made the situation **harder to read**, and you trained yourself to override warnings. In a team, that habit eventually deletes something important.

The correct sequence is to remove what depends on the image first:

```bash
docker container rm keep-me
docker image rm nginx
```

| Command | Effect | Use it when |
|---|---|---|
| `docker image rm <name:tag>` | Removes the tag; deletes the content if nothing else uses it | Normal case |
| `docker image rm` refused | Something (a container, another tag) still uses it | Read the error. Remove the dependant first |
| `docker image rm -f` | Removes the tag(s) even if a container uses it; content stays for that container | Almost never as a beginner |

---

## Task 9 — disk usage

> **[SHOULD FIX]** This task was skipped. You jumped straight to deleting.

There is a command to **measure** before you act:

```bash
docker system df
docker system df -v      # detailed, per image / container / volume
```

It shows images, containers, local volumes and build cache, with the size that is **reclaimable**. A good engineer measures first, then decides.

---

## Task 10 — `docker system prune -a --volumes`

> **[BLOCKER]** The most destructive clean-up command available, run without reading what it does.

The task said: *read the documentation and write down exactly what it will delete before running it.* Here is what this command removes:

| Part of the command | What it deletes |
|---|---|
| `docker system prune` | All **stopped containers**, all **unused networks**, all **dangling images**, all unused **build cache** |
| `-a` / `--all` | Additionally, **every image not used by a container** — not just dangling ones. You will re-download everything next time |
| `--volumes` | Additionally, **unused volumes**. In recent Docker versions this means *anonymous* volumes; older versions removed named volumes too. Volumes are where **data** lives (databases!) |

On a learning machine, the cost is a few re-downloads. On a developer's machine with real projects, this is how someone deletes the only copy of their local database, or a stopped container they were going to inspect on Monday. The `[y/N]` prompt protects nobody who answers `y` by reflex.

> **[LEARNING]** Question 6 — dangling versus unused:

| | Dangling image | Unused image |
|---|---|---|
| Definition | An image with **no tag** (`<none>:<none>`), usually an old version replaced by a newer build with the same name | Any image (tagged or not) that **no container** uses |
| Removed by | `docker image prune` | `docker image prune -a` |
| Safe? | Almost always | Only if you accept re-downloading/rebuilding |

The professional approach: **measure, then prune by category, narrowest first.**

```bash
docker system df                    # 1. what takes space?
docker container prune              # 2. stopped containers
docker image prune                  # 3. dangling images only
docker builder prune                # 4. build cache, if it is large
# docker image prune -a             # 5. only if you really want to drop unused images
# volumes: only individually, by name, once you know what is in them
```

---

## The reviewed version

```bash
# 1. Pull with explicit tags
docker pull alpine                      # = alpine:latest, kept deliberately for comparison
docker pull alpine:3.20
docker pull debian:bookworm-slim
docker pull ubuntu:24.04                # or the current LTS

# 2. and 3. Sizes and IDs
docker image ls
docker image ls alpine                  # compare the IMAGE ID of the two alpine tags

# 4. Layers
docker pull nginx
docker image history nginx

# 5. Precise inspection
docker image inspect alpine --format '{{.Os}}/{{.Architecture}}'
docker image inspect alpine --format '{{json .Config.Cmd}}'
docker image inspect alpine --format '{{json .Config.Env}}'

# 6. Digests
docker image ls --digests alpine

# 7. Pull again: nothing to download
docker pull alpine

# 8. Remove targeted; respect the guard rail
docker image rm alpine:3.20
docker run --name keep-me nginx true
docker image rm nginx                   # refused: keep-me uses it -> read why
docker container rm keep-me
docker image rm nginx                   # now it works, cleanly

# 9. Measure
docker system df

# 10. Clean narrowly, after reading the docs
docker container prune
docker image prune
```

---

## Key takeaways

1. **Tags move, digests do not.** Pin versions with tags in everyday work; add digests where you need certainty.
2. **`latest` is a label, not a promise.**
3. **Smaller is not automatically better.** Know what Alpine trades away, then choose.
4. **Errors are guard rails.** `-f` should be the result of understanding the error, never a way to make it go away.
5. **Measure (`docker system df`), then delete narrowly.** Never run a destructive command you have not read the documentation for — especially one with `-a` and `--volumes` in it.

**Previous review:** [Exercise 03](./03-explanations.md) · **Next review:** [Exercise 05 — Seeing Linux at work](./05-explanations.md)
