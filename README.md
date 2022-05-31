# Reusable Workflows for CI/CD Pipeline
Reusable workflows for CI/CD Pipeline

# Overview
Rather than copying and pasting from one workflow to another, you can make workflows reusable. You and anyone with access to the reusable workflow can then call the reusable workflow from another workflow.  Reusing workflows avoids duplication. This makes workflows easier to maintain and allows you to create new workflows more quickly by building on the work of others, just as you do with actions.
## NexGen Starter Workflow
A workflow that uses another workflow is referred to as a "sarter: or "caller" workflow.  One starter workflow can use multiple called workflows. Each called workflow is referenced in a single line. The result is that the caller workflow file may contain just a few lines of YAML, but may perform a large number of tasks when it's run. When you reuse a workflow, the entire called workflow is used, just as if it was part of the caller workflow.

To pass named inputs and secrets to a called workflow (reusable), the with keyword is used:
```
jobs:
  call-workflow-passing-data:
    uses: octo-org/example-repo/.github/workflows/reusable-workflow.yml@main
    with:
      username: mona
    secrets:
      envPAT: ${{ secrets.envPAT }}
```
Currently, we have 3 caller workflows assigned specifically to feature, develop, and master branches, they are:

+ main_feature.yml : execute jobs against current feature branch
+ main_develop.yml : only execute jobs when a commit is done against the develop branch
+ main_master.yml : only execute jobs when a commit is done against the master branch

It is possible to put these files under a single caller workflow file (main.yml) and using more filtering.

## NexGen Reusable Workflows
For a workflow to be reusable, the values for on must include workflow_call as well if using inputs and secrets.
```
on:
  workflow_call:
    inputs:
      username:
        required: true
        type: string
    secrets:
      envPAT:
        required: true
## Then the reference inputs and secrets are used as:
jobs:
reusable_workflow_job:
  runs-on: ubuntu-latest
  environment: production
  steps:
    - uses: ./.github/workflows/my-action
      with:
        username: ${{ inputs.username }}
        token: ${{ secrets.envPAT }}
```
If you reuse a workflow from a different repository, any actions in the called workflow run as if they were part of the caller workflow.  Currently, NexGen project has identified the following Called Workflows:

+ test.yml
+ sast.yml
+ sca.yml
+ build.yml
+ deploy.yml

### Limitations
+ Reusable workflows can't call other reusable workflows.
+ Reusable workflows stored within a private repository can only be used by workflows within the same repository.
+ Any environment variables set in an env context defined at the workflow level in the caller workflow are not propagated to the called workflow. 
+ The strategy property is not supported in any job that calls a reusable workflow.

## Workflow Triggers
Workflow triggers are events that cause a workflow to run.  These events can be:

+ Events that occur in your workflow's repository using event activity types & filters
+ Events that occur outside of GitHub and trigger a repository_dispatch event on GitHub
+ Scheduled times
+ Manual

For this pipeline, the jobs are controlled by two factors: the branch and keywords in the commit message that is used as a filter to whether or not execute specific jobs.  The keywords controlling the execution of the jobs within a starter workflows are:

+ skip-actions : run no jobs
+ all-jobs : run all jobs
+ sca-only : run SonarQube only
+ sast-only : run SAST scans only
+ test-only : run unit tests
+ build-only : build binary / image
+ deploy : deploy application locally (stand alone java, docker, or kubernetes)
+ merge : raise Pull Request to merge feature to develop branch
+ raise-pr : raise Pull Request to move feature to develop branch  

These filters are set in the commit message such as when pushing new code:
```
name: CI Pipeline - Feature Branch Workflow
on:
  push:
    branches:
      - 'feature-*'
job:
   log-env:
    runs-on: ubuntu-latest
    steps:
      - name: Log ENV Variables
        run: |
          echo "WF_MESSAGE: ${WF_MESSAGE}"
          
  call-sca-workflow:
    if: "contains(github.event.head_commit.message, 'sca-only') || contains(github.event.head_commit.message, 'all-jobs')" 
    needs: log-env
    uses: BLOCK-AID/ci_workflows/.github/workflows/sonar.yml@master
    secrets:
      host: ${{ secrets.SONARQUBE_HOST }}
      login: ${{ secrets.SONARQUBE_TOKEN }}

  call-sast-workflow:
    if: "contains(github.event.head_commit.message, 'sast-only') || contains(github.event.head_commit.message, 'all-jobs')" 
    needs: log-env
    uses: BLOCK-AID/ci_workflows/.github/workflows/trivy.yml@master

  call-test-workflow:
    if: "contains(github.event.head_commit.message, 'test-only') || contains(github.event.head_commit.message, 'all-jobs')" 
    needs: log-env
    uses: BLOCK-AID/ci_workflows/.github/workflows/test.yml@master
```
In this example, only the following jobs will be executed:

+ call-sca-workflow
+ call-sast-workflow

