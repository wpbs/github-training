# 🔨 Hands-on: My first workflow

In this hands-on lab you will create your first GitHub Actions Workflow and learn how you can use Actions to automate tasks in your software development lifecycle. If you like more background information, please refer to the [GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) pages on GitHub Docs. Good luck! 👍

This lab is done in [your lab repository](../). If you haven't created it yet, follow [Setting up your lab repository](00-Setup.md) first.

This hands on lab consists of the following steps:
- [Creating the workflow](#creating-the-workflow)
- [Viewing your workflow results](#viewing-your-workflow-results)

## Creating the workflow

Go to **Actions** | [New Workflow](/../../actions/new) and click on [set up a workflow yourself](/../../new/main?filename=.github%2Fworkflows%2Fmain.yml&workflow_template=blank).

1. Rename the file `main.yml` in the `.github/workflows` directory to `github-actions-demo.yml`.
<img width="400" alt="image" src="https://user-images.githubusercontent.com/5276337/174096754-4e2d7219-9caf-42e8-bfd9-c190762886d3.png">

2. Remove the template content - we want to create the workflow from scratch.
3. Click <kbd>Ctrl</kbd>+<kbd>Space</kbd> and select name as the first element:

<img width="400" alt="image" src="https://user-images.githubusercontent.com/5276337/174097468-8be92e37-7948-4895-b5ed-20a22c5773bc.png">

4. Set the workflow name to `GitHub Actions Demo`:

```YAML
name: GitHub Actions Demo
```

5. Add the triggers to the workflow with the help of <kbd>Ctrl</kbd>+<kbd>Space</kbd> and the documentation. We want the workflow to trigger:
  - on every push to the `main` branch, but not when there are changes to files in the `.github` folder.
  - on every pull request with `main` as the base branch
  - Every sunday at 6:15 UTC
  - Manually

<details>
  <summary>Solution</summary>

```YAML
on:
  push:
    branches: [ main ]
    paths-ignore: [.github/**]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '15 6 * * 0'
  workflow_dispatch:
```

</details>

6. Create a job `Build` that runs on the newest Ubuntu image (`ubuntu-26.04`) on GitHub hosted runners. Check the documentation of the [runner images](https://github.com/actions/runner-images) and [GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) for the available labels (`ubuntu-slim` is a lightweight option for simple jobs that don't need Docker). The job should do the following things:
  - Output the name of the event that triggered the workflow
  - Output the name of the branch that the repository is currently referencing
  - List all files in the repository

<details>
  <summary>Solution</summary>

```YAML
permissions:
  contents: read

jobs:
  Build:
    runs-on: ubuntu-26.04
    steps:
      - run: |
          echo "🎉 The job was triggered by event: $EVENT_NAME"
          echo "🔎 The name of your branch is $REF and your repository is $REPOSITORY."
        env:
          EVENT_NAME: ${{ github.event_name }}
          REF: ${{ github.ref }}
          REPOSITORY: ${{ github.repository }}

      - uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false

      - name: List files in the repository
        run: |
          echo "The repository $REPOSITORY contains the following files:"
          tree
        env:
          REPOSITORY: ${{ github.repository }}
```

> **Note**
> Security best practices used in these labs: grant the `GITHUB_TOKEN` only the permissions it needs (`permissions:`), set `persist-credentials: false` on `actions/checkout`, pass untrusted values (branch names, inputs) to scripts through `env:` instead of inlining expressions in `run:`, and don't use caching in deployment (CD) workflows.

</details>

7. Commit the workflow file - and trigger the workflow manually. It should not run automatically if your path filter works. Go to [Actions](/../../actions),  select [GitHub Actions Demo](/../../actions/workflows/github-actions-demo.yml) and `Run workflow`:

<img width="600" alt="image" src="https://user-images.githubusercontent.com/5276337/174105162-19f33fd1-8533-4860-9279-88fabec84451.png">


## Viewing your workflow results

1. Click on your workflow run:

<img width="600" alt="image" src="https://user-images.githubusercontent.com/5276337/174105747-0e205e0d-37cc-464c-905b-5b29be74fc75.png">

2. Click on the job 'Build':

<img width="400" alt="image" src="https://user-images.githubusercontent.com/5276337/174105990-a1c204c6-fb7d-44a4-9343-6982899edb25.png">

3. Expand `Set up job` and note the log with the line number (line numbers are links). Check the information about the virtual environment and included software:

<img width="600" alt="image" src="https://user-images.githubusercontent.com/5276337/174106759-c2a8f933-74cf-42a4-899b-b29dc67eccd7.png">

4. Expand your jobs and check that the output was correct.

<img width="400" alt="image" src="https://user-images.githubusercontent.com/5276337/174107136-af9187c1-dbee-4109-9ddc-f2abd4830282.png">

5. Verify your other triggers by modifying the [README.md](/README.md) file:
  - Modify and commit: triggers build (`push`)
  - Modify and add `[skip ci]` (not triggering the workflow):
  <img width="350" alt="image" src="https://user-images.githubusercontent.com/5276337/174110845-93d4a38a-9c8a-4336-9b6a-9089ea9a1cfd.png">

  - Modify the [README.md](/README.md) file and create a pull request (trigger: `pull_request`)

## Summary

In this hands-on you've learned to create your first workflow with triggers, jobs, steps, and expressions. Next you will set up [CI/CD with staged deployments](02-Staged-deployments.md).
