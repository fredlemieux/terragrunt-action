# terragrunt-action

A GitHub Action for installing and running Terragrunt

## Inputs

Supported GitHub action inputs:

| Input Name | Description                                       | Required | Example values |
|:-----------|:--------------------------------------------------|:--------:|:--------------:|
| tf_version | Terraform version to be used in Action execution  |  `true`  |     1.4.6      | 
| tg_version | Terragrunt version to be user in Action execution |  `true`  |     0.50.8     |
| tg_dir     | Directory in which Terragrunt will be invoked     |  `true`  |      work      |
| tg_command | Terragrunt command to execute                     |  `true`  |   plan/apply   |
| tg_comment | Add comment to Pull request with execution output | `false`  |      0/1       |

## Environment Variables

Supported environment variables:

| Input Name            | Description                                                                                                 | 
|:----------------------|:------------------------------------------------------------------------------------------------------------|
| GITHUB_TOKEN          | GitHub token used to add comment to Pull request                                                            |
| TF_LOG                | Log level for Terraform                                                                                     |
| TF_VAR_name           | Define custom variable name as inputs                                                                       |
| INPUT_PRE_EXEC_number | Environment variable is utilized to provide custom commands that will be executed before running Terragrunt |

## Outputs

Outputs of GitHub action:

| Input Name          | Description                     |
|:--------------------|:--------------------------------|
| tg_action_exit_code | Terragrunt exit code            |
| tg_action_output    | Terragrunt output as plain text |

## Usage

Example definition of Github Action workflow:

```yaml
name: 'Terragrunt GitHub Actions'
on:
  - pull_request

env:
  tf_version: '1.4.6'
  tg_version: '0.46.3'
  working_dir: 'project'

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - name: 'Checkout'
        uses: actions/checkout@main

      - name: Check terragrunt HCL
        uses: gruntwork-io/terragrunt-action@v1
        with:
          tf_version: ${{ env.tf_version }}
          tg_version: ${{ env.tg_version }}
          tg_dir: ${{ env.working_dir }}
          tg_command: 'hclfmt --terragrunt-check --terragrunt-diff'

  plan:
    runs-on: ubuntu-latest
    needs: [ checks ]
    steps:
      - name: 'Checkout'
        uses: actions/checkout@main

      - name: Plan
        uses: gruntwork-io/terragrunt-action@v1
        with:
          tf_version: ${{ env.tf_version }}
          tg_version: ${{ env.tg_version }}
          tg_dir: ${{ env.working_dir }}
          tg_command: 'plan'

  deploy:
    runs-on: ubuntu-latest
    needs: [ plan ]
    environment: 'prod'
    if: github.ref == 'refs/heads/main'
    steps:
      - name: 'Checkout'
        uses: actions/checkout@main

      - name: Deploy
        uses: gruntwork-io/terragrunt-action@v1
        with:
          tf_version: ${{ env.tf_version }}
          tg_version: ${{ env.tg_version }}
          tg_dir: ${{ env.working_dir }}
          tg_command: 'apply'
```

Example of passing custom code before running Terragrunt:

```yaml
...
- name: Plan
  uses: gruntwork-io/terragrunt-action@v1
  env:
    # configure git to use custom token to clone repository.
    INPUT_PRE_EXEC_1: |
      git config --global url."https://user:${{secrets.PAT_TOKEN}}@github.com".insteadOf "https://github.com"
    # print git configuration
    INPUT_PRE_EXEC_2: |
      git config --global --list
  with:
    tg_command: 'plan'
...
```

### Using SSH Key for modules in private repositories
This action is executed inside a docker container, as a result setting up the ssh key in the action will not work on 
its own.  To get around this we can pass 

- create SSH key and public key to your GH account or to private repository as deploy key
- save private key as action secret, I usually save it under SSH_PRIVATE_KEY name
- use webfactory/ssh-agent to start SSH agent and use SSH_PRIVATE_KEY
- pass agent socket to Terragrunt action under SSH_AUTH_SOCK env variable

Simplified example:

``` yaml
    - name: create directory to pass ssh-agent socket
      run: "mkdir -p /home/runner/work/_temp/_github_workflow"
      shell: bash

    - uses: webfactory/ssh-agent@v0.7.0
      with:
        ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
        ssh-auth-sock: "/home/runner/work/_temp/_github_workflow/ssh-agent.sock"

    - name: Plan
      uses: gruntwork-io/terragrunt-action@v1
      env:
          # /home/runner/work/_temp/_github_workflow is mapped to /github/workflow during action execution, so we pass different path
          SSH_AUTH_SOCK: "/github/workflow/ssh-agent.sock"
```

Reference:
https://github.com/denis256/tg-gh-action-examples/blob/master/.github/workflows/private-repo.yml#L36
