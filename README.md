# GitLab, GitLab Runners, Autoscaling, Caching - on cloudscale.ch

💡 A guide and Ansible playbook for running GitLab on cloudscale.ch, together with autoscaled runners and distributed caching.

## Introduction

We are heavy users of GitLab, as are many of our customers. One feature a number lot of our customers use is autoscaling GitLab runners. With them, CI capacity is added automatically when needed and removed when it is not.

This is precisely where a cloud is best: you get the resources when you need them and do not pay for them when you do not. Compute usage at cloudscale.ch is billed to-the-second, and CI usage is highly variable.

This repository explains how to configure GitLab to use autoscaling GitLab runners with cloudscale.ch. Additionally, it includes the use of distributed caching, which is often desirable when using several CI runners.

Below, you will find two approaches:

🚀 A playbook contained in this repository will help you set up GitLab, and GitLab runners automatically. You can use this to get started or to take the setup for a test-drive.

📕 Documentation on how to set this up yourself. This approach is useful if you already use GitLab or have a different configuration and deployment method.

Note: Though it is possible to use your own runners for the GitLab SaaS product, the focus here is on managed GitLab.

## 🚀 Setup With Ansible Playbook

To use the playbook provided in this repository, you need the following:

* a Python virtual environment
* a cloudscale.ch API token (with write permission)

### Preparing Your Virtual Environment

Clone this repository:

```bash
git clone https://github.com/cloudscale-ch/gitlab-autoscale
cd gitlab-autoscale
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

As a minimum, you need to set the following:

- `ssh_keys` (add your SSH public key)
- `lets_encrypt_contact` (an e-mail address for Let's Encrypt certificates)

### Create Your Instance

Run the following playbook to create a GitLab VM, install GitLab, and configure a GitLab runner on it, which will be responsible for managing other runners:

```bash
playbooks/gitlab-autoscale.yml
```

The thus installed GitLab runner does not run any jobs of its own. Instead, it is co-located on the GitLab server where it ensures that queued jobs are worked on by autoscaled workers.

Look at the source code to see how everything is put together. The different elements are organized into sections that should be easy to follow.

### Additional Information

To test your new setup, see [Test Drive](#test-drive).

To read about additional considerations, see [Additional Considerations](#additional-considerations).

## 📕 Manual Setup

To get started, install GitLab according to GitLab's official documentation:

https://about.gitlab.com/install/

Next, you need at least one GitLab runner that is always on. This runner will be responsible for launching other runners, as needed. It can be run anywhere and can also run its own jobs.

A good option so that you do not have to use extra resources for this runner, is to simply install it on the same host as your GitLab server. Here, you want a runner that has no other jobs (or its CI jobs will compete with your GitLab server for resources).

Either way, the following applies whether you install this runner on your GitLab server host or in a dedicated VM or container.

### Install GitLab Runner

There are multiple ways to install GitLab runner:

https://docs.gitlab.com/runner/install/

We recommend using a distro-specific package, as it makes updates easier:

https://docs.gitlab.com/runner/install/linux-repository.html

Please skip the "Register a runner" step, as this will be done with a more specific call further down.

### Install Docker Machine

GitLab runner uses Docker Machine to launch VMs and to run the GitLab runner container. Though Docker Machine has been deprecated by Docker Inc., there is currently no better alternative.

GitLab maintains a fork of Docker Machine, which will be be supported until a better solution is provided.

To install, simply run the following commands on your VM:

```bash
sudo curl -L https://gitlab-docker-machine-downloads.s3.amazonaws.com/v0.16.2-gitlab.19/docker-machine-Linux-x86_64 \
-o /usr/local/bin/docker-machine

sudo chmod +x /usr/local/bin/docker-machine
```

Additionally, you will need the cloudscale.ch driver for Docker Machine:

```bash
curl -L https://github.com/cloudscale-ch/docker-machine-driver-cloudscale/releases/download/v1.2.1/docker-machine-driver-cloudscale_1.2.1_linux_amd64.tar.gz \
| tar xz docker-machine-driver-cloudscale

sudo mv docker-machine-driver-cloudscale /usr/local/bin/
```

You can check the following repositories for the latest releases:

- https://gitlab.com/gitlab-org/ci-cd/docker-machine
- https://github.com/cloudscale-ch/docker-machine-driver-cloudscale

### Acquire GitLab Runner Registration Token

To register your runner, you need a registration token. This can be a global token (to create a shared runner) or a project-specific token.

For a shared runner, open the admin dashboard at `/admin`, and select "Runners" from the left (`/admin/runners`). Clicking on "Register an instance runner" allows you to view and copy the registration token.

### Register GitLab Runner

The following command will register the GitLab runner, and configure it to autoscale CI jobs using ephemeral runner instances. Run this on the VM you have installed your runner on:

```bash
# The token that will be used to create new runners
export CLOUDSCALE_API_TOKEN="..."

# The flavor to use for autoscale GitLab runners
export CLOUDSCALE_FLAVOR="flex-8-4"

# Your GitLab instance
export GITLAB_URL="https://..."

# The token to register runners with
export GITLAB_RUNNER_REGISTRATION_TOKEN="..."

sudo gitlab-runner register \
   --non-interactive \
   --url "$GITLAB_URL" \
   --registration-token "$GITLAB_RUNNER_REGISTRATION_TOKEN" \
   --run-untagged \
   --executor "docker+machine" \
   --docker-image ubuntu \
   --description "GitLab Runner Manager" \
   --machine-machine-driver cloudscale \
   --machine-machine-name 'autoscale-%s' \
   --machine-machine-options "cloudscale-token=$CLOUDSCALE_API_TOKEN" \
   --machine-machine-options "cloudscale-flavor=$CLOUDSCALE_FLAVOR"
```

Having done this, open the `/etc/gitlab-runner/config.toml` file, and change the `concurrent = 1` value to a higher value. The number you select is the
maximum number of runners starting in parallel.

After editing the file, run `sudo systemctl restart gitlab-runner`.

## 💡 Additional Information

### Test Drive

To run a simple CI job on your new setup, you can login to your GitLab instance, create an empty project, and then use "Web IDE" to add a new file.

Select `.gitlab-ci.yml`. and pick the `Bash` template. It will not do anything useful, but if you commit it, the job should be run on a VM managed by your new GitLab setup!

### Additional Considerations

#### Docker Image Cache

GitLab recommends the use of a pull-through cache for the Docker registry. This is useful if you hit rate limits when accessing docker images.

This is not currently covered in this guide, but let us know you would like it and we will be happy to add additional information.

Here is the GitLab guide on this subject:

https://docs.gitlab.com/runner/configuration/speed_up_job_execution.html#docker-hub-registry-mirror

A good approach would be to put this cache on a VM with private network, which the runners could access.

You can also use alternative registries, or login to the Docker Hub before using it, which yields a higher rate limit. See the following StackOverflow answer on how to set up GitLab runner to authenticate to a Docker registry, before pulling images:

https://stackoverflow.com/a/65963183
