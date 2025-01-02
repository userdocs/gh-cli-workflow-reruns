# gh cli workflow reruns

Automatically rerun failed github workflows

This is an example repo with a demonstration workflow to show how we can use just [gh cli](https://github.com/cli/cli) to trigger a rerun of failed jobs in a workflow using a drop in solution that only requires one customization.

-   [Actions overview](https://github.com/userdocs/gh-cli-workflow-reruns/actions)
-   [example-workflow.yml](https://github.com/userdocs/gh-cli-workflow-reruns/blob/main/.github/workflows/example-workflow.yml)
-   [ci-auto-rerun-failed-jobs.yml](https://github.com/userdocs/gh-cli-workflow-reruns/blob/main/.github/workflows/ci-auto-rerun-failed-jobs.yml)

### How the examples work

This example repo has a [matrix job](https://github.com/userdocs/gh-cli-workflow-reruns/blob/main/.github/workflows/example-workflow.yml) with two parts.

```yml
strategy:
    fail-fast: false
    matrix:
        name: [workflow-example]
        conclusion: ["job-success", "job-failure"]
```

This matrix fails on the second matrix in the first job, `job-failure`, until it has been rerun twice, based on the `github.run_attempt` context

```yml
if: matrix.conclusion == 'job-failure' && github.run_attempt < 3
run: exit 1
```

Otherwise the `ci-auto-rerun-failed-jobs.yml` would attempt to rerun up to 5 (default) times before it stops. This setting can be configured.

```yml
if: inputs.attempts <= inputs.retries
```

It does this using this an end of workflow job and triggering the workflow file `ci-auto-rerun-failed-jobs.yml` if there were any failures `if:  failure()`

### The fundamental components

There are 3 main components of this to apply. They will be detailed and explained here.

#### Part 1

This goes at the start. It provides the core requirements to use the `ci-auto-rerun-failed-jobs.yml` workflow such as `workflow_dispatch` and `inputs`

```yml
on:
    workflow_dispatch:
        inputs:
            skip_rerun:
                description: "Skip rerun?"
                required: true
                default: false
                type: boolean
            retries:
                description: "Number of rerun retries"
                required: true
                default: "5"
                type: choice
                options: ["1", "2", "3", "4", "5", "6", "7", "8", "9"]
```

> [!NOTE]
> When manually running the job via `workflow_dispatch` you can set two options.

-   `skip_rerun` - a true or false (default false) options to bypass the rerun.
-   `retries` - the number of retry attempts as a list of options, 1 to 9 times.

![](docs/assets/images/workflow_dispatch.png)

#### Part 2

This would go at the end of you workflow, as a separate job, that you want to make sure completes. It will call the `ci-auto-rerun-failed-jobs.yml` and pass some critical inputs.

> [!WARNING]
> The one thing you need to make sure is customized is the `needs: [build, release]` to match the job name it is tracking. No other customizations are required.
> This job will work for `workflow_dispatch` and `schedule` jobs.

```yml
ci-auto-rerun-failed-jobs:
    if: failure() && (github.event.inputs.skip_rerun || 'false') == 'false'
    needs: [build, release]
    concurrency:
        group: ci-auto-rerun-failed-jobs
        cancel-in-progress: true
    permissions:
        actions: write
    runs-on: ubuntu-latest
    env:
        GH_TOKEN: "${{ secrets.AUTO_RERUN || github.token }}"
        github_repo: "" # To use ci-auto-rerun-failed-jobs.yml hosted in a remote repository else default to the current repository. Requires PAT token AUTO_RERUN
    steps:
        - uses: actions/checkout@v4
        - name: ci-auto-rerun-failed-jobs via ${{ env.github_repo || github.repository }}
          run: >
              gh workflow run ci-auto-rerun-failed-jobs.yml
              --repo "${{ env.github_repo || github.repository }}"
              -f github_repo=${{ github.repository }}
              -f run_id=${{ github.run_id }}
              -f attempts=${{ github.run_attempt }}
              -f retries=${{ github.event.inputs.retries || '1' }}
              -f distinct_id=${{ github.event.inputs.distinct_id }}
```

> [!TIP]
> Since we are using [gh cli](https://cli.github.com/manual/index) you can define inputs to be passed you can add them like this

```yml
-f name=value
```

The just add it as an input in the `ci-auto-rerun-failed-jobs.yml` and process it there as `${{ inputs.name }}`

#### Part 3

This unique workflow will then take the defined inputs and rerun the job, watch it to conclusion and provide a small job summary

> [!NOTE]
> It works best locally but is setup to also be triggered from a remote repo
>
> You will need a PAT token configured as `secrets.AUTO_RERUN` with actions/content/workflow perms in the local and remote repo.

```yaml
name: ci auto rerun failed jobs

on:
    workflow_dispatch:
        inputs:
            run_id:
                description: "The run id of the workflow to rerun"
                required: true
            attempts:
                description: "The number of attempts to rerun the workflow"
                required: true
            retries:
                description: "The number of retries to rerun the workflow"
                required: true
            github_repo:
                description: "The repository to rerun the workflow"
                required: false
            distinct_id:
                description: "The distinct id of the workflow to rerun"
                required: false

run-name: ci auto rerun failed jobs - attempt ${{ inputs.attempts }}

jobs:
    gh-cli-rerun:
        name: rerun - attempt ${{ inputs.attempts }}
        permissions:
            actions: write
        runs-on: ubuntu-latest
        env:
            GH_TOKEN: "${{ secrets.AUTO_RERUN || github.token }}"
        steps:
            - name: Host - Checkout action ${{ inputs.distinct_id }}
              uses: actions/checkout@v4

            - name: gh cli rerun and summaries ${{ inputs.distinct_id }}
              if: inputs.attempts <= inputs.retries
              run: |
                  github_repo="${{ inputs.github_repo || github.repository }}"
                  failures="$(gh run view ${{ inputs.run_id }} --log-failed --repo "${github_repo}" | sed "s,\x1B\[[0-9;]*[a-zA-Z],,g")"

                  if [[ -z "${failures}" ]]; then
                      failures="$(gh run view ${{ inputs.run_id }} --repo "${github_repo}" | sed "s,\x1B\[[0-9;]*[a-zA-Z],,g")"
                  fi

                  if [[ "${{ inputs.retries }}" -ge "2" ]]; then
                      gh run rerun "${{ inputs.run_id }}" --failed --debug --repo "${github_repo}"
                  else
                      gh run rerun "${{ inputs.run_id }}" --failed --repo "${github_repo}"
                  fi

                  printf '%b\n' "# gh cli workflow reruns" >> $GITHUB_STEP_SUMMARY
                  printf '\n%b\n' ":octocat: Here is a summary of inputs from the failed workflow" >> $GITHUB_STEP_SUMMARY
                  printf '\n%b\n' "🟥 Failures at:\n\n\`\`\`log\n${failures}\n\`\`\`" >> $GITHUB_STEP_SUMMARY
                  printf '\n%b\n' "🟦 Attempt: ${{ inputs.attempts }} - Rerun failed jobs in ${{ inputs.run_id }} :hammer:" >> $GITHUB_STEP_SUMMARY

                  if gh run watch ${{ inputs.run_id }} --exit-status --repo "${github_repo}"; then
                      printf '\n%b\n' "✅ Attempt: ${{ inputs.attempts }} succeeded 😺" >> $GITHUB_STEP_SUMMARY
                  else
                      printf '\n%b\n' "❌ Attempt: ${{ inputs.attempts }} failed 😾" >> $GITHUB_STEP_SUMMARY
                  fi
```

## Visual representation

Some images visually demonstrating the progress of the example.

#### First failure

![](docs/assets/images/1.png)

#### First rerun attempt

![](docs/assets/images/2.png)

#### Second rerun attempt

![](docs/assets/images/3.png)

#### Jobs successfully completed with no failures

![](docs/assets/images/4.png)

#### A helpful job summary

#### ![](docs/assets/images/5.png)

## Known limitations

There are a few things you need to understand about how this works and what you can and cannot do with it.

### workflow_call

-   `workflow_call` - This is used when calling the workflow from a reusable workflow. The problem with this is that the workflow is run as a child of the parent.

    In order to rerun a workflow using `gh cli` it must have ended or you will get an error. So as this would be a child process it would prevent the parent from ending and prevent us from being able to rerun it. A child process attempting to restart the parent process. It won't work and you will get gh cli errors and a failed rerun.

    ```
    parent-worklow (ID)
    ├─ child-failed-job
    └─ child-rerun-job # a child trying to restart parent ID whilst running will result in an error
    ```

### schedule

-   `schedule` - currently the `schedule` key cannot take inputs. So if your workflow was started via a schedule the`workflow_dispatch` inputs are all ignored and null. There are two ways to handle this.

    We can set defaults like this, where we provide a default values if the inputs are null. These are what we call the schedule default values.

    ```yml
    if: failure() && (github.event.inputs.skip_rerun || 'true') == 'false'
    run: gh workflow run ci-auto-rerun-failed-jobs.yml -f run_id=${{ github.run_id }} -f attempts=${{ github.run_attempt }} -f retries=${{ github.event.inputs.retries || '5' }}
    ```

    Another option is to use a small job to define the default values as outputs that can be used throughout the workflow:

    ```yml
    scheduled_defaults:
    runs-on: ubuntu-latest
    outputs:
    skip_rerun: ${{ github.event.inputs.skip_rerun || 'false' }}
    retries: ${{ github.event.inputs.retries || '5' }}
    steps:
        - name: Setting Outputs from inputs
          run: printf '%b\n\n' "Setting Outputs from Inputs"
    ```

    So the checks would become like this instead

    ```yml
    if: failure() && needs.scheduled_defaults.outputs.skip_rerun == 'false'
    run: gh workflow run ci-auto-rerun-failed-jobs.yml -f run_id=${{ github.run_id }} -f attempts=${{ github.run_attempt }} -f retries=${{ needs.scheduled_defaults.outputs.retries }}
    ```

### use as a composite action

You can use this via a composite action hosted in this repo using this example:

https://github.com/userdocs/gh-cli-workflow-reruns/blob/main/.github/workflows/ci-auto-rerun-failed-jobs-action.yml
