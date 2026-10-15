# 🔨 Hands-on: Supply chain security

In this hands-on lab you will secure the workflows in [your lab repository](../): you will let GitHub code scanning find a script injection vulnerability in a workflow, block merging until it is fixed, let Dependabot keep the actions you use up to date, and pin/lock actions to an immutable commit.

This hands on lab consists of the following steps:
- [🔨 Hands-on: Supply chain security](#-hands-on-supply-chain-security)
  - [Enabling code scanning for GitHub Actions](#enabling-code-scanning-for-github-actions)
  - [Introducing a script injection](#introducing-a-script-injection)
  - [Exploiting the injection](#exploiting-the-injection)
  - [Seeing GitHub detect the problem](#seeing-github-detect-the-problem)
  - [Enforcing code scanning checks](#enforcing-code-scanning-checks)
  - [Enabling Dependabot for GitHub Actions](#enabling-dependabot-for-github-actions)
  - [Automatically updating an outdated action reference](#automatically-updating-an-outdated-action-reference)
  - [Locking and pinning actions to a commit](#locking-and-pinning-actions-to-a-commit)
    - [Today: `gh-pin-actions`](#today-gh-pin-actions)
    - [Lockfiles with `gh-actions-lock`](#lockfiles-with-gh-actions-lock)
  - [Summary](#summary)

> **Note**
> Code scanning is free for public repositories. For private repositories you need [GitHub Code Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security) enabled.

## Enabling code scanning for GitHub Actions

1. Go to [Settings](/../../settings) | [Advanced Security](/../../settings/security_analysis).
2. Enable `GitHub Advanced Security` / `Code Security` if it's not enabled yet (public repositories have it already).
3. Under `Code scanning` select `Set up` | `Default`. In the dialog, click `Edit`, make sure the language `Actions` is selected and set `Query suite` to `Extended`. Click `Enable CodeQL`.

> **Note**
> The `Actions` language makes CodeQL analyze the workflow files in `.github/workflows` and the `action.yml` files in your repository.
>
> The `Extended` query suite is needed for this lab: script injection in a workflow triggered by `pull_request` is reported by the `actions/code-injection/medium` query, which is not in the `Default` suite. The `Default` suite only reports injection in privileged workflows (like `pull_request_target`, `issue_comment` or `workflow_run`), via `actions/code-injection/critical`.

4. Verify that the `CodeQL` workflow ran under [Actions](/../../actions).

> **Note**
> You can also run CodeQL for Actions locally, before you push, with the [CodeQL extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql). This gives you the same findings in your editor without waiting for a workflow run.

## Introducing a script injection

Expressions like `${{ github.event.issue.title }}` are replaced *before* the script runs. If a value is controlled by an attacker (issue titles, pull request titles, branch names, commit messages), they can inject their own commands into your script.

1. Create a new branch `script-injection`.

2. Create the file [`.github/workflows/injection.yml`](/../../new/main?filename=.github%2Fworkflows%2Finjection.yml&workflow_template=blank) on that branch. The workflow runs when a pull request is opened or edited, and echoes the title of the pull request in a `run` step by using the expression directly in the script.

<details>
  <summary>Solution</summary>

```YAML
name: Script injection demo
on:
  pull_request:
    types: [opened, edited]

permissions:
  contents: read

jobs:
  greet:
    runs-on: ubuntu-26.04
    steps:
      - name: Echo the title
        run: |
          echo "Title: ${{ github.event.pull_request.title }}"
```

</details>

3. Commit the file and create a pull request to `main`.

## Exploiting the injection

Show what an attacker can do with this: crash the runner.

1. Edit the title of the pull request to the text below. This closes the `echo` string and injects a new command.

```text
x"; sudo shutdown -h now; echo "
```

2. Editing the title triggers the workflow (`edited`). The step now runs the following script, in which the title is no longer data but part of the code:

```bash
echo "Title: x"; sudo shutdown -h now; echo ""
```

3. Open the run under [Actions](/../../actions). The runner powers off in the middle of the job and the run fails with the message `The runner has received a shutdown signal` or `The hosted runner lost communication with the server`.

> **Note**
> GitHub-hosted Ubuntu runners give the job user passwordless `sudo`, so injected commands run with full control of the VM. A real attacker would rather steal secrets or the `GITHUB_TOKEN` than shut down the machine. Only try this in your own lab repository.

## Seeing GitHub detect the problem

1. Go to the pull request and wait for the `CodeQL` checks to finish.
2. Open the `Code scanning results / CodeQL` check, or go to [Security](/../../security/code-scanning). You should see a `Code injection` alert (`actions/code-injection/medium`, severity `Medium`) pointing to the `run` line.
3. Read the alert details: it explains which untrusted input flows into the script.

## Enforcing code scanning checks

Now make sure a pull request with such a vulnerability can't be merged. Use a ruleset (or a classic branch protection rule).

1. Go to [Settings](/../../settings) | [Rules](/../../settings/rules) | [Rulesets](/../../settings/rules/new?target=branch) and create a new branch ruleset named `Protect main`.
2. Set `Enforcement status` to `Active` and add the target branch `Default`.
3. Enable `Require a pull request before merging`. Leave `Required approvals` at `0`, because you are the only person in the repository.
4. Enable `Require code scanning results`, add the tool `CodeQL` and set `Security alerts` to `Medium or higher` and `Alerts` to `Errors`. (`High or higher` would not block this alert, because its severity is `Medium`.)
5. Save the ruleset.
6. Go back to the pull request. Merging should now be blocked.

> **Note**
> From now on you can't push directly to `main`. Make all following changes on a branch and merge them with a pull request.

<details>
  <summary>Fix the vulnerability</summary>

Pass the untrusted value through an environment variable. The shell then treats it as data instead of as code.

```YAML
      - name: Echo the title
        run: |
          echo "Title: $PR_TITLE"
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
```

Commit the fix to the branch, wait for the checks to finish and see that the alert is closed and the pull request can be merged.

</details>

## Enabling Dependabot for GitHub Actions

Actions are dependencies of your workflows. Dependabot can notify you when a newer version of an action is released.

1. Create the file [`.github/dependabot.yml`](/../../new/main?filename=.github%2Fdependabot.yml) and configure a `github-actions` package ecosystem that checks the repository root (`/`) every week.

<details>
  <summary>Solution</summary>

```YAML
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

</details>

> **Note**
> The `github-actions` ecosystem with directory `/` covers the workflows in `.github/workflows` and an `action.yml` in the root of the repository. It does **not** cover the `action.yml` files of the custom actions you created in [Custom Actions](04-My-first-action.md), because they live in subfolders.

2. Add the folder of each custom action you created (for example `/hello-world-composite-bash-action`) to the configuration. Use the `directories` option (plural) instead of `directory`. It accepts a list and supports wildcards, so you can cover all actions in a folder at once. If you keep all your actions in a specific folder (like `.github/actions/*`), a single wildcard entry is enough.

<details>
  <summary>Solution</summary>

Either list the folders explicitly:

```YAML
version: 2
updates:
  - package-ecosystem: github-actions
    directories:
      - /
      - /hello-world-docker-action
      - /hello-world-composite-bash-action
      - /hello-world-composite-pwsh-action
      - /hello-world-composite-python-action
    schedule:
      interval: weekly
```

Or use wildcards. `/*/` matches every folder directly under the root, and for actions stored in a dedicated folder you can use for example `/.github/actions/*`:

```YAML
version: 2
updates:
  - package-ecosystem: github-actions
    directories:
      - /
      - /*/
    schedule:
      interval: weekly
```

</details>

3. Configure a [cooldown](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference#cooldown-). Attackers often publish a compromised release of an action and rely on automated updates to pick it up straight away. A cooldown makes Dependabot wait a number of days after a new release before it proposes the update, which gives the community and the publisher time to notice and withdraw a malicious or broken release. Set a default cooldown of 3 days.

<details>
  <summary>Solution</summary>

```YAML
version: 2
updates:
  - package-ecosystem: github-actions
    directories:
      - /
      - /*/
    schedule:
      interval: weekly
    cooldown:
      default-days: 3
```

</details>

> **Note**
> `default-days` applies to all version updates. For the `github-actions` ecosystem only `default-days` is supported; the `semver-major-days`, `semver-minor-days` and `semver-patch-days` options are not. You can use `include` and `exclude` to apply the cooldown to specific actions only.
>
> Dependabot applies a default cooldown of 3 days to version updates even when you don't configure `cooldown`, so configuring it explicitly is how you make it visible and change the duration. The default cooldown does not apply to security updates, so a fix for a known vulnerability isn't delayed. See the [`cooldown` option](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-) in the Dependabot options reference.

4. Commit the file on a branch and merge it to `main` with a pull request.

## Automatically updating an outdated action reference

1. Create a branch and in one of your workflows (for example the workflow from [My first workflow](01-My-first-workflow.md), or the [Custom Actions](04-My-first-action.md) CI workflow) replace the version of the checkout action with an older major version.

<details>
  <summary>Solution</summary>

```YAML
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
```

</details>

2. Commit the change and merge it to `main` with a pull request.

3. Go to [Insights](/../../network/updates) | Dependency graph | Dependabot (or Settings | Advanced Security | Dependabot) and click `Check for updates` to run Dependabot immediately.

4. After a minute a pull request `Bump actions/checkout from 6 to 7.0.1` (or the latest version) is opened by `dependabot[bot]`. Review the release notes in the pull request and merge it. Because of the cooldown, Dependabot only proposes releases that are at least 3 days old; if the newest release is younger, the pull request targets the newest release that has passed the cooldown.

## Locking and pinning actions to a commit

> **Note**
> This is the last step on purpose: pinning and lockfiles add overhead every time you add a workflow or change an action reference, because the SHAs or the lockfile must be updated as well. Do this once the rest of your workflows are in place.

A tag like `@v7.0.1` is not immutable *by default*: unless protected, it can be moved by the publisher (or by an attacker who compromises the publisher's account) to point at different, possibly malicious, code without the version number changing. Pinning each `uses:` to the full commit SHA removes that risk, because a SHA always refers to the exact same code.

GitHub's [immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases) partially fix this at the source: once a maintainer publishes a release as immutable, its Git tag is locked to a commit and can no longer be moved or deleted, and the release gets a verifiable attestation. This protects consumers of that action going forward.

So even with immutable releases, when you pin manually you should still check each ref before trusting it: look up the commit the tag/SHA resolves to, confirm it comes from the action's own release/tag history, and verify there's a release attestation if one is expected (`gh release verify <tag> -R <owner>/<repo>`), rather than assuming any version tag is safe to pin to blindly.

> **Warning**
> GitHub resolves a SHA from *any fork* of a repository through the parent's name. `actions/checkout@<sha>` therefore also runs if `<sha>` only exists in someone's fork (an "impostor commit"). Always copy the SHA from the tag or release in the original repository, never from a pull request or a blog post.

```YAML
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

Doing this by hand for every action, and keeping the SHAs current, does not scale. Two tools help:

### Today: `gh-pin-actions`

[`gh-pin-actions`](https://github.com/amenocal/gh-pin-actions) is a `gh` CLI extension that rewrites `uses:` lines to `owner/repo@sha # version` for you.

1. Install the extension:

```bash
gh extension install amenocal/gh-pin-actions
```

2. Pin every action referenced in `.github/workflows` to the SHA of its currently declared version, writing the result to review copies first:

```bash
gh pin-actions workflows
```

3. Review the generated `*-pin.yml` files, then apply them:

```bash
gh pin-actions workflows --overwrite
```

4. Commit the pinned workflow files on a branch and merge them with a pull request.

> **Note**
> Because the `uses:` value is now a SHA instead of a version tag, Dependabot can still open update pull requests for it (it understands the `# v7.0.1` comment) — you don't lose the update notifications from the previous step by pinning.

### Lockfiles with `gh-actions-lock`

GitHub is replacing SHA-pinning-per-line with a repository-wide lockfile, via the (preview) [`gh-actions-lock`](https://github.com/github/gh-actions-lock) extension. Instead of rewriting every `uses:` to a SHA, it records the resolved, verified commit for every workflow dependency in a lockfile (`.github/workflows/actions.lock`), while your workflow files keep their readable version tags.

1. Install the extension:

```bash
gh extension install github/gh-actions-lock
```

2. Onboard and lock the repository's workflows:

```bash
gh actions-lock
```

3. Commit the resulting `.github/workflows/actions.lock` file alongside your workflows.

4. In CI, verify the lockfile is still valid and up to date (for example as a required status check) without writing any changes:

```bash
gh actions-lock --verify
```

> **Note**
> `gh-actions-lock` is a technical preview: the lockfile format and commands may still change.
>
> Local-path actions like `uses: ./hello-world-composite-bash-action` can't be locked. By default `gh actions-lock` rewrites them to the new same-repository syntax `uses: $/hello-world-composite-bash-action`, which always runs the code at the current commit. The [Custom Actions](04-My-first-action.md) lab already uses the `$/` syntax. 

## Summary

In this hands-on lab you've learned how to detect script injection in workflows with code scanning, how to block merges with a ruleset, how to keep your actions up to date with Dependabot, and how to pin or lock actions to an immutable commit.

You can continue with the [README](../README.md).
