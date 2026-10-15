# 🔨 Hands-on: Reusable workflows

In this hands-on lab you will create a [reusable workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows#creating-a-reusable-workflow) and a workflow that consumes it in [your lab repository](../). You will learn to pass in parameters to the reusable workflow and use output parameters in the consuming workflow.

This hands on lab consists of the following steps:
- [Creating a reusable workflow](#creating-a-reusable-workflow)
- [Adding an output parameter](#adding-an-output-parameter)
- [Consuming the reusable workflow](#consuming-the-reusable-workflow)

## Creating a reusable workflow

1. Create the file [`.github/workflows/reusable.yml`](/../../new/main?filename=.github%2Fworkflows%2Freusable.yml&workflow_template=blank).
2. Set the name to `Reusable workflow`.

<details>
  <summary>Solution</summary>

```YAML
name: Reusable workflow
```

</details>

3. Add a `workflow_call` trigger with an [input parameter](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_callinputs) `who-to-greet` of the type `string` that is optional, with the default value `World`.

<details>
  <summary>Solution</summary>

```YAML
on:
  workflow_call:
    inputs:
      who-to-greet:
        description: 'The person to greet'
        type: string
        required: false
        default: World
```

</details>

> **Note**
> A `default` is only used when the input is optional. If you set `required: true`, the caller must always pass a value.

4. Add a job named `reusable-job` that runs on `ubuntu-26.04` that echos "Hello <input parameter>" to the console.

<details>
  <summary>Solution</summary>

```YAML
jobs:
  reusable-job:
    runs-on: ubuntu-26.04
    steps:
      - name: Greet someone
        run: echo "Hello $WHO_TO_GREET"
        env:
          WHO_TO_GREET: ${{ inputs.who-to-greet }}
```

</details>

## Adding an output parameter

1. Add an additional step with the id `time` that writes an output named `time` with the current date and time (use `$(date)` for that) to the `GITHUB_OUTPUT` file. See [workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands#setting-an-output-parameter).

<details>
  <summary>Solution</summary>

```YAML
      - name: Set time
        id: time
        run: echo "time=$(date)" >> "$GITHUB_OUTPUT"
```

</details>

2. Add an output called `current-time` to the `reusable-job`.

<details>
  <summary>Solution</summary>

```YAML
    outputs:
      current-time: ${{ steps.time.outputs.time }}
```

</details>

3. Add an output parameter called `current-time` to `workflow_call` and set it to the output of the job.

<details>
  <summary>Solution</summary>

```YAML
    outputs:
      current-time:
        description: 'The time when greeting.'
        value: ${{ jobs.reusable-job.outputs.current-time }}
```

</details>


<details>
  <summary>Complete Solution</summary>

```YAML
name: Reusable workflow

on:
  workflow_call:
    inputs:
      who-to-greet:
        description: 'The person to greet'
        type: string
        required: false
        default: World
    outputs:
      current-time:
        description: 'The time when greeting.'
        value: ${{ jobs.reusable-job.outputs.current-time }}

permissions:
  contents: read

jobs:
  reusable-job:
    runs-on: ubuntu-26.04
    outputs:
      current-time: ${{ steps.time.outputs.time }}
    steps:
      - name: Greet someone
        run: echo "Hello $WHO_TO_GREET"
        env:
          WHO_TO_GREET: ${{ inputs.who-to-greet }}
      - name: Set time
        id: time
        run: echo "time=$(date)" >> "$GITHUB_OUTPUT"
```

</details>

## Consuming the reusable workflow

1. Create the file [`.github/workflows/reuse.yml`](/../../new/main?filename=.github%2Fworkflows%2Freuse.yml&workflow_template=blank).
2. Set the name to `Reuse other workflow` and add a manual trigger.

<details>
  <summary>Solution</summary>

```YAML
name: Reuse other workflow

on: [workflow_dispatch]

permissions:
  contents: read
```

</details>

> **Note**
> A called workflow can never get more `GITHUB_TOKEN` permissions than the caller grants it.

3. Add a job `call-workflow` that uses the reusable workflow and passes in your user name as an input parameter.

<details>
  <summary>Solution</summary>

```YAML
jobs:
  call-workflow:
    uses: $/.github/workflows/reusable.yml
    with:
      who-to-greet: '@octocat'
```

</details>

4. Add another job `use-output` that writes the output parameter `current-time` to the console. (Hint: use the needs context to access the output)

<details>
  <summary>Solution</summary>

```YAML
  use-output:
    runs-on: ubuntu-26.04
    needs: [call-workflow]
    steps:
      - run: echo "Time was $CURRENT_TIME"
        env:
          CURRENT_TIME: ${{ needs.call-workflow.outputs.current-time }}
```

</details>

5. Run the workflow and observe the output.

## Summary

In this lab you have learned to create a reusable workflow and a workflow that consumes it. You also have learned to pass in parameters to the reusable workflow and to use output parameters in the consuming workflow.

You can continue with the [Supply chain security](06-Supply-chain-security.md) lab.
