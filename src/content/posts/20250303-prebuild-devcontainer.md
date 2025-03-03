---
title: "Prebuild dev containers with GitHub Actions"
meta_title: "Prebuild dev containers with GitHub Actions"
description: "In this article, we are using the 'devcontainers/ci' action to prebuild dev containers for your project."
date: 2025-03-03T05:00:00Z
image: "/images/posts/prebuild.jpg"
categories: ["github"]
authors: ["Bas Steins"]
tags: ["vscode", "codespaces", "github-actions"]
draft: false
---

Development Containers (or DevContainers) are a great way to ensure that everyone working on a project has the same development environment. This is especially useful when working with a team or when you want to make sure that your project can be built and run on any machine. In this article, we are using the `devcontainers/ci` action to prebuild dev containers for your project.

## What are DevContainers?

DevContainers are a way to define a development environment using a `devcontainer.json` file. This file specifies the base image, the extensions that should be installed, and any other settings that are needed to set up the development environment. When you open a project in Visual Studio Code, it will automatically detect the `devcontainer.json` file and set up the development environment for you.

## Prebuilding DevContainers

A `devcontainer.json` file can get arbitrarily complex. Especially, when you are using a custom `Dockerfile` for your project, build times can get quite long. This is where prebuilding dev containers comes in handy. By prebuilding the dev container, you can reduce the time it takes to set up the development environment for your project.

We are using the `devcontainers/ci` action, so that on each `push` to your repository, the dev container is built and pushed to the GitHub Container Registry. This way, the dev container is ready to be used by anyone who wants to work on your project.

In our example, we use a custom docker file with Python 3.12 installed via `pyenv` (which involved compiling). We want to target the `amd64` and `arm64` architectures, so that the image can run natively on Apple Silicon and Intel/AMD based PCs.

## The workflow yaml file

```yaml
name: Pre-build Devcontainer

on:
    workflow_dispatch:
    push:
        branches:
        - main

jobs:
    build_devcontainer:
        runs-on: ubuntu-24.04
        steps:
        - name: Checkout code
          uses: actions/checkout@v4

        - name: Set up QEMU
          uses: docker/setup-qemu-action@v3

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v3
          with:
            version: v0.16.1
            platforms: linux/amd64,linux/arm64
            install: true
            use: true

        - name: Login to GitHub Container Registry
          uses: docker/login-action@v3
          with:
            registry: ghcr.io
            username: ${{ github.repository_owner }}
            password: ${{ secrets.GITHUB_TOKEN }}

        - name: Pre-build dev container image
          uses: devcontainers/ci@v0.3
          with:
            imageName: ghcr.io/bascodes/prebuild-devcontainer-gha
            # cacheTo: type=inline  # This is the default value.
            runCmd: sleep 1
            push: always
            platform: linux/amd64,linux/arm64
```

This workflow sets up Docker's `buildx`, and `qemu`, which is needed to build for two architectures. 

**Note**: Our simple setup takes about an hour to build on GitHub Actions runners. This is because we're using `qemu` to emulate the `arm64` architecture. If you're rebuilding the image frequently, you might want to use dedicated `arm` and `amd` runners to speed up the build process.

### The `devcontainers/ci` action

The `devcontainers/ci` action is a GitHub Action that builds and pushes a Docker image to the GitHub Container Registry. It is a simple way to prebuild dev containers for your project since it automatically detects the `devcontainer.json` file and builds the image based on the settings in that file.

**Note**: The most recent version of the action automatically sets the `--cache-to type=inline` flag, which caches the image layers on the runner. In more complex setups, you might want to provide a dedicated cache image like this:

```yaml
        - name: Pre-build dev container image
          uses: devcontainers/ci@v0.3
          with:
            imageName: ghcr.io/bascodes/prebuild-devcontainer-gha
            cacheTo: type=registry,ref=ghcr.io/bascodes/prebuild-devcontainer-gha:cache
            runCmd: sleep 1
            push: always
            platform: linux/amd64,linux/arm64
```

## The `devcontainer.json` file

```json
{
    "name": "devcontainers-ci",
    "build": {
        "dockerfile": "./Dockerfile",
        "context": ".",
        "cacheFrom": "type=registry,ref=ghcr.io/bascodes/prebuild-devcontainer-gha:latest"
    },
    "remoteUser": "root",
    "features": {
        "ghcr.io/devcontainers/features/github-cli:1": "latest"
    }
}
```

Here, we specify that the dev container should be build according to a custom `Dockerfile`. We also specify a `cacheTo` argument to the `build` options.

This is the exact cache layer that we build using the GitHub Action Workflow above.

**Note**: When setting up your repository with a `devcontainer.json` file and a Github Actions workflow, please remove the `cacheFrom` argument from the `devcontainer.json` file at first. On the first run, there is no image to cache from. Once your workflow ran once successfully, you can add the `cacheFrom` argument to the `devcontainer.json` file.


## Seeing it in action

Check out the Github repository at [bascodes/prebuild-devcontainer-gha](https://github.com/bascodes/prebuild-devcontainer-gha) to see the workflow in action.

Once, you open it in VSCode, you will be prompted to open the project in a dev container. The dev container will be pulled from the GitHub Container Registry and set up in your local environment.

![reopen in dev container](/images/posts/prebuild/reopen.png)

Once you click that, the dev container will be pulled from the GitHub Container Registry and set up in your local environment.

You can verify that the build process actually used the cache by checking the logs of the GitHub Actions workflow.

![cached layers](/images/posts/prebuild/cached.png)

All the layers should be marked with `CACHED`.


## Conclusion

Prebuilding dev containers is a great way to speed up the setup of your development environment. By using the `devcontainers/ci` action, you can easily build and push the dev container to the GitHub Container Registry. This way, the dev container is ready to be used by anyone who wants to work on your project.
