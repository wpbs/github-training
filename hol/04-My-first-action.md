# 🔨 Hands-on: My first Action

In this hands-on lab you will learn how to create a custom action, pass in parameters and return values to your workflow. Create the action in [your lab repository](../). You pick one (or more) of four flavours and test the action locally with a CI build:

| Flavour | Folder | Reference |
|---|---|---|
| A. [Docker action](04.1-docker-action.md) | `hello-world-docker-action` | [actions/hello-world-docker-action](https://github.com/actions/hello-world-docker-action) (or [actions/container-action](https://github.com/actions/container-action)) |
| B. [Composite action using bash](04.2-composite-bash-action.md) | `hello-world-composite-bash-action` | [composite actions docs](https://docs.github.com/en/actions/tutorials/create-actions/create-a-composite-action) |
| C. [Composite action using PowerShell (pwsh)](04.3-composite-pwsh-action.md) | `hello-world-composite-pwsh-action` | [composite actions docs](https://docs.github.com/en/actions/tutorials/create-actions/create-a-composite-action) |
| D. [Composite action using Python](04.4-composite-python-action.md) | `hello-world-composite-python-action` | [composite actions docs](https://docs.github.com/en/actions/tutorials/create-actions/create-a-composite-action) |

All flavours have the same contract: an input `who-to-greet` (default `World`) and an output `time` with the time of the greeting.

This hands on lab consists of the following steps:
- [🔨 Hands-on: My first Action](#-hands-on-my-first-action)
  - [Creating the action](#creating-the-action)
  - [Testing the action](#testing-the-action)
  - [Optional: Use the action from another workflow](#optional-use-the-action-from-another-workflow)
  - [Summary](#summary)

> **Note**
> Never use the deprecated `::set-output` and `::set-env` workflow commands (or `::save-state`). Write to the files referenced by the `GITHUB_OUTPUT`, `GITHUB_ENV`, `GITHUB_PATH` and `GITHUB_STATE` environment variables instead. See [workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands).

## Creating the action

Pick the flavour you want to try. You can create more than one.

- [A. Docker action](04.1-docker-action.md)
- [B. Composite action using bash](04.2-composite-bash-action.md)
- [C. Composite action using PowerShell (pwsh)](04.3-composite-pwsh-action.md)
- [D. Composite action using Python](04.4-composite-python-action.md)

Come back here when you've committed your action(s) to continue with testing.

> **Note**
> For more extensive custom actions, there are more options than the flavours in this lab:
> - [`actions/github-script`](https://github.com/actions/github-script) runs inline JavaScript with a pre-authenticated Octokit client (`github`) and the workflow `context`, which makes it handy for calling the GitHub API. You can wrap it in a composite action to give the script a name, inputs and outputs, so you can reuse it across repositories.
> - [`actions/typescript-action`](https://github.com/actions/typescript-action) is the official template for a JavaScript/TypeScript action, with tests, linting, bundling and a release workflow. Use it when the logic grows beyond a few steps or needs npm dependencies. It uses the [`@actions/toolkit`](https://github.com/actions/toolkit) packages.
> - [`actions/javascript-action`](https://github.com/actions/javascript-action) is the plain JavaScript variant.

## Testing the action

1. To test the action we create a new workflow file [`.github/workflows/hello-world-ci.yml`](/../../new/main?filename=.github%2Fworkflows%2Fhello-world-ci.yml&workflow_template=blank).

2. The workflow should run on every push if one of the actions has changed. Also add a manual trigger to start the build manually.
   Checkout the repository to reference the action locally and without a git reference. Add one job per flavour you created.

<details>
  <summary>Solution</summary>

```YAML
name: CI Build for custom actions
on:
  push:
    branches: [ main ]
    paths: [ 'hello-world-*-action/**' ]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  docker:
    runs-on: ubuntu-26.04
    steps:
      - uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false
      - id: hello-action
        uses: $/hello-world-docker-action
        with:
          who-to-greet: '@jessehouwing'
      - run: echo "The time was $TIME when the action said hello"
        env:
          TIME: ${{ steps.hello-action.outputs.time }}

  composite-bash:
    runs-on: ubuntu-26.04
    steps:
      - uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false
      - id: hello-action
        uses: $/hello-world-composite-bash-action
        with:
          who-to-greet: '@jessehouwing'
      - run: echo "The time was $TIME when the action said hello"
        env:
          TIME: ${{ steps.hello-action.outputs.time }}

  composite-pwsh:
    strategy:
      matrix:
        os: [ubuntu-26.04, windows-2025]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false
      - id: hello-action
        uses: $/hello-world-composite-pwsh-action
        with:
          who-to-greet: '@jessehouwing'
      - run: echo "The time was $env:TIME when the action said hello"
        shell: pwsh
        env:
          TIME: ${{ steps.hello-action.outputs.time }}

  composite-python:
    strategy:
      matrix:
        os: [ubuntu-26.04, windows-2025]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false
      - id: hello-action
        uses: $/hello-world-composite-python-action
        with:
          who-to-greet: '@jessehouwing'
      - run: echo "The time was $env:TIME when the action said hello"
        shell: pwsh
        env:
          TIME: ${{ steps.hello-action.outputs.time }}
```

</details>

> **Note**
> The `uses` value of a step cannot contain expressions, so a matrix over action folders is not possible. Remove the jobs for flavours you did not create.

3. Run the workflow and see how the parameters are passed into the action and back to the workflow.

<img width="600" alt="image" src="https://user-images.githubusercontent.com/5276337/174239255-262a8014-4b66-40df-aa17-6f043f948342.png">

## Optional: Use the action from another workflow

If time permits, create a workflow [`.github/workflows/use-action.yml`](/../../new/main?filename=.github%2Fworkflows%2Fuse-action.yml&workflow_template=blank) in [your lab repository](../) that uses the action you created. The `$` reference addresses an action in the same repository, regardless of its host or owner.

<details>
  <summary>Solution</summary>

```YAML
name: Test custom action
on: [workflow_dispatch]

permissions:
  contents: read

jobs:
  test-action:
    runs-on: ubuntu-26.04
    steps:
      - name: Say hello
        uses: $/hello-world-composite-bash-action
        with:
          who-to-greet: '@octocat'
```

</details>

## Summary

In this hands-on lab you've learned how to create a Docker or composite action (bash, pwsh or Python), pass in parameters, return values to your workflow using `GITHUB_OUTPUT`, and to test the action locally with a CI build.

You can continue now with [Reusable workflows](05-Reusable-workflows.md).
