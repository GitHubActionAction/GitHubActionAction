# GitHHubActionAction

## Solution 
To prevent storing login credentials we found a solution that consists of two ```.yml``` files. 
1. ```trigger.yml```
2. ```CI.yml```

The first one defines a "dummy_workflow" that triggers the second workflow. 
The second workflow is the actual call of the gitlab-CI that uses the GITLAB_TOKEN.
This workflow will always be executed on the default branch (main branch). 
By doing this we make sure that the ```CI.yml``` that is executed, is always the default branch (should be protected) and can not be changed buy someone permission. 
In that way the GITLAB_TOKEN is secure. 

### Protect the TOKEN 
We created an environment with protection rules, called ```protected_branches```.
In our case we made the rule such that only proteced branches can use this environment. (main)
Within that environment we created a github_secret: the GITLAB_TOKEN. 
It is saved with the name ```ENV_TOKEN```.
This token is only accesible for jobs that are deployed in the ```protected_branches``` environment.

### Trigger Workflows
```trigger.yml``` is just an arbitray github action. The only thing to take care of is to name the workflow the right way. In this case it is named ```Trigger Workflow``` as specified by 
```
name: Trigger Workflow

on:
  push:
...
```
and it is executed each time someone pushs. 

```CI.yml``` contains all the magic.
First define:
```
workflow_run: 
  workflows: [Trigger Workflow]
  types: 
    - completed
...
```
This specifies that the workflow will start once ```Trigger Workflow``` is finished. As alternatives to the flag ```completed``` one can also use ```requested``` or ```in_progress```.

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





We have several CI jobs that require an infrastructure that is not easily available in GitHub Actions runners:

    GPU-equipped runners
    AMD-equipped runners (for portability tests)
    Mac Mini runners (for portability tests)
    large persistent storage for artifacts (2 GB per pipeline, retention for 3 months)
    large persistent shared cache (more than 4 GB)

This is currently achieved via webhooks, with a firewall allow list for GitHub IP addresses. However, our GitLab instance relies on custom, fragile PHP code to handle these webhooks. We need to find a more sustainable solution that is actively maintained.
Tasks
Communication between services

Find a suitable third-party app to handle communication between GitHub and GitLab.

Criteria:

    the app must be actively maintained
    must work both on the main branch and on pull requests
        storing login credentials as secret environment variables is not acceptable, since they can be exfiltrated in base64
    should preferably be written in programming languages that ICP people are familiar with
        this way someone can step in when the app fails and submit an actionable bug report upstream

One should look for suitable candidates in the Marketplace (query). Maybe github2lab_action? If it's impossible to establish a communication channel with the GitLab API in a safe way (i.e. without a way for a malicious actor to exfiltrate the credentials), look into webhook-based solutions, such as trigger-gitlab-ci-through-webhooks.
Registering new runners

We have two Mac Mini computers that need to become part of the CI fleet.

Two options are possible:

    register them as self-hosted runners
        pros: one can adapt the existing GitHub Action to use these runners instead of the cloud runner
        cons: can only run on the main branch, because PRs can be opened by anyone (see self-hosted runner security)
    register them as GitLab runners
        pros: can be maintained like any other runner
        cons: one has to find a suitable containerization method to avoid privilege escalation (maybe Cilicon?)
