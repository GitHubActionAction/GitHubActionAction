# GitHHubActionAction

## Solution 
To prevent storing login credentials we found a solution that consinsts of two ''.yml''' files. 
- '''trigger.yml'''
- '''CI.yml'''
The first one defines a "dummy_workflow" that triggers the second workflow. 
The second workflow is the actual call of the gitlab-CI that uses the GITLAB_TOKEN.
This workflow will always be triggered on the default branch (main branch). 
By protecting the main branch we make sure that the '''CI.yml''' ran, is always the real one and it is not changed buy someone to echo the GITLAB_TOKEN

### Protect the TOKEN 
We created an environment with protection rules. 
Within that environment we created a github_secret: the GITLAB_TOKEN. 
This token is only accesible form the branches on which the environment applies to. 
In our case we made it onlz accesible from the main branch. 

### Trigger Workflows
'''trigger.yml''' is just an arbitray github action. 
'''CI.yml''' contains all the magic.
First define:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  workflow_run: 
    workflows: [Trigger Workflow]
    types: 
      - completed
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
