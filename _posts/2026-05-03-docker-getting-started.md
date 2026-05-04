---
layout: post
title: "Frontend engineer's guide to getting started with Docker"
date: 2026-05-02 12:00:00 -0500
categories: [Docker, Developer Tools]
author: Abhishek Bansal
tags: [docker, backend, productivity]
---
I’ve spent most of my career in the frontend world and mostly in Android, where there is not much of an enviroment to setup. While there are a lot of nuances that are mobile specific but you can mostly just hop on to Android Studio and it takes care of the build and installation/deployments. 
For a long time, Docker was just a buzzword to me. I did have some idea of core concepts like how it is alternative to dedicated VMs and its solves for "works on my machine" issues, but, I never really had any requirement to get my hands dirtly on it.

For a recent project I finally got a chance to do some hands on. I mostly learn by doing, in this post I will layout some basic concepts that I learned and also a cheatsheet that came in handy as I tackeled different issues. Note that this is **not a full Docker guide** but some notes on it from my experience.

# The Mental Model: Containers vs. Virtualization

The biggest hurdle is understanding what a container actually *is*. 

In the old days, we used **Virtual Machines (VMs)**. A VM is a heavy, self-contained box that includes a full Operating System. If you run three VMs, you’re running three entire copies of Linux. It’s slow and resource-heavy.

**Containers** are different. They share the host machine's kernel (the core OS) but stay isolated from each other. Think of it like this: a VM is a standalone building with its own plumbing and power; a Container is an apartment in that building that shares the main infrastructure but has its own front door.

# Docker vs. docker-compose

My project involved dealing with 10 different containers. That is where docker-compose come into picture

* **Docker (The Engine):** This handles individual containers. It’s what you use when you want to run one specific thing, like a one-off database or service.

* **docker-compose (The Orchestrator):** Most modern web apps aren't just one thing. You have a React frontend, a Node backend, and a Postgres database. `docker-compose` allows you to define and run all of them together using a single `.yml` file.


# The Runtime: Do you really need Docker Desktop?

If you're on a Mac, you need a "runtime" to provide a Linux environment for Docker to work. 

**Docker Desktop** is the standard, but it requires a paid license. If your current org does not have paid licence, **Colima** or **Podman** are the alternatives.

I personally settled on **Colima**. It’s lightweight, open-source, and provides the standard `docker.sock`. This means your existing `docker-compose` commands and VS Code extensions just work without extra configuration.

# Getting Started: The First Build

When you join a project using Docker, your first steps usually look like this:

1. **Environment Setup:** Most projects use a template for environment variables.
   ```bash
   cd deployment/docker_compose
   cp env.template .env
   ```

2. **The "Everything" Build:** This compiles your code into images.
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.dev.yml build
   ```

3. **Spinning it up:** Use `-d` to run in the background (detached mode) and `--wait` to ensure services are ready.
   ```bash
   docker-compose up -d --wait
   ```

## Managing Specific Services

You don't always want to rebuild the entire universe. If you’ve only changed the frontend code, you can target just the `web_server`.

**Building a specific service:**
```bash
docker-compose build web_server
```

**Building without cache:**
Sometimes Docker gets "stuck" on an old version of a layer. Using `--no-cache` forces a fresh install of all dependencies.
```bash
docker-compose build --no-cache web_server
```

**Checking Logs:**
Running `docker-compose logs` shows a wall of text from every service. To focus on just your API or Frontend:
```bash
docker-compose logs -f api_server
```
*(The `-f` flag "follows" the logs, meaning it stays open and updates in real-time.)*


# When things break: The Troubleshooting Guide

Docker is great until you hit a wall. Here are the three scenarios I encountered most during my first week.

## 1. The "Exit Code 137" (Out of Memory)
If your build crashes with Code 137, your Docker runtime (Colima/Docker Desktop) has run out of RAM. This actually happened quite frequently for me.

Check your current status:
```bash
colima list
```

Bump the limits and restart:
```bash
colima stop && colima start --memory 12 --cpu 4
```

## 2. The Frozen Disk
If Colima’s virtual disk hits 100%, everything will hang. You need to reset the VM and purge the junk.
```bash
# Force a restart
colima stop -f && colima start

# Reclaim space immediately
docker system prune -a --volumes -f
docker builder prune -af
```

## 3. The "Ghost" Container
Sometimes you change a config, but the container doesn't seem to notice. You can force a hard reset:
```bash
docker-compose up --force-recreate
```

# More Tips for the Daily Workflow

* **Check Status:** Use `docker-compose ps` to see what’s actually running.

* **Monitor Resources:** Use `docker-compose stats` to see which container is eating your CPU.

* **Clean Code:** Ensure `.next`, `node_modules`, and local build artifacts are in your `.dockerignore` file. You don't want to copy gigabytes of local junk into your clean Docker image.

Docker can feel like a lot of overhead at first, but once you have your `colima` settings dialed in and your `docker-compose` commands memorized, it saves you from the "it works on my machine" nightmare forever.