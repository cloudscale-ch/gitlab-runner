# GitLab, GitLab Runners, Autoscaling, Caching - on cloudscale.ch

💡 A guide and Ansible playbook for running GitLab on cloudscale.ch, together with autoscaled runners and distributed caching.

## ℹ️ Introduction

We are heavy users of GitLab, as are many of our customers. One feature a number of our customers use, is autoscaling GitLab runners. With them, CI capacity is added automatically when needed, and removed when it is not.

This is precisely where a cloud shines: you get the resources when you need them and do not pay for them when you do not. Compute usage at cloudscale.ch is billed to-the-second, and CI usage is highly variable.

This repository explains how to configure GitLab to use autoscaling GitLab runners with cloudscale.ch. Additionally, it includes the use of distributed caching, which is often desirable when using several CI runners.

The playbook contained in this repository will help you set up GitLab, and GitLab runners automatically. You can use this to get started or to take the setup for a test-drive.

### Manual Setup

Documentation on how to configure our autoscale plugin for GitLab, called `fleeting-plugin-cloudscale`, can be found in the plugin repository's README:

https://github.com/cloudscale-ch/fleeting-plugin-cloudscale

## 🚀 Setup With Ansible Playbook

To use the playbook provided in this repository, you need the following:

* a Python virtual environment
* a cloudscale.ch API token (with write permission)

### Preparing Your Virtual Environment

Clone this repository:

```bash
git clone https://github.com/cloudscale-ch/gitlab-runner.git
cd gitlab-runner
```

Create a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

### Preparing Your API Key

To create a new API token:

1. Login to https://control.cloudscale.ch
2. Select the project you want to use
3. Select "API Tokens" in the menu
4. Create an API token with "Write Access"

Before continuing, configure your environment to use the key as follows:

```bash
export CLOUDSCALE_API_TOKEN="..."
```

### Configure Your Instance

Create your own version of the configuration example and adjust it to your
liking:

```bash
cp config-example.yml config.yml
```

At a minimum, you need to set the following:

- `ssh_keys` (add your SSH public key)
- `lets_encrypt_contact` (an e-mail address for Let's Encrypt certificates)

### Create Your Instance

Run the following playbook to create a GitLab VM, install GitLab, and configure a GitLab runner on it, which will be responsible for managing other runners:

```bash
playbooks/gitlab-autoscale.yml
```

The installed GitLab runner does not run any jobs of its own. Instead, it is co-located on the GitLab server where it ensures that queued jobs are processed by autoscaled workers.

Look at the source code to see how everything is put together. The different elements are organized into sections that should be easy to follow.

## 💡 Additional Information

### Test Drive

To run a simple CI job on your new setup, you can login to your GitLab instance, create an empty project, and then commit the following `.gitlab-ci.yml` file:

```yml
image: debian

cache:
  key: global-cache

build:
  script:
    - apt-get update
    - apt-get install -y cowsay
    - >
      test -f cached
      && /usr/games/cowsay "I ran on $CI_RUNNER_DESCRIPTION and found a cache"
      || /usr/games/cowsay "I ran on $CI_RUNNER_DESCRIPTION and found no cache"
    - touch cached
    
  cache:
    paths:
      - cached
```

### Docker Image Cache

GitLab recommends the use of a pull-through cache for the Docker registry. This is useful if you hit rate limits when accessing docker images.

This is not currently covered in this guide, but let us know if you are interested, and we will add additional information.

Here is the related GitLab guide:

https://docs.gitlab.com/runner/configuration/speed_up_job_execution.html#docker-hub-registry-mirror

A good approach would be to put this cache on a VM with a private network, which the runners could access.

You can also use alternative registries, or login to the Docker Hub before using it, which yields a higher rate limit. See the following StackOverflow answer on how to set up GitLab runner to authenticate to a Docker registry, before pulling images:

https://stackoverflow.com/a/65963183
