+++
author = "Codemageddon"
title = "How to Build a Perfect Docker Image for your UV/PDM Project"
description = "This article describes how to build a perfect Docker image for your uv or pdm-based project"
tags = ["docker", "pdm", "uv", "python"]
toc = true
categories = ["devops"]
image = "images/cover.png"
date = "2024-09-29T12:00:00Z"
comments = true
+++

## Introduction

In my [previous post](../poetry-docker/), I described how to build and maintain a Docker image for a Poetry-based project. In this article, I will focus on projects using [`UV`](https://docs.astral.sh/uv/), a Python package manager focused on dependency management for complex systems, and [`PDM`](https://pdm-project.org), a modern Python project management tool that supports [PEP-621](https://peps.python.org/pep-0621/) metadata. I will skip the preparatory steps, reasoning, and comparisons, as they are mostly the same to the Poetry case. If you'd like to review those, you're welcome to read [this post](../poetry-docker/). You can find example projects related to this article here:

- [UV-based](https://github.com/codemageddon/uv-docker-demo)
- [PDM-based](https://github.com/codemageddon/pdm-docker-demo)

## UV

The UV documentation [covers](https://docs.astral.sh/uv/guides/integration/docker/) Docker usage and provides many examples. You can refer to it whenever you need more in-depth guidance. However, it doesn't offer a complete Dockerfile that meets all the [requirements](../poetry-docker/#what-makes-a-docker-image-perfect), so I’ve created one for you. Below is a Dockerfile for Debian-based images:

```Dockerfile
ARG UV_VERSION=0.4.17
ARG PYTHON_VERSION=3.12
ARG WORKDIR=/usr/src/app
ARG BASE_BUILD_DISTRO=bookworm
ARG BASE_RUNTIME_DISTRO=slim-${BASE_BUILD_DISTRO}

FROM ghcr.io/astral-sh/uv:python${PYTHON_VERSION}-${BASE_BUILD_DISTRO} AS build
ARG WORKDIR
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_CACHE_DIR=/tmp/uv-cache \
    UV_PYTHON_DOWNLOADS=never

WORKDIR ${WORKDIR}

RUN --mount=type=cache,target=${UV_CACHE_DIR} \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

FROM python:${PYTHON_VERSION}-${BASE_RUNTIME_DISTRO} AS runtime
ARG WORKDIR
WORKDIR ${WORKDIR}
ENV PATH="${WORKDIR}/.venv/bin:${PATH}"

COPY --from=build ${WORKDIR}/.venv ${WORKDIR}/.venv
RUN useradd -U -M -d /nonexistent app
USER app
COPY hello_world ./hello_world
ENTRYPOINT ["python", "hello_world/main.py"]
```

Here’s an explanation of what this Dockerfile does:

1. **Declare build arguments** , primarily for version pinning and maintainability.
2. **Use a prebuilt image** that already contains UV as a build image. The base image is almost identical to the runtime image (in this case, Python 3.12 with Debian Bookworm).
3. **Set UV environment variables** to enable bytecode compilation, set link mode, cache directory, and prevent UV from downloading Python.
4. **Set the working directory.**
5. **Run the `uv sync` command** to install application dependencies into a virtual environment located at `/usr/src/app/.venv`. It uses these mounts (temporary connections to resources):
    - A cache directory that helps UV avoid re-downloading packages it has already fetched before.
    - `uv.lock` and `pyproject.toml` to provide dependencies data.
   The following arguments are passed:
    - `--frozen` to avoid updating the lock file during installation.
    - `--no-install-project` to skip application installation at this stage.
    - `--no-dev` to avoid installing development dependencies.
6. **Use a regular slim Python image** without UV for the runtime environment.
7. **Set the working directory.**
8. **Extend the `PATH`** so the Python binary and dependencies can be found in the virtual environment.
9. **Copy the virtual environment** from the build image to the runtime image.
10. **Add the `app` user** and set it as the runtime user.
11. **Copy the application source files** to the work directory.
12. **Set the entrypoint** to launch the application.

## PDM

For PDM-based projects, the process is almost the same:

```Dockerfile
ARG PDM_VERSION=2.19.1
ARG PYTHON_VERSION=3.12
ARG WORKDIR=/usr/src/app
ARG BASE_BUILD_DISTRO=bookworm
ARG BASE_RUNTIME_DISTRO=slim-${BASE_BUILD_DISTRO}

FROM python:${PYTHON_VERSION}-${BASE_BUILD_DISTRO} AS build
ARG PDM_VERSION
ARG WORKDIR
RUN pip install pdm==${PDM_VERSION}

ENV PDM_CHECK_UPDATE=false \
    PDM_CACHE_DIR=/tmp/pdm-cache

WORKDIR ${WORKDIR}

RUN --mount=type=cache,target=${PDM_CACHE_DIR} \
    --mount=type=bind,source=pdm.lock,target=pdm.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    pdm sync --no-self --prod

FROM python:${PYTHON_VERSION}-${BASE_RUNTIME_DISTRO} AS runtime
ARG WORKDIR
WORKDIR ${WORKDIR}
ENV PATH="${WORKDIR}/.venv/bin:${PATH}"

COPY --from=build ${WORKDIR}/.venv ${WORKDIR}/.venv
RUN useradd -U -M -d /nonexistent app
USER app
COPY hello_world ./hello_world
ENTRYPOINT ["python", "hello_world/main.py"]
```

The main differences are:

1. **Manual installation of PDM** via pip.
2. **Different environment variables and arguments**, as expected.

## Bonus

As a bonus, I’ve prepared Alpine-based Dockerfiles for both package managers. The only differences from the Debian-based images are:

1. **Same base image** is used for both the build and runtime (Alpine doesn’t have a "large" version).
2. Instead of using `useradd -U -M -d /nonexistent app`, you need to use `adduser -S -D -h /nonexistent app` to add the `app` user.

Alpine-based images are about 100MB lighter than Debian-slim images (around 50-60MB vs. 150-160MB in my case), but they may introduce runtime and build-time issues for complex projects. This is due to Alpine’s minimalistic nature, which might lack certain libraries and tools commonly found in Debian. Therefore, use Alpine images with caution, especially for more complex setups.

### UV (Alpine-based)

```Dockerfile
ARG UV_VERSION=0.4.17
ARG PYTHON_VERSION=3.12
ARG WORKDIR=/usr/src/app
ARG BASE_DISTRO=alpine

FROM ghcr.io/astral-sh/uv:python${PYTHON_VERSION}-${BASE_DISTRO} AS build
ARG WORKDIR
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_CACHE_DIR=/tmp/uv-cache \
    UV_PYTHON_DOWNLOADS=never

WORKDIR ${WORKDIR}

RUN --mount=type=cache,target=${UV_CACHE_DIR} \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

FROM python:${PYTHON_VERSION}-${BASE_DISTRO} AS runtime
ARG WORKDIR
WORKDIR ${WORKDIR}
ENV PATH="${WORKDIR}/.venv/bin:${PATH}"

COPY --from=build ${WORKDIR}/.venv ${WORKDIR}/.venv
RUN adduser -S -D -h /nonexistent app
USER app
COPY hello_world ./hello_world
ENTRYPOINT ["python", "hello_world/main.py"]
```

### PDM (Alpine-based)

```Dockerfile
ARG PDM_VERSION=2.19.1
ARG PYTHON_VERSION=3.12
ARG WORKDIR=/usr/src/app
ARG BASE_DISTRO=alpine

FROM python:${PYTHON_VERSION}-${BASE_DISTRO} AS build
ARG PDM_VERSION
ARG WORKDIR
RUN pip install pdm==${PDM_VERSION}

ENV PDM_CHECK_UPDATE=false \
    PDM_CACHE_DIR=/tmp/pdm-cache

WORKDIR ${WORKDIR}

RUN --mount=type=cache,target=${PDM_CACHE_DIR} \
    --mount=type=bind,source=pdm.lock,target=pdm.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    pdm sync --no-self --prod

FROM python:${PYTHON_VERSION}-${BASE_DISTRO} AS runtime
ARG WORKDIR
WORKDIR ${WORKDIR}
ENV PATH="${WORKDIR}/.venv/bin:${PATH}"

COPY --from=build ${WORKDIR}/.venv ${WORKDIR}/.venv
RUN adduser -S -D -h /nonexistent app
USER app
COPY hello_world ./hello_world
ENTRYPOINT ["python", "hello_world/main.py"]
```

## Conclusion

In conclusion, this guide provides Dockerfile templates for both UV and PDM projects, using either Debian or Alpine base images. Depending on your project’s complexity, you can choose between the larger Debian images or the lightweight Alpine ones. Be mindful of potential build issues with Alpine, and ensure that your dependencies are well-supported. Happy building!