# Trigger Remote Workflow

Composite GitHub Action that:

1. triggers a workflow in a remote repository,
2. waits for GitHub to register the new run,
3. polls the run status until completion,
4. fails if the remote run does not end with `success` or if it times out.

## How It Works

The action uses the `gh` CLI with these commands:

- `gh workflow run` to trigger the remote workflow;
- `gh run list` to retrieve the latest run ID;
- `gh run view` in a polling loop to read `status` and `conclusion`.

It also supports passing dynamic inputs to the remote workflow through `workflow_inputs` in this format:

```text
key1=value1,key2=value2
```

which is converted into:

```text
--raw-field key1=value1 --raw-field key2=value2
```

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `remote_gh_token` | Yes | - | GitHub token used by the `gh` CLI against the remote repository. |
| `remote_repo` | Yes | - | Remote repository in `owner/repo` format. |
| `remote_workflow` | Yes | - | Workflow file to trigger in the remote repository (for example, `deploy.yml`). |
| `waiting_after_trigger` | No | `10` | Seconds to wait after triggering before reading the run list. |
| `waiting_between_attempts` | No | `20` | Seconds to wait between status checks. |
| `max_attempts` | No | `30` | Maximum polling attempts before timeout. |
| `workflow_inputs` | No | `""` | Comma-separated `key=value` list to pass as workflow inputs. |

## Required Permissions for `remote_gh_token`

The token must be allowed to call GitHub Actions APIs on the **remote repository** (`remote_repo`) in order to:

- trigger a run (`workflow_dispatch`),
- read the run list,
- read run status and conclusion.

Recommended setup:

- **Fine-grained PAT**:
  - repository access: at least the repository configured in `remote_repo`;
  - repository permission `Actions`: **Read and write**.
- **Classic PAT** (if used):
  - scope `repo` (or `public_repo` if the remote repository is public).

> Note: some organizations enforce additional policies that may require extra scopes or approvals.

## Usage Example

```yaml
jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trigger remote workflow
        uses: pagopa/mil-actions/trigger-remote-workflow@main
        with:
          remote_gh_token: ${{ secrets.REMOTE_GH_TOKEN }}
          remote_repo: pagopa/idpay-deploy-aks
          remote_workflow: deploy.yml
          waiting_after_trigger: "15"
          waiting_between_attempts: "20"
          max_attempts: "40"
          workflow_inputs: "environment=uat,version=1.2.3"
```

## Failure Behavior

The action fails if:

- triggering the remote workflow fails;
- `max_attempts` is reached before completion;
- the remote run finishes with `conclusion != success`.
