# 🔨 Hands-on: CI/CD with staged deployments

In this hands-on lab you will create environments and a staged deployment workflow with approvals.

> Note: environments are available in public repositories on all plans. For private repositories you need GitHub Pro, Team or Enterprise, and required reviewers in private repositories need GitHub Enterprise. Use a public [lab repository](../) where your organization allows it; otherwise, check that your plan and organization policy support the features in this lab.

This hands on lab is based on [My first workflow](01-My-first-workflow.md) and adds the following steps:
- [🔨 Hands-on: CI/CD with staged deployments](#-hands-on-cicd-with-staged-deployments)
  - [Creating and protecting environments](#creating-and-protecting-environments)
  - [Adding a input for picking environments to manual workflow trigger](#adding-a-input-for-picking-environments-to-manual-workflow-trigger)
  - [Turning off caching for deployments](#turning-off-caching-for-deployments)
  - [Chaining workflow steps and conditional execution](#chaining-workflow-steps-and-conditional-execution)
  - [Reviewing and approving deployments](#reviewing-and-approving-deployments)
  - [Summary](#summary)

## Creating and protecting environments

1. Go to [Settings](/../../settings) | [Environments](/../../settings/environments) and click [New environment](/../../settings/environments/new)
2. Enter the name `Production` and click `Configure environment`
3. Add yourself as the `Required reviewer` for this environment:

<img width="349" alt="image" src="https://user-images.githubusercontent.com/5276337/174113475-967127de-45a7-4dc9-8477-4de4df62c7e6.png">

4. Only allow the `main` branch to be deployed to this environment:

<img width="350" alt="image" src="https://user-images.githubusercontent.com/5276337/174113782-70a1b18a-0ab9-49fd-a53e-cb2ea78916e1.png">

5. Create two more environments. `Test` and `Load-Test` without any restrictions.

## Adding a input for picking environments to manual workflow trigger

Modify the workflow yml file you created in hands on lab 1 ('My first workflow').
Add an input of the type environment to the `workflow_dispatch` trigger that you created previously.

<details>
  <summary>Solution</summary>

```YAML
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        type: environment
        required: true
```

</details>

## Turning off caching for deployments

Deployment (CD) workflows must not restore or save the dependency cache: a poisoned cache entry could end up in what you ship. Explicitly turn caching off with [`cache-mode`](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#cache-mode).

1. Add `cache-mode: none` at the top level of the workflow file (next to `permissions`) so it applies to every job, including the deployment jobs you add in the next section. A job can still override it with `jobs.<job_id>.cache-mode`.

<details>
  <summary>Solution</summary>

```YAML
permissions:
  contents: read

cache-mode: none
```

</details>

## Chaining workflow steps and conditional execution

1. Now add 3 jobs to the workflow file:
  - Test: runs on `ubuntu-26.04` after `Build`. Only runs when the workflow was triggered manually. Runs on the environment `Test`. The job writes `Testing...` to the workflow log.
  - Load-Test: runs on `ubuntu-26.04` after `Build`. Only runs when the workflow was triggered manually. Runs on the environment `Load-Test`. The job writes `Testing...` to the workflow log and sleeps for 15 seconds.
  - Production: runs on `ubuntu-26.04` after `Test` and `Load-Test`. Deploys to the environment `Production` only if this was selected as the input parameter. The environment has the URL `https://writeabout.net`. To simulate deployment, the job will execute 5 steps. Each step writes `Step x deploying...` to the workflow log and sleeps for 10 seconds.

<details>
  <summary>Solution</summary>

```YAML
  Test:
    runs-on: ubuntu-26.04
    if: github.event_name == 'workflow_dispatch'
    needs: Build
    environment: Test
    steps:
      - run: echo "🧪 Testing..."

  Load-Test:
    runs-on: ubuntu-26.04
    if: github.event_name == 'workflow_dispatch'
    needs: Build
    environment: Load-Test
    steps:
      - run: |
          echo "🧪 Testing..."
          sleep 15

  Production:
    runs-on: ubuntu-26.04
    needs: [Test, Load-Test]
    environment:
      name: Production
      url: https://writeabout.net
    if: inputs.environment == 'Production'
    steps:
      - run: |
          echo "🚀 Step 1 deploying..."
          sleep 10
      - run: |
          echo "🚀 Step 2 deploying..."
          sleep 10
      - run: |
          echo "🚀 Step 3 deploying..."
          sleep 10
      - run: |
          echo "🚀 Step 4 deploying..."
          sleep 10
      - run: |
          echo "🚀 Step 5 deploying..."
          sleep 10
```

</details>

2. Trigger the workflow and select `Production` as the environment:

<img width="212" alt="image" src="https://user-images.githubusercontent.com/5276337/174119722-9b76d479-e355-414b-a534-03d8634536ef.png">

## Reviewing and approving deployments

1. Open the workflow run and start the review.

<img width="600" alt="image" src="https://user-images.githubusercontent.com/5276337/174120029-f395e8ec-5e6e-4350-94c5-130caefaafc2.png">

2. And approve it with a comment:

<img width="350" alt="image" src="https://user-images.githubusercontent.com/5276337/174120086-fed98feb-2d7f-476b-a997-1aa099de7d0e.png">

3. Note the progress bar while deploying...

<img width="200" alt="image" src="https://user-images.githubusercontent.com/5276337/174120314-c900585c-6b94-4fc2-8fe9-92452b0cf187.png">

4. The result looks like this and contains the approval and the URL for the production environment:

<img width="800" alt="image" src="https://user-images.githubusercontent.com/5276337/174120381-cef48594-6663-481a-aadd-1ef0dbd50b0a.png">

## Summary

In this lab you have learned to create and protect environments in GitHub and use them in a workflow, and to turn off cache access for deployments with `cache-mode: none`. You have also learned to conditionally
execute jobs or steps and to chain jobs using the `needs` keyword.

You can continue with [Containers in GitHub Actions](03-Containers.md).
