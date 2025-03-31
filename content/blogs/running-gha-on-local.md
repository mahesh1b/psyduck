---
date: '2025-03-31T16:52:51+05:30'
draft: false
title: 'Running GitHub Actions on Local'
tags: ["git", "cicd"]
cover: 
    image: images/gha-on-local.jpg
---

GitHub Actions has been one of the popular platforms among developers to automate software workflows, enabling them to streamline CI/CD pipelines and automate testing processes. However, testing and iterating on these workflows directly on GitHub can be time-consuming and expensive, especially when dealing with complex actions or debugging errors. That's where the act tool comes in. It offers a seamless solution for running GitHub Actions locally on a local system.

## What is act?

act is a command-line tool designed to run GitHub Actions workflows locally. By simulating the GitHub Actions environment on the local machine, act enables testing, debugging, and iterating on workflows without the need to push changes to GitHub and consume valuable Actions minutes. This significantly accelerates the development process and ensures that workflows are error-free before deploying them.

## How does it work?

When act is run in the GitHub repository, it reads the GitHub Actions from **.github/workflows/** folder and finds out the set of actions that need to be performed. First, it pulls the docker images as mentioned in the workflow files with the help of docker APIs and then runs the workflows on the docker container.

## Key Features

- **Local Workflow Execution**: With act, workflows can be run locally, saving time and GitHub Actions minutes.
- **Environment Simulation**: act simulates the GitHub Actions runner environment, ensuring that workflows run as if they were executed on GitHub.
- **Customization**: act allows for customization by supporting custom Docker images, specifying event types (e.g., push, pull request) for workflow runs.
- **Secrets and Environment Variables**: Secrets and environment variables can be passed effortlessly to workflow with act, perfectly emulating the GitHub environment.

## Prerequisite

As mentioned earlier act uses docker containers in the background to run Github workflows so it is a must that docker is installed on the local system. Here's the documentation to install docker on different operating systems:

- macOS: [https://docs.docker.com/desktop/install/mac-install/](https://docs.docker.com/desktop/install/mac-install/)
- Linux: [https://docs.docker.com/desktop/install/windows-install/](https://docs.docker.com/desktop/install/windows-install/)
- Windows: [https://docs.docker.com/desktop/install/linux-install/](https://docs.docker.com/desktop/install/linux-install/)

## Installation

act is available for macOS, Linux, and Windows. It can be installed with package managers or download the binary directly from the GitHub [releases page](https://github.com/nektos/act/releases/). Here's how to install act on different operating systems:

- macOS: Use Homebrew:

  ```bash
  brew install act
  ```

- Linux: Run the following command:

  ```bash
  curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
  ```

- Windows: Download the binary from the releases page, or use Chocolatey:
  ```console
  choco install act-cli
  ```

## Usage

Using act is simple. Just navigate to the root of the GitHub repository where **.github/workflows** directory resides, and execute the following command:

```bash
act -l
```

This command will scan workflows and present a list of workflows to run. Or specify a workflow by its event name using the **-e** flag:

```bash
act -e push
```

To run a specific job from workflows, use the **-j** flag

```bash
act -j test
```

To pass secrets or environment variables to workflow, utilize the **-s** or **--secret** option:

```bash
act -s MY_SECRET=value
```

## Advanced Usage

**Custom Docker Images**
If workflows require a specific environment, custom docker images can be used with act. Simply create a **.actrc** file in the home directory and specify the Docker image. For example:

```bash
echo "-P ubuntu-latest=nektos/act-environments-ubuntu:18.04" > .actrc
```

**Specifying the GitHub Event Payload**
If the workflow needs to be triggered with specific data then the GitHub event payload can be specified with the **-p** option:

```bash
act push -p event.json
```

## `Act`ion

Let us see the act in action. Here, I have a repository with a simple to-do list react app and GitHub workflow in **.github/workflows/node.js.yml** file. Workflow runs simple echo commands for each of the jobs. This is for understanding purposes, you will have to update those jobs as per your requirements.

```yaml
name: Build Test and Deploy React App

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: code check
        run: echo "Building the code!!!"
  test:
    runs-on: ubuntu-latest
    steps:
      - name: sanity test
        run: echo "Running code test!!!"
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: sanity test
        run: echo "Deploying code!!!"
```

The workflow file consists of three jobs:

1. `build`: Build the code.
2. `test`: Run the test cases.
3. `deploy`: Deploy the code.

I will start by listing all the workflows present in the repository:

```bash
$ github-actions-with-act git:(master) ✗ act -l
WARN  ⚠ You are using Apple M-series chip and you have not specified container architecture, you might encounter issues while running act. If so, try running it with '--container-architecture linux/amd64'. ⚠  
Stage  Job ID        Job name      Workflow name             Workflow file   Events           
0      code-check    code-check    Code Check                code-check.yml  push             
0      sanity-check  sanity-check  Code Check                code-check.yml  push             
0      test          test          Build and Test React App  node.js.yml     push,pull_request
0      deploy        deploy        Build and Test React App  node.js.yml     push,pull_request
0      build         build         Build and Test React App  node.js.yml     push,pull_request
```
So as per the output, we have 5 jobs out of which 2 jobs belong to **Code Check** workflow and the rest to the **Build and Test React App**.
<div style={{ backgroundColor: "#C5DDE3", padding: "1rem" }}>
  **NOTE**: If you too feel annoyed by the warning then use
  **--container-architecture linux/amd64** flag or just add it to the
  **~/.actrc** file. 
  ``` 
  echo '--container-architecture linux/amd64' >> ~/.actrc
  ```
</div>

Lets check all jobs that are triggered for **push** event:

```bash
$ github-actions-with-act git:(master) ✗ act push -l
Stage  Job ID        Job name      Workflow name             Workflow file   Events           
0      code-check    code-check    Code Check                code-check.yml  push             
0      sanity-check  sanity-check  Code Check                code-check.yml  push             
0      deploy        deploy        Build and Test React App  node.js.yml     push,pull_request
0      build         build         Build and Test React App  node.js.yml     push,pull_request
0      test          test          Build and Test React App  node.js.yml     push,pull_request
```

When `act` is used without any flags it will run all workflows that are present in the repository:

```bash
$ github-actions-with-act git:(master) ✗ act        
[Code Check/code-check          ] 🚀  Start image=catthehacker/ubuntu:act-latest
[Build and Test React App/test  ] 🚀  Start image=catthehacker/ubuntu:act-latest
[Code Check/sanity-check        ] 🚀  Start image=catthehacker/ubuntu:act-latest
[Build and Test React App/deploy] 🚀  Start image=catthehacker/ubuntu:act-latest
[Build and Test React App/build ] 🚀  Start image=catthehacker/ubuntu:act-latest
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Code Check/code-check          ]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/build ]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/test  ]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/deploy]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Code Check/sanity-check        ]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
[Build and Test React App/test  ] using DockerAuthConfig authentication for docker pull
[Build and Test React App/build ] using DockerAuthConfig authentication for docker pull
[Code Check/sanity-check        ] using DockerAuthConfig authentication for docker pull
[Code Check/code-check          ] using DockerAuthConfig authentication for docker pull
[Build and Test React App/deploy] using DockerAuthConfig authentication for docker pull
INFO[0004] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/build ]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
INFO[0004] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/test  ]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
INFO[0004] Parallel tasks (0) below minimum, setting to 1 
[Build and Test React App/deploy]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
INFO[0004] Parallel tasks (0) below minimum, setting to 1 
[Code Check/sanity-check        ]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
INFO[0004] Parallel tasks (0) below minimum, setting to 1 
[Code Check/code-check          ]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Build and Test React App/deploy]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Code Check/sanity-check        ]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Build and Test React App/build ]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Build and Test React App/test  ]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Code Check/code-check          ]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Code Check/sanity-check        ] ⭐ Run Main sanity test
[Build and Test React App/deploy] ⭐ Run Main sanity test
[Build and Test React App/test  ] ⭐ Run Main sanity test
[Build and Test React App/build ] ⭐ Run Main code check
[Code Check/code-check          ] ⭐ Run Main code check
[Build and Test React App/test  ]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
[Code Check/sanity-check        ]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
[Build and Test React App/build ]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
[Build and Test React App/deploy]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
[Code Check/code-check          ]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
| Building the code!!!
[Build and Test React App/build ]   ✅  Success - Main code check
| Deploying code!!!
| Running code test!!!
[Build and Test React App/deploy]   ✅  Success - Main sanity test
[Build and Test React App/build ] Cleaning up container for job build
[Build and Test React App/test  ]   ✅  Success - Main sanity test
| Checking the app code
[Build and Test React App/deploy] Cleaning up container for job deploy
| Perfroming the sanity tets!
[Build and Test React App/test  ] Cleaning up container for job test
[Code Check/code-check          ]   ✅  Success - Main code check
[Code Check/sanity-check        ]   ✅  Success - Main sanity test
[Code Check/sanity-check        ] Cleaning up container for job sanity-check
[Code Check/code-check          ] Cleaning up container for job code-check
[Build and Test React App/build ] 🏁  Job succeeded
[Build and Test React App/deploy] 🏁  Job succeeded
[Build and Test React App/test  ] 🏁  Job succeeded
[Code Check/sanity-check        ] 🏁  Job succeeded
[Code Check/code-check          ] 🏁  Job succeeded
```

It is not always required to run all the workflows. If I just want to run the **sanity-check** job, I can use the **-j** flag to specify the job:

```bash
$ github-actions-with-act git:(master) ✗ act -j sanity-check
[Code Check/sanity-check] 🚀  Start image=catthehacker/ubuntu:act-latest
INFO[0000] Parallel tasks (0) below minimum, setting to 1 
[Code Check/sanity-check]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform=linux/amd64 username= forcePull=true
[Code Check/sanity-check] using DockerAuthConfig authentication for docker pull
INFO[0002] Parallel tasks (0) below minimum, setting to 1 
[Code Check/sanity-check]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Code Check/sanity-check]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Code Check/sanity-check] ⭐ Run Main sanity test
[Code Check/sanity-check]   🐳  docker exec cmd=[bash --noprofile --norc -e -o pipefail /var/run/act/workflow/0] user= workdir=
| Perfroming the sanity tets!
[Code Check/sanity-check]   ✅  Success - Main sanity test
[Code Check/sanity-check] Cleaning up container for job sanity-check
[Code Check/sanity-check] 🏁  Job succeeded
```
act only ran the **sanity-check** job which not only saves a lot of time but also helps in debugging issues in the workflow jobs because with act it is possible to run each job independently. 

## Tips for Getting the Most Out of act

- **Debugging**: Enable verbose logging with the **-v** flag to assist with troubleshooting.

- **Workflow Selection**: If there are too many workflows in the repository, use **-W** flag to specify the workflow to run.

- **Local Execution**: In case, it is required to run the workflows directly on the host system then use **-P** flag with one of the following parameters:
  ```bash
  # TO run the workflow on Linux host
  act -P ubuntu-latest=-self-hosted
  # TO run the workflow on Windows host
  act -P windows-latest=-self-hosted
  # TO run the workflow on macOS host
  act -P macos-latest=-self-hosted
  ```

## Conclusion

The act tool is a powerful asset for developers seeking to streamline their GitHub Actions workflow development. By allowing local execution, act not only saves time but Integrating act into the development process can significantly enhance workflow efficiency and reliability. Happy automating!