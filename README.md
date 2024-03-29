# Template for deploying k3s backed by Flux <!-- omit in toc -->

Highly opinionated template for deploying a single [k3s](https://k3s.io) cluster with [Ansible](https://www.ansible.com) and [Terraform](https://www.terraform.io) backed by [Flux](https://toolkit.fluxcd.io/) and [SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/).

The purpose here is to showcase how you can deploy an entire Kubernetes cluster and show it off to the world using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://toolkit.fluxcd.io/). When completed, your Git repository will be driving the state of your Kubernetes cluster. In addition with the help of the [Ansible](https://github.com/ansible-collections/community.sops), [Terraform](https://github.com/carlpett/terraform-provider-sops) and [Flux](https://toolkit.fluxcd.io/guides/mozilla-sops/) SOPS integrations you'll be able to commit Age encrypted secrets to your public repo.

# MY PROGRESS [TEMPORARY SECTION] <!-- omit in toc -->

## 0.1. Install tools required locally

### Ansible


# TOC <!-- omit in toc -->

- [1. Overview](#1-overview)
- [2. :wave:&nbsp; Introduction](#2-wave-introduction)
- [3. :memo:&nbsp; Prerequisites](#3-memo-prerequisites)
  - [3.1. :computer:&nbsp; Systems](#31-computer-systems)
  - [3.2. :wrench:&nbsp;Tools](#32-wrenchtools)
    - [3.2.1. Required](#321-required)
    - [3.2.2. Optional](#322-optional)
  - [3.3. :warning:&nbsp; pre-commit](#33-warning-pre-commit)
- [4. :open_file_folder:&nbsp; Repository structure](#4-open_file_folder-repository-structure)
- [5. :rocket:&nbsp; Lets go!](#5-rocket-lets-go)
  - [5.1. :closed_lock_with_key:&nbsp; Setting up GnuPG keys](#51-closed_lock_with_key-setting-up-gnupg-keys)
  - [5.2. :cloud:&nbsp; Global Cloudflare API Key](#52-cloud-global-cloudflare-api-key)
  - [5.3. :page_facing_up:&nbsp; Configuration](#53-page_facing_up-configuration)
  - [5.4. :zap:&nbsp; Preparing Ubuntu with Ansible](#54-zap-preparing-ubuntu-with-ansible)
  - [5.5. :sailboat:&nbsp; Installing k3s with Ansible](#55-sailboat-installing-k3s-with-ansible)
  - [5.6. :small_blue_diamond:&nbsp; GitOps with Flux](#56-small_blue_diamond-gitops-with-flux)
  - [5.7. :cloud:&nbsp; Configure Cloudflare DNS with Terraform](#57-cloud-configure-cloudflare-dns-with-terraform)
- [6. :mega:&nbsp; Post installation](#6-mega-post-installation)
  - [6.1. :point_right:&nbsp; Troubleshooting](#61-point_right-troubleshooting)
  - [6.2. :robot:&nbsp; Integrations](#62-robot-integrations)
- [7. :grey_question:&nbsp; What's next](#7-grey_question-whats-next)
- [8. :handshake:&nbsp; Thanks](#8-handshake-thanks)
- [9. :arrows_counterclockwise: Keeping this repo upto date with the template repo](#9-arrows_counterclockwise-keeping-this-repo-upto-date-with-the-template-repo)

# 1. Overview

- [Introduction](https://github.com/k8s-at-home/template-cluster-k3s#wave-introduction)
- [Prerequisites](https://github.com/k8s-at-home/template-cluster-k3s#memo-prerequisites)
- [Repository structure](https://github.com/k8s-at-home/template-cluster-k3s#open_file_folder-repository-structure)
- [Lets go!](https://github.com/k8s-at-home/template-cluster-k3s#rocket-lets-go)
- [Post installation](https://github.com/k8s-at-home/template-cluster-k3s#mega-post-installation)
- [Thanks](https://github.com/k8s-at-home/template-cluster-k3s#handshake-thanks)

# 2. :wave:&nbsp; Introduction

The following components will be installed in your [k3s](https://k3s.io/) cluster by default. They are only included to get a minimum viable cluster up and running. You are free to add / remove components to your liking but anything outside the scope of the below components are not supported by this template.

Feel free to read up on any of these technologies before you get started to be more familiar with them.

- [cert-manager](https://cert-manager.io/) - SSL certificates - with Cloudflare DNS challenge
- [calico](https://www.tigera.io/project-calico/) - CNI (container network interface)
- [flux](https://toolkit.fluxcd.io/) - GitOps tool for deploying manifests from the `cluster` directory
- [hajimari](https://github.com/toboshii/hajimari) - start page with ingress discovery
- [kube-vip](https://kube-vip.io/) - layer 2 load balancer for the Kubernetes control plane
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) - default storage class provided by k3s
- [metallb](https://metallb.universe.tf/) - bare metal load balancer
- [reloader](https://github.com/stakater/Reloader) - restart pods when Kubernetes `configmap` or `secret` changes
- [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) - upgrade k3s
- [traefik](https://traefik.io) - ingress controller

For provisioning the following tools will be used:

- [Ubuntu](https://ubuntu.com/download/server) - this is a pretty universal operating system that supports running all kinds of home related workloads in Kubernetes
- [Ansible](https://www.ansible.com) - this will be used to provision the Ubuntu operating system to be ready for Kubernetes and also to install k3s
- [Terraform](https://www.terraform.io) - in order to help with the DNS settings this will be used to provision an already existing Cloudflare domain and DNS settings

# 3. :memo:&nbsp; Prerequisites

## 3.1. :computer:&nbsp; Systems

- One or more nodes with a fresh install of [Ubuntu Server 20.04](https://ubuntu.com/download/server). These nodes can be bare metal or VMs.
- A [Cloudflare](https://www.cloudflare.com/) account with a domain, this will be managed by Terraform.
- Some experience in debugging problems and a positive attitude ;)

## 3.2. :wrench:&nbsp;Tools

:round_pushpin: You should install the below CLI tools on your workstation. Make sure you pull in the latest versions.

### 3.2.1. Required

| Tool                                               | Purpose                                                                                                                                 |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| [ansible](https://www.ansible.com)                 | Preparing Ubuntu for Kubernetes and installing k3s                                                                                      |
| [direnv](https://github.com/direnv/direnv)         | Exports env vars based on present working directory                                                                                     |
| [flux](https://toolkit.fluxcd.io/)                 | Operator that manages your k8s cluster based on your Git repository                                                                     |
| [age](https://github.com/FiloSottile/age)          | A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability. |
| [go-task](https://github.com/go-task/task)         | A task runner / simpler Make alternative written in Go                                                                                  |
| [ipcalc](http://jodies.de/ipcalc)                  | Used to verify settings in the configure script                                                                                         |
| [jq](https://stedolan.github.io/jq/)               | Used to verify settings in the configure script                                                                                         |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Allows you to run commands against Kubernetes clusters                                                                                  |
| [sops](https://github.com/mozilla/sops)            | Encrypts k8s secrets with Age                                                                                                           |
| [terraform](https://www.terraform.io)              | Prepare a Cloudflare domain to be used with the cluster                                                                                 |

### 3.2.2. Optional

| Tool                                                   | Purpose                                                  |
|--------------------------------------------------------|----------------------------------------------------------|
| [helm](https://helm.sh/)                               | Manage Kubernetes applications                           |
| [kustomize](https://kustomize.io/)                     | Template-free way to customize application configuration |
| [pre-commit](https://github.com/pre-commit/pre-commit) | Runs checks pre `git commit`                             |
| [gitleaks](https://github.com/zricethezav/gitleaks)    | Scan git repos (or files) for secrets                    |
| [prettier](https://github.com/prettier/prettier)       | Prettier is an opinionated code formatter.               |

## 3.3. :warning:&nbsp; pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) and [gitleaks](https://github.com/zricethezav/gitleaks) will check to make sure you are not by accident committing your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
task pre-commit:init
```
**Remember to run this on each new clone of the repository for it to have effect.**

Commands are of interest, for learning purposes:

This command makes it so pre-commit runs on `git commit`, and also installs environments per the config file.
```
pre-commit install --install-hooks
```
This command checks for new versions of hooks, though it will occasionally make mistakes, so verify its results.
```
pre-commit autoupdate
```

# 4. :open_file_folder:&nbsp; Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

- **base** directory is the entrypoint to Flux
- **crds** directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists
- **core** directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by Flux
- **apps** directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, Flux will prune resources here if they are not tracked by Git anymore

```
cluster
├── apps
│   ├── default
│   ├── networking
│   └── system-upgrade
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
│   └── system-upgrade
└── crds
    └── cert-manager
```

# 5. :rocket:&nbsp; Lets go!

Very first step will be to create a new repository by clicking the **Use this template** button on this page.

Clone the repo to you local workstation and `cd` into it.

:round_pushpin: **All of the below commands** are run on your **local** workstation, **not** on any of your cluster nodes.

### :closed_lock_with_key:&nbsp; Setting up Age

:round_pushpin: Here we will create a Age Private and Public key. Using SOPS with Age allows us to encrypt and decrypt secrets.

1. Create a Age Private / Public Key

```sh
age-keygen -o age.agekey
```

2. Set up the directory for the Age key and move the Age file to it

```sh
mkdir -p ~/.config/sops/age
mv age.agekey ~/.config/sops/age/keys.txt
```

3. Export the `SOPS_AGE_KEY_FILE` variable in your `bashrc`, `zshrc` or `config.fish` and source it, e.g.

```sh
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
source ~/.bashrc
```

4. Fill out the Age public key in the `.config.env` under `BOOTSTRAP_AGE_PUBLIC_KEY`, **note** the public key should start with `age`...

## 5.2. :cloud:&nbsp; Global Cloudflare API Key

In order to use Terraform and `cert-manager` with the Cloudflare DNS challenge you will need to create a API key.

1. Head over to Cloudflare and create a API key by going [here](https://dash.cloudflare.com/profile/api-tokens).

2. Under the `API Keys` section, create a global API Key.

3. Use the API Key in the configuration section below.

## 5.3. :page_facing_up:&nbsp; Configuration

:round_pushpin: The `.config.env` file contains necessary configuration that is needed by Ansible, Terraform and Flux.

1. Copy the `.config.sample.env` to `.config.env` and start filling out all the environment variables. **All are required** and read the comments they will explain further what is required.

2. Once that is done, verify the configuration is correct by running `./configure.sh --verify`

3. If you do not encounter any errors run `./configure.sh` to start having the script wire up the templated files and place them where they need to be.

## 5.4. :zap:&nbsp; Preparing Ubuntu with Ansible

:round_pushpin: Here we will be running a Ansible Playbook to prepare Ubuntu for running a Kubernetes cluster.

:round_pushpin: Nodes are not security hardened by default, you can do this with [dev-sec/ansible-collection-hardening](https://github.com/dev-sec/ansible-collection-hardening) or something similar.

1. Ensure you are able to SSH into you nodes from your workstation with using your private ssh key. This is how Ansible is able to connect to your remote nodes.

2. Install the deps by running `task ansible:deps`

3. Verify Ansible can view your config by running `task ansible:list`

4. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

5. Finally, run the Ubuntu Prepare playbook by running `task ansible:playbook:ubuntu-prepare`

6. If everything goes as planned you should see Ansible running the Ubuntu Prepare Playbook against your nodes.

## 5.5. :sailboat:&nbsp; Installing k3s with Ansible

:round_pushpin: Here we will be running a Ansible Playbook to install [k3s](https://k3s.io/) with [this](https://galaxy.ansible.com/xanmanning/k3s) wonderful k3s Ansible galaxy role. After completion, Ansible will drop a `kubeconfig` in `./provision/kubeconfig` for use with interacting with your cluster with `kubectl`.

1. Verify Ansible can view your config by running `task ansible:list`

2. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

3. Run the k3s install playbook by running `task ansible:playbook:k3s-install`

4. If everything goes as planned you should see Ansible running the k3s install Playbook against your nodes.

5. Verify the nodes are online

```sh
kubectl --kubeconfig=./provision/kubeconfig get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-0          Ready    control-plane,master      4d20h   v1.21.5+k3s1
# k8s-1          Ready    worker                    4d20h   v1.21.5+k3s1
```

## 5.6. :small_blue_diamond:&nbsp; GitOps with Flux

:round_pushpin: Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

```sh
flux --kubeconfig=./provision/kubeconfig check --pre
# ► checking prerequisites
# ✔ kubectl 1.21.5 >=1.18.0-0
# ✔ Kubernetes 1.21.5+k3s1 >=1.16.0-0
# ✔ prerequisites checks passed
```

2. Pre-create the `flux-system` namespace

```sh
kubectl --kubeconfig=./provision/kubeconfig create namespace flux-system --dry-run=client -o yaml | kubectl --kubeconfig=./provision/kubeconfig apply -f -
```

3. Add the Age key in-order for Flux to decrypt SOPS secrets

```sh
cat ~/.config/sops/age/keys.txt |
    kubectl --kubeconfig=./provision/kubeconfig \
    -n flux-system create secret generic sops-age \
    --from-file=age.agekey=/dev/stdin
```

:round_pushpin: Variables defined in `./cluster/base/cluster-secrets.sops.yaml` and `./cluster/base/cluster-settings.yaml` will be usable anywhere in your YAML manifests under `./cluster`

4. **Verify** the `./cluster/base/cluster-secrets.sops.yaml` and `./cluster/core/cert-manager/secret.sops.yaml` files are **encrypted** with SOPS

5. If you verified all the secrets are encrypted, you can delete the `tmpl` directory now

6.  Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

7. Install Flux

:round_pushpin: Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.

```sh
kubectl --kubeconfig=./provision/kubeconfig apply --kustomize=./cluster/base/flux-system
# namespace/flux-system configured
# customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
# ...
# unable to recognize "./cluster/base/flux-system": no matches for kind "Kustomization" in version "kustomize.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "GitRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
```

8. Verify Flux components are running in the cluster

```sh
kubectl --kubeconfig=./provision/kubeconfig get pods -n flux-system
# NAME                                       READY   STATUS    RESTARTS   AGE
# helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
# kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
# notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
# source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
```

:tada: **Congratulations** you have a Kubernetes cluster managed by Flux, your Git repository is driving the state of your cluster.

## 5.7. :cloud:&nbsp; Configure Cloudflare DNS with Terraform

:round_pushpin: Review the Terraform scripts under `./terraform/cloudflare/` and make sure you understand what it's doing (no really review it). If your domain already has existing DNS records be sure to export those DNS settings before you continue. Ideally you can update the terraform script to manage DNS for all records if you so choose to.

1. Pull in the Terraform deps by running `task terraform:init:cloudflare`

2. Review the changes Terraform will make to your Cloudflare domain by running `task terraform:plan:cloudflare`

3. Finally have Terraform execute the task by running `task terraform:apply:cloudflare`

If Terraform was ran successfully head over to your browser and you _should_ be able to access `https://hajimari.${BOOTSTRAP_CLOUDFLARE_DOMAIN}`

# 6. :mega:&nbsp; Post installation

## 6.1. :point_right:&nbsp; Troubleshooting

Our [wiki](https://github.com/k8s-at-home/template-cluster-k3s/wiki) is a good place to start troubleshooting issues. If that doesn't cover your issue, start a new thread in the #support channel on our [Discord](https://discord.gg/k8s-at-home).

## 6.2. :robot:&nbsp; Integrations

Our Check out our [wiki](https://github.com/k8s-at-home/template-cluster-k3s/wiki) for more integrations!

# 7. :grey_question:&nbsp; What's next

The world is your cluster, try installing another application or if you have a NAS and want storage back by that check out the helm charts for [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) or [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner).

If you plan on exposing your ingress to the world from your home. Checkout [our rough guide](https://docs.k8s-at-home.com/guides/dyndns/) to run a k8s `CronJob` to update DDNS.

# 8. :handshake:&nbsp; Thanks

Big shout out to all the authors and contributors to the projects that we are using in this repository.

# 9. :arrows_counterclockwise: Keeping this repo upto date with the template repo

At some point you may want to update your Git repository with some commit from this repository. The following is one method to achieve this.

1. Add this repository as an additional remote

    ```sh
    git remote add tmpl git@github.com:k8s-at-home/template-cluster-k3s.git
    ```

2. Fetch all the branches

    ```sh
    git fetch tmpl
    ```

3. List the commits from this repository

    ```sh
    git log tmpl/main
    ```

4. There are two methods to bring changes from template in here: 
   
   4.1. Pick the commit you want to bring over to your repository

    ```sh
    git cherry-pick ce67a3c
    ```

   4.2. Use difftool to compare latest with the current repo and add changes manually

   ```bash
   git difftool tmpl/main
   ```

   If a new file that was not present in my local main was added to the template, Meld (difftool) wouldn't be able to save that file. If this is the case, you'll get an error when trying to save. for these files find the file location with `git diff tmpl/main` and do `git checkout tmpl/main path/to/new/file` to get the new file. 


5. Push the changes up to your Git remote

    ```sh
    git push origin main
    ```


