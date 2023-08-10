# GitHHubActionAction

## Solution 
To prevent storing login credentials we found a solution that consists of two ```.yml``` files. 
1. ```trigger.yml```
2. ```CI.yml```

The first one defines a "dummy_workflow" that triggers the second workflow. 
The second workflow is the actual call of the gitlab-CI that uses the GITLAB_TOKEN.
This workflow will always be executed on the default branch (main branch). 
By doing this we make sure that the ```CI.yml``` that is executed, is always the default branch (should be protected) and can not be changed by someone permission. 
In that way the GITLAB_TOKEN is secure. 

### Protect the TOKEN 
We created an environment with protection rules, called ```protected_branches```.
In our case we made the rule such that only protected branches can use this environment (main).
Within that environment we created a github_secret: the GITLAB_TOKEN. 
It is saved with the name ```ENV_TOKEN```.
This token is only accessible for jobs that are deployed in the ```protected_branches```-environment.

### Trigger Workflows
```trigger.yml``` is just an arbitray github action. The only thing to take care of is to name the workflow the right way. In this case it is named ```Trigger Workflow``` as specified by 
```
name: Trigger Workflow

on:
  push:
...
```
and it is executed each time someone pushes. 

```CI.yml``` contains all the magic.
First define:
```
workflow_run: 
  workflows: [Trigger Workflow]
  types: 
    - completed
...
```
This specifies that the workflow will start once ```Trigger Workflow``` is finished. As alternatives to the flag ```completed``` one can also use ```requested``` or ```in_progress```. See [here](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run) for documentation. 

Next, specify 
```
jobs:
  main_CI_job:
    runs-on: ubuntu-latest
    environment: protected_branches
...
```
the ```environment```-flag specifies the environment to use (obviously). 

The last important part is:
```
- name: Trigger gitlab pipeline
  run: |
    echo ${{ github.event.workflow_run.head_sha }}
    curl -X POST --fail -F token=${{ secrets.ENV_TOKEN }} -F ref=main https://gitlab.icp.uni-stuttgart.de/api/v4/projects/950/trigger/pipeline
...
```
where we echo the **commit hash** of the commit that triggered the whole workflow by using ```github.event.workflow_run.head_sha``` and the token is accessed with ```secrets.ENV_TOKEN```. 
If more information than the commit hash is needed, one can access it buy using the information provided in the response snippet on the right side [here](https://docs.github.com/en/rest/actions/workflow-runs?apiVersion=2022-11-28#list-workflow-runs-for-a-repository).
