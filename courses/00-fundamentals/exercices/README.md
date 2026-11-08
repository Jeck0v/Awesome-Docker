# Fundamentals — Exercises

These exercises are for someone who has **just installed Docker** (Docker Desktop on Windows/macOS, or Docker Engine on Linux) and has read the [theory lessons](../theory/).

**This page does not give you the commands.** That is on purpose. Finding the right command and the right options in the documentation is a core skill, and you will remember things far better if you find them yourself.

---

## How to work

1. **Read the theory first.** Each exercise tells you which lesson it is based on.
2. **Keep a notes file** (for example `my-answers.md`). For every task, write:
   - the command(s) you used;
   - what you saw;
   - your answers to the questions.
3. **Use the hints only when you are stuck.** They point to the theory and to the official documentation, never to the answer itself.
4. **Use the built-in help.** Every Docker command explains itself: add `--help` after it.
5. **Only then** open the matching file in [`solutions/`](./solutions/). The solutions are written as a **code review** and explain *why* one option is better than another. You will get much more out of them if you have your own attempt to compare.

### Ground rules

- Breaking things is allowed. Containers are disposable that is the whole point.
- If something fails, **read the error message**, all of it, before searching anywhere else.
- At the end of each exercise, **clean up** what you created. Leaving things lying around is a habit you do not want.

### Useful starting points in the official documentation

- Docker CLI reference: <https://docs.docker.com/reference/cli/docker/>
- Get started guide: <https://docs.docker.com/get-started/>
- Docker concepts: <https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/>

---

## Overview

| # | Exercise | Based on | Difficulty |
|---|---|---|---|
| 01 | [First contact: check your installation](#exercise-01--first-contact-check-your-installation) | Lessons 00, 03, 04 | Very easy |
| 02 | [Your first web server](#exercise-02--your-first-web-server) | Lessons 01, 03 | Easy |
| 03 | [Life inside a container](#exercise-03--life-inside-a-container) | Lessons 01, 05 | Easy |
| 04 | [Images, tags and spring cleaning](#exercise-04--images-tags-and-spring-cleaning) | Lessons 01, 03, 05 | Easy to medium |
| 05 | [Seeing Linux at work](#exercise-05--seeing-linux-at-work) | Lessons 02, 05 | Medium |

---

## Exercise 01 — First contact: check your installation

**Read first:** [Lesson 03 — Docker architecture](../theory/03-docker-architecture.md) and [Lesson 04 — Docker Engine](../theory/04-docker-engine.md).

**Goal:** prove that Docker works, and *see with your own eyes* the client, the daemon and the engine's components described in the theory.

### Tasks

1. Display the version of Docker. Make sure the output shows **two separate parts**: one for the client and one for the server.
2. Display the detailed, system-wide information about your Docker installation.
3. Run the official test image called `hello-world`.
4. Run the same image a **second time**.
5. List the containers that are **currently running**.
6. List **all** containers, including the ones that have stopped.
7. List the images stored on your computer.
8. Remove the containers created by `hello-world`. Keep the image.

### Questions to answer in your notes

1. In the output of task 1, what are the versions of the **Engine**, **containerd** and **runc**? Which lesson explained what each one does?
2. In the output of task 2, find: the **storage driver**, the **cgroup version**, the **kernel version** and the **operating system**. If you are on Windows or macOS, why is the kernel a **Linux** kernel?
3. Read the message printed by `hello-world` carefully. Match each of its numbered steps with a part of the diagram in Lesson 03.
4. What was different between the first and the second run? Why?
5. Why did task 5 show nothing, whereas task 6 showed your containers? What is their **status**, and what does the number in brackets mean?
6. What happens if you try to run a Docker command while Docker Desktop (the daemon) is **closed**? Try it, and explain the error in "restaurant" terms.

### Checkpoint

- You have seen a **Client** section and a **Server** section.
- You have seen the text "Hello from Docker!" twice.
- At the end, no `hello-world` container remains, but the `hello-world` image is still listed.

### Hints

- `docker version`: <https://docs.docker.com/reference/cli/docker/version/>
- System-wide information: <https://docs.docker.com/reference/cli/docker/system/info/>
- Listing containers — look carefully at the options: <https://docs.docker.com/reference/cli/docker/container/ls/>
- Removing containers: <https://docs.docker.com/reference/cli/docker/container/rm/>
- Listing images: <https://docs.docker.com/reference/cli/docker/image/ls/>
- The `hello-world` image page: <https://hub.docker.com/_/hello-world>

---

## Exercise 02 — Your first web server

**Read first:** [Lesson 01 — Containerisation](../theory/01-containerization.md) and [Lesson 03 — Docker architecture](../theory/03-docker-architecture.md).

**Goal:** run a real, long-running service (the `nginx` web server), reach it from your browser, and control its whole life cycle.

### Tasks

1. Start a container from the official `nginx` image so that:
   - it runs **in the background** (your terminal must be free once the command returns);
   - it has a **name of your choice**, for example `my-web`;
   - the welcome page is reachable in your browser at **<http://localhost:8080>**.
2. Open <http://localhost:8080> in your browser. You should see "Welcome to nginx!".
3. Display the **logs** of the container. Refresh the browser a few times, and display the logs again.
4. Follow the logs **live** (the command keeps running and shows new lines as they arrive). Refresh the browser and watch. Then leave the live view.
5. **Stop** the container. Check the browser again.
6. **Start** the same container again (do not create a new one). Check the browser.
7. Try to start a **second** nginx container that also uses port **8080** on your computer. Read the error carefully.
8. Start that second container successfully on another port of your computer (for example 8081), while the first one keeps running.
9. Clean up: remove **both** containers.

### Questions to answer in your notes

1. Inside the container, on which port does nginx listen? On which port of **your computer** do you reach it? Which part of your command connects the two? (Look again at Lesson 05, section 7.)
2. Why could the two containers both use port 80 **inside**, but not both use port 8080 **outside**?
3. What is the difference between **stopping** and **removing** a container? What survived the stop/start cycle?
4. What would have happened if you had not run the container in the background? How would you have got your terminal back?
5. What happened when you tried to remove a container that was still running? Why do you think Docker behaves like that?
6. **Bonus:** By default, is your web server reachable only from your own computer, or also from other machines on your network? How could you restrict it to your own computer only?

### Checkpoint

- The nginx welcome page appeared at `http://localhost:8080`.
- You saw lines in the logs that correspond to your browser's visits.
- After the stop, the page was no longer reachable; after the start, it came back.
- At the end, no nginx container remains.

### Hints

- Running containers, background mode, names and ports: <https://docs.docker.com/reference/cli/docker/container/run/>
- Publishing ports (concept): <https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/>
- Port publishing and its security implications: <https://docs.docker.com/engine/network/port-publishing/>
- Logs: <https://docs.docker.com/reference/cli/docker/container/logs/>
- Stop / start: <https://docs.docker.com/reference/cli/docker/container/stop/> and <https://docs.docker.com/reference/cli/docker/container/start/>
- The `nginx` image page (which port does it use?): <https://hub.docker.com/_/nginx>

---

## Exercise 03 — Life inside a container

**Read first:** [Lesson 01 — Containerisation](../theory/01-containerization.md) (sections 3 and 4) and [Lesson 05 — Linux kernel primitives](../theory/05-linux-kernel-primitives.md) (sections 3 and 5).

**Goal:** go *inside* a container, change things, and discover for yourself what is kept and what is lost.

### Tasks

1. Start a container from the `alpine` image and get an **interactive shell** inside it. Give the container a name, for example `playground`.
2. Inside the container:
   - display the current user;
   - display the hostname;
   - list the running processes;
   - create a file `/root/hello.txt` containing a sentence of your choice.
3. Leave the shell. Check the **status** of the container.
4. Start the **same** container again, get a shell inside it, and check whether `/root/hello.txt` still exists.
5. Now leave the container **without stopping it**, so that it keeps running in the background.
6. While it is running, open a **second**, separate shell session inside the **same** running container. List the processes again. Compare with task 2.
7. Leave that second shell. Is the container still running?
8. Create a **brand-new** container from the `alpine` image and look for `/root/hello.txt`.
9. Start a throwaway container that **deletes itself automatically** as soon as it exits. Check afterwards that it left nothing behind.
10. Try to open `bash` in an `alpine` container. Then try in an `ubuntu` container.
11. Clean up everything you created.

### Questions to answer in your notes

1. Which user were you inside the container? Which process had the **PID 1**? Which lesson explains why?
2. Why does the container **stop** when you leave the shell in task 3, but **not** when you leave the shell in task 7?
3. Why did the file survive in task 4 but not exist in task 8? Use the words "writable layer" and "image" in your answer.
4. What is the hostname of the container? Where else have you seen that value?
5. Why does `bash` not exist in `alpine`? What do you use instead?
6. Imagine this container was a database. Where should its data be stored, and why?

### Checkpoint

- You managed to have **two shells** in the same container at the same time.
- You observed a file that survives a stop/start but does not exist in a new container.
- At the end, `playground` and any other test containers are gone.

### Hints

- Interactive mode and terminal options: <https://docs.docker.com/reference/cli/docker/container/run/> (look for the options about STDIN and TTY, and for the option that removes the container on exit)
- Starting and attaching to an existing container: <https://docs.docker.com/reference/cli/docker/container/start/>
- Running a command in a running container: <https://docs.docker.com/reference/cli/docker/container/exec/>
- Detaching from a container without stopping it ("detach keys"): <https://docs.docker.com/reference/cli/docker/container/attach/>
- The `alpine` image page: <https://hub.docker.com/_/alpine>
- Basic Linux commands to know who/where you are: `whoami`, `hostname`, `ps` — see <https://man7.org/linux/man-pages/>

---

## Exercise 04 — Images, tags and spring cleaning

**Read first:** [Lesson 01 — Containerisation](../theory/01-containerization.md) (section 3) and [Lesson 03 — Docker architecture](../theory/03-docker-architecture.md) (section 3.4).

**Goal:** understand image names, tags and layers, compare base images, and learn to clean up safely.

### Tasks

1. Download (without running) the following images:
   - `alpine` with **no tag**;
   - `alpine` with a **specific version tag** of your choice (find one on Docker Hub);
   - `debian` with the tag `bookworm-slim`;
   - `ubuntu` with a specific version tag (for example an LTS release).
2. List your images and compare their **sizes**. Sort them in your notes from smallest to largest.
3. Look at the **IMAGE ID** of `alpine:latest` and of the version tag you chose. Are they the same? Why might they be (or not be)?
4. Display the **history** (the layers) of the `nginx` image (pull it if you removed it). How many layers are there? Which lines added the most size?
5. Display the **full low-level details** of one image, and find in them: the processor architecture, the operating system, the default command, and the environment variables.
6. Display the **digest** of one image (a long `sha256:...` value). Find out what makes a digest different from a tag.
7. Pull an image that you already have. What does Docker say?
8. Remove **one** of the images you pulled. Then try to remove an image that is used by an existing container (create a stopped container first if needed). Read the error.
9. Find out how much disk space Docker is using in total (images, containers, volumes, build cache).
10. Clean up unused data — but **before** running the clean-up command, read its documentation and write down exactly what it will delete.

### Questions to answer in your notes

1. What tag does Docker use when you give none? Does that tag guarantee you get the newest version?
2. Why is `alpine` so much smaller than `ubuntu`? What do you **lose** by choosing the smaller one?
3. Why does pulling a second image sometimes say `Already exists` for some layers?
4. What is the difference between a **tag** and a **digest**? Which one would you use if you need to be **certain** you get exactly the same image next year?
5. Why would you want to pin an image to a specific version in a real project?
6. What is the difference between a **dangling** image and an **unused** image? Which clean-up option removes which?

### Checkpoint

- You can explain why `alpine`, `debian:bookworm-slim` and `ubuntu` have such different sizes.
- You found the architecture and default command of an image yourself.
- You know how much disk space Docker uses, and you cleaned up **knowing** what would be deleted.

### Hints

- Pulling images and tags: <https://docs.docker.com/reference/cli/docker/image/pull/>
- What is an image? <https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/>
- Listing images (and showing digests): <https://docs.docker.com/reference/cli/docker/image/ls/>
- Image history: <https://docs.docker.com/reference/cli/docker/image/history/>
- Low-level information: <https://docs.docker.com/reference/cli/docker/inspect/>
- Removing images: <https://docs.docker.com/reference/cli/docker/image/rm/>
- Disk usage: <https://docs.docker.com/reference/cli/docker/system/df/>
- Cleaning up: <https://docs.docker.com/engine/manage-resources/pruning/>
- Image pages on Docker Hub (look at the "Tags" tab): <https://hub.docker.com/_/alpine>, <https://hub.docker.com/_/debian>, <https://hub.docker.com/_/ubuntu>

---

## Exercise 05 — Seeing Linux at work

**Read first:** [Lesson 02 — Containers vs virtual machines](../theory/02-containers-vs-virtual-machines.md) and, above all, [Lesson 05 — Linux kernel primitives](../theory/05-linux-kernel-primitives.md).

**Goal:** observe namespaces and cgroups with your own eyes, and prove that containers share one kernel.

### Part A — One shared kernel

1. Display the **kernel version** from inside an `alpine` container, then from inside an `ubuntu` container, then from inside a `debian` container.
2. Display the **distribution name and version** from inside the same three containers (Linux keeps this information in a text file under `/etc`).
3. Compare the kernel version with the one shown by Docker's system-wide information command (Exercise 01).

### Part B — Namespaces

4. Start two containers in the background from the same image (for example `nginx`), with different names.
5. Display the **hostname** of each one. Display the **IP address** of each one (from the outside, using Docker's low-level information command).
6. List the processes **inside** one of the containers. Then use Docker's command that lists the processes of a container **as seen by the host**. Compare the PIDs.

### Part C — Cgroups

7. Start a container with a **memory limit** of 64 MB, a **CPU limit** of half a processor, and a **maximum of 50 processes**.
8. From **inside** that container, find the three limits in the cgroup files under `/sys/fs/cgroup/`. Write down the exact values and explain each one.
9. Display the **live resource usage** of your running containers. Find the memory limit column. What does it show for a container started **without** a limit?
10. **Bonus:** Run a container with a very small memory limit and a command that keeps consuming memory until it hits the limit. Observe how the container ends, and find its **exit code**. Is it a Docker decision or a kernel decision?
11. Clean up everything.

### Questions to answer in your notes

1. Why is the kernel version identical in all three distributions? What exactly does an "ubuntu" image contain, then?
2. Why does each container have a different hostname and IP address, although they come from the same image? Which namespaces are responsible?
3. Why are the PIDs different inside the container and on the host? Is it the same process or a copy?
4. What happens to a container that reaches its **CPU** limit? And its **memory** limit? Why is the behaviour different?
5. By default, how much memory may a container use? Why can this be dangerous on a server?
6. Using the table in Lesson 05, section 9, pick **two** advanced Docker topics and explain which Linux knowledge you will need for them.

### Checkpoint

- You proved that three different distributions report the **same kernel**.
- You found your **own** limits written in the cgroup files.
- You can explain the difference between being slowed down by the CPU limit and being killed by the memory limit.

### Hints

- The theory, sections 3, 4 and 8: [Lesson 05](../theory/05-linux-kernel-primitives.md)
- Resource constraints: <https://docs.docker.com/engine/containers/resource_constraints/>
- Processes of a container as seen from the host: <https://docs.docker.com/reference/cli/docker/container/top/>
- Live statistics: <https://docs.docker.com/reference/cli/docker/container/stats/>
- Low-level information and its `--format` option: <https://docs.docker.com/reference/cli/docker/inspect/>
- Linux: kernel information — <https://man7.org/linux/man-pages/man1/uname.1.html>
- Linux: distribution information — <https://man7.org/linux/man-pages/man5/os-release.5.html>
- Linux: cgroups v2 interface files — <https://docs.kernel.org/admin-guide/cgroup-v2.html> (look for `memory.max`, `cpu.max` and `pids.max`)

---

## When you have finished

Open the reviews in [`solutions/`](./solutions/):

| Exercise | Review |
|---|---|
| 01 | [01-explanations.md](./solutions/01-explanations.md) |
| 02 | [02-explanations.md](./solutions/02-explanations.md) |
| 03 | [03-explanations.md](./solutions/03-explanations.md) |
| 04 | [04-explanations.md](./solutions/04-explanations.md) |
| 05 | [05-explanations.md](./solutions/05-explanations.md) |

Compare them with **your** notes. Where you did something differently, ask yourself: *"Is my way wrong, or just different? What are the consequences?"* That question is what turns a beginner into a professional.
