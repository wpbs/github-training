# 🔨 Hands-on: Containers in GitHub Actions

In this hands-on lab you will use containers in the workflows in [your lab repository](../). There are two different ways to use a container from a workflow:

- run a single **step** in a container with `uses: docker://<image>`
- run a whole **job** in a container with `jobs.<job_id>.container`

This hands on lab consists of the following steps:
- [Running a step in a container](#running-a-step-in-a-container)
- [Running a whole job in a container](#running-a-whole-job-in-a-container)
- [Caveat: actions expect tools on the image](#caveat-actions-expect-tools-on-the-image)
- [Summary](#summary)

> **Note**
> Containers are only supported on Linux runners (for example `ubuntu-latest`). They are not supported on macOS runners, and on Windows runners only for some scenarios. The runner needs Docker installed, which is the case for GitHub-hosted Ubuntu runners.

## Running a step in a container

With `uses: docker://<image>` a single step runs in a container. The image is pulled (from Docker Hub, or from another registry by using the full URL), the container is started with your workspace mounted, and the step's `with:` values are passed as arguments. Other steps in the job keep running directly on the runner.

1. Create a workflow [`.github/workflows/containers.yml`](/../../new/main?filename=.github%2Fworkflows%2Fcontainers.yml&workflow_template=blank) with a manual trigger and a job `step-container`. Add a step that runs the `alpine:3.22` image through `uses`, and prints the contents of `/etc/os-release` by overriding the entrypoint.

<details>
  <summary>Solution</summary>

```YAML
name: Containers
on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  step-container:
    runs-on: ubuntu-latest
    steps:
      - name: On the runner
        run: cat /etc/os-release

      - name: In an Alpine container
        uses: docker://alpine:3.22
        with:
          entrypoint: /bin/sh
          args: -c "cat /etc/os-release"
```

</details>

2. Commit the file and run the workflow from [Actions](/../../actions). Compare the output of both steps: the first shows Ubuntu, the second shows Alpine.

> **Note**
> - `with.entrypoint` overrides the image's `ENTRYPOINT`; `with.args` replaces its `CMD`.
> - Use a full image reference such as `docker://ghcr.io/<owner>/<image>:<tag>` for other registries. `uses: docker://` only supports public images: they are pulled while the job is initialised, before any step runs, so there is no way to authenticate the pull (an earlier `docker/login-action` step doesn't help). For a private image, log in and call `docker run` from a `run:` step, or use a job `container:`, which supports [`credentials:`](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container#defining-credentials-for-a-container-registry) ([background](https://josh-ops.com/posts/github-actions-docker-actions-private-registry/)).
> - Tags are mutable. For supply chain safety you can pin the image by digest: `docker://alpine@sha256:<digest>`.
> - Environment variables of the step are passed into the container, but the filesystem is not shared with the runner except for the workspace (`GITHUB_WORKSPACE`), `GITHUB_OUTPUT` and the other `GITHUB_*` files.

## Running a whole job in a container

With `container:` on a job, **all** steps of the job run inside the container, including `run:` steps and JavaScript actions. This is useful if you need a specific toolchain or OS userland that isn't on the runner, or if you want the same environment as your local development container.

1. Add a second job `job-container` to the workflow that runs in the `node:22-bookworm-slim` image, and prints the Node.js version and the operating system.

<details>
  <summary>Solution</summary>

```YAML
  job-container:
    runs-on: ubuntu-latest
    container:
      image: node:22-bookworm-slim
      env:
        MY_SETTING: from-the-container-block
    steps:
      - run: |
          node --version
          cat /etc/os-release
          echo "MY_SETTING=$MY_SETTING"
```

</details>

2. Commit the change and run the workflow again. Verify that the job reports Debian with Node.js 22, instead of the Ubuntu runner image.

3. A job can also have **service containers** which run next to the job container on a shared network, for example a database. Add a `redis` service to `job-container`, and check that you can reach it by its service name.

<details>
  <summary>Solution</summary>

```YAML
  job-container:
    runs-on: ubuntu-latest
    container:
      image: node:22-bookworm-slim
    services:
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
    steps:
      - name: Reach the service by name
        run: |
          node -e "const s=require('net').connect(6379,'redis',()=>{s.write('PING\r\n')});s.on('data',d=>{console.log(d.toString());s.end()})"
```

When the job itself runs in a container, services are reachable by their service name (`redis`). When the job runs directly on the runner, services are reachable on `localhost` and you need to map the ports.

</details>

> **Note**
> - Set [`container.credentials`](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container#defining-credentials-for-a-container-registry) (a `username` and `password`, like `docker login`) to pull the image from a private registry, for example `username: ${{ github.actor }}` and `password: ${{ secrets.GITHUB_TOKEN }}` for `ghcr.io`.
> - Use `container.options` for extra `docker create` flags (for example `--cpus 1`). The `--network` and `--entrypoint` options are not supported. Be careful with `--user`: the runner's work directory is owned by the user the runner uses (usually `1001`).
> - Container actions (`uses: docker://` or Docker actions) in a container job run as sibling containers on the same network with the same volume mounts.
> - Container jobs run as `root` by default. Take care when you give an untrusted workflow root in a container that shares the workspace.
> - Defaults for `run:` steps differ per image: `shell: sh` is used when `bash` isn't available. Set `defaults.run.shell` explicitly if you rely on bash features.

## Caveat: actions expect tools on the image

Many actions assume the image has a typical set of tools available. This applies to actions published by GitHub (`actions/*`) as well as to actions on the public marketplace. The GitHub-hosted runner images include them, but a minimal container image like `alpine` or `distroless` usually doesn't. Typical assumptions:

- `git`, for `actions/checkout` to fall back to a clone. Without it checkout uses the REST API to download an archive instead, which results in a workspace without a `.git` folder, and some features (like fetching tags or history) won't work.
- `bash`, because the default shell for `run:` on Linux is `bash` and many composite actions explicitly specify `shell: bash`. Alpine only has `sh` (busybox) by default.
- `jq`, `curl`, `wget`, `tar`, `unzip` and `gzip`, used by setup actions and shell scripts to download, extract and parse JSON.
- `node`, because JavaScript actions run on the runner's Node.js, which is mounted into the container. That only works if the image is compatible with it: Alpine (`musl`) images are a known problem, so install `nodejs` there or choose a `glibc` based image.
- `sudo`, `ca-certificates`, `openssl`, a recent `glibc` and `python3` for many marketplace actions and setup scripts.
- `docker`, if the action itself needs to build or run containers.

1. Add a third job `missing-tools` which runs the `alpine:3.22` image and uses `actions/checkout` followed by a `run` step with `git log -1`, and `shell: bash`. Run the workflow and read the errors.

<details>
  <summary>Solution</summary>

```YAML
  missing-tools:
    runs-on: ubuntu-latest
    container: alpine:3.22
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Needs git and bash
        shell: bash
        run: git log -1
```

The job fails: `bash` isn't available (`OCI runtime exec failed ... bash: not found`), and the checkout falls back to downloading an archive, so there is no git history either.

</details>

2. Fix the job by installing the tools in the container first, before the checkout step, and run the workflow again.

<details>
  <summary>Solution</summary>

```YAML
    steps:
      - name: Install the tools that actions expect
        run: apk add --no-cache bash git jq curl

      - uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Needs git and bash
        shell: bash
        run: git log -1
```

</details>

> **Note**
> Installing tools at the start of every run is slow and fragile. Better options are to:
> - pick a fuller base image (for example the `ubuntu` or `node:<version>-bookworm` images rather than `-slim` or `alpine` variants),
> - build and publish your own base image which includes the tools your workflows need,
> - use the [images GitHub publishes for the runners](https://github.com/actions/runner-images), or [actions/runner](https://github.com/actions/runner/pkgs/container/actions-runner) for self-hosted scenarios.
>
> Before you adopt an image for container jobs, run your normal set of actions on it first.

## Summary

In this hands-on lab you've learned how to run a single step in a container with `uses: docker://`, how to run a whole job (and service containers) with `container:`, and that many actions assume tools like `git`, `bash` and `jq` are available on the image.

You can continue now with [Custom Actions](04-My-first-action.md).
