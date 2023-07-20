## Overview

This tutorial demonstrates a platform engineering approach to IBM MQ
development and deployment. It demonstrates continuous integration, continuous
deployment, GitOps, Infrastructure as Code and DevOps using containers,
Kubernetes and a set of popular cloud native tools such as ArgoCD and Tekton.

In this tutorial, you will:

1. Create a Kubernetes cluster and image registry, if required.
2. Create an operational repository to store MQ resources that are
   deployed to the Kubernetes cluster.
3. Install ArgoCD to manage the continuous deployment of MQ-related
   resources to the cluster.
4. Create a source Git repository that holds the MQ development artifacts
   for a queue manager.
5. Install Tekton to provide continuous integration of the source MQ
   artifacts. These pipeline ensures that all changes to these artifacts are
   successful built, packaged, versioned and tested before they are delivered
   into the operational repository, read for deployment.
6. Gain experience with the IBM-supplied MQ operator and container.

By the end of the tutorial, you will have practical experience and knowledge of
platform engineering with MQ in a Kubernetes environment.

---

## Introduction

The following diagram shows a CICD pipeline for MQ:

![diagram1](./xdocs/images/diagram1.drawio.png)

Notice:

- The git repository `mq01-src` holds the source development artifacts for a
  queue manager `mq01`.
- A Tekton pipeline uses the `mq01-src` repository to build, package, test,
  version and deliver resources that define the `mq01` MQ appliance.
- If the pipeline is successful, then the YAMLs that define `mq01` are stored in
  the operational repository `mq01-ops` and the container image for `mq01` is
  stored in an image registry.
- Shortly after the changes are committed to the git repository, an ArgoCD
  application detects the updated YAMLs. It applies them to the cluster to create or
  update a running `mq01` queue manager.


This tutorial will walk you through the process of setting up this configuration:
- Step 1: Follow [these instructions this
  repository](https://github.com/mq-modernization-demo/mq01-ops#readme) to set up your
  cluster, ArgoCD and the `mq01-ops` repository. When complete, you will return
  to this repository to complete step 2.
- Step 2: Continue with the instructions in this README to create the `mq01-src`
  repository, run a tekton pipeline to populate the `mq01-ops` repository and
  interact with the new or updated queue manager`mq01`.

---

## Recreating Environment Variables

If you're doing this tutorial over an extended period, it may result in you
using a different terminal session. In this case, don't forget to re-define the
following environment variables:

```bash
export GITUSER=<GitHub user name>
export GITORG=mqorg-$GITUSER
export GITTOKEN=<PAT copied from GitHub>
export GITCONFIG=$(printf "[credential \"https://github.com\"]\n  helper = store")
```

---

## Creating the `mq01-src` repository

We use this [template repository](https://github.com/mq-modernization-demo/mq01-src) to create
`mq01-src` in our new organization. Forking a template creates a repository with
a clean git history, allowing us to track the history of changes to our queue manager `mq01` every time we update `mq01-src`.

> **Note**<br>
>
> Might need to add screenshot here

<br> Click on [this URL](https://github.com/mq-modernization-demo/mq01-src/generate) to fork
from the `mq01-src` template repository:

<img src="./xdocs/images/diagram2.png" alt="drawing" width="800"/>

This screen allows you to define the properties for you copy of the `mq01-src`
repository.

Specifically:

* In the `Owner` dropdown, select your recently created organization, e.g. `mqorg-xxxxx`
* In the `Repository name` field, specify `mq01-src`.
* In the `Description` field, specify `Source repository for mq01 queue manager`.
* Select `Public` for the repository visibility.
* Select `Include all branches`.

<br> Click on `Create repository from template`:

<br> This repository will be cloned to the specified GitHub account:
<img src="./xdocs/images/diagram3.png" alt="drawing" width="800"/>

<br> You have successfully created a copy of the `mq01-src` repository in your
organization.

---

## Clone repository to your local machine

We're going to use the contents of the `mq01-src` repository to build the `mq01`
queue manager. First we need to clone this repository to our local machine.

In the terminal window, issue the following command:

```bash
cd $HOME/git/$GITORG-tutorial
git clone https://$GITTOKEN@github.com/$GITORG/mq01-src.git
cd mq01-src
```

which will clone the repository to your local machine.

```bash
Cloning into 'mq01-src'...
remote: Enumerating objects: 69, done.
remote: Counting objects: 100% (69/69), done.
remote: Compressing objects: 100% (58/58), done.
remote: Total 69 (delta 4), reused 52 (delta 2), pack-reused 0
Receiving objects: 100% (69/69), 390.75 KiB | 1.01 MiB/s, done.
Resolving deltas: 100% (4/4), done.
```

---

## The source repository branches

The `mq01-src` repository holds both the development artifacts for the MQ
queue manager `mq01` as well as the Tekton pipeline definition to build it.

Issue the following commands to see these branches

```bash
git branch -r
```

which shows two branches (`HEAD` refers to the current branch):

```bash
origin/HEAD -> origin/main
origin/main
origin/pipelines
```

The source artifacts for `mq01` are held in the `main` branch. The `pipelines`
branch holds the Tekton pipeline definitions used to build, test and version
these artifacts.

---

### The `main` branch

The `main` branch folder structure of the `mq01-src` repository reflects the
structure of the `mq01` queue manager.

<br> There are three elements of the `mq01` virtual queue manageras shown
in the following diagram:

<img src="./xdocs/images/diagram4.drawio.png" alt="drawing" width="800"/>

Note
* `mq01` has a set of multi-protocol gateway definitions that are developed by a
  user, each of which is specified in a gateway `local` folder.
* Each gateway has a user specified configurations that is specified in a gateway
  `.cfg` file.
* `mq01` runs these gateways and configuration in a container that uses an image
  based on binary code provided by IBM. We use a `Dockerfile` to identify the
  precise version of this binary code.

Let's explore the repository to see this structure.

```bash
tree -L 1
```

```bash
.
├── LICENSE
├── README.md
├── bin
├── config
├── user
└── xdocs
```

Notice the simplicity of this structure: a `LICENSE` file, this `README` and
associated documentation in `xdocs`, together with three other folders; let's
explore these `bin` and `config` and `user` folders a little more deeply.

Issue the following command:

```bash
tree bin config user
```

which will show the structure of these folders:

```bash
bin
└── Dockerfile
config
├── qm.ini
├── scripts
│   ├── kustomization.yaml
│   ├── start-mqsc.sh
│   └── stop-mqsc.sh
└── yamls
    ├── kustomization.yaml
    └── qmgr.yaml
user
└── mqsc
    └── qmgr.mqsc
```

Note:
* It makes sense for each queue manager to map to a container. It provides a
  natural unit for isolation, management and scaling. The `Dockerfile` in the
  `bin` folder contains the exact version MQ image being used by `mq01`.
* `config` folder
* `user` folder

---

## Work on pipelines

```bash
cd mq01-src
git checkout pipelines
```

---

## Login to cluster

```bash
oc login
```

---

## Locate MQ pipeline source

```bash
cd $HOME/git/$GITORG-tutorial/mq01-src/pipelines/dev-build
ls
```

---

## Update ingress YAML

The ingress used by the `mq01` appliance needs to be customized for the cluster.
It would be better to do this customization at deployment time, as this is the
point where we know which cluster to which we're going to deploy. To simplify
the tutorial we do it now.

The YAML is sourced from the `mq01-src` repository and looks like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mq01
  namespace: mq01-dev
spec:
  rules:
    - host: mq01-ibm-mq.mq01-dev.<cluster sub-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mq01-ibm-mq
                port:
                  number: 1414
```

Issue the following command to locate the ingress YAML used by your `mq01` appliance:

```bash
echo https://github.com/$GITORG/mq01-src/blob/main/config/yamls/mq01-ingress.yaml
```

It will return a URL, for example:

```bash
https://github.com/mqorg-odowdaibm/mq01-src/blob/main/config/yamls/mq01-ingress.yaml
```

Copy this URL into your favorite browser.

Edit the ingress YAML file, replacing the `<cluster sub-domain>` with the output from the following command:

```bash
oc get ingresscontrollers/default -n openshift-ingress-operator -o jsonpath='{.status.domain}'
```

which will look something like this:

```bash
mq-cluster-1-d02cf90349a0fe46c9804e3ab1fe2643-0000.eu-gb.containers.appdomain.cloud
```

Commit this change to GitHub; this Ingress file will now be used for your `mq01` queue manager.

---

## Create cluster pipeline resources

```bash
oc apply -f mq-clone.yaml
oc apply -f mq-gen-yamls.yaml
oc apply -f mq-push.yaml
oc apply -f mq-dev-pipeline.yaml
oc apply -f mq-store-yamls.yaml
oc apply -f mq-build-image.yaml
oc apply -f mq-test.yaml
```

---

## Customize pipeline

```bash
envsubst <mq-dev-pipelinerun.yaml > mq-dev-pipelinerun.yaml
```


## Run pipeline

```bash
oc create -f mq-dev-pipelinerun.yaml
```

In the following command replace `xxxxx` with the new pipeline run identifier:

```bash
tkn pipelinerun logs ,mq-dev-pipeline-run-xxxxx -n mq01-dev -f
```

## View pipelinerun in the web console

Show progress in `tkn` command line tool?

## Try out service using API tool

Now that the `mq01` queue manager is running the BookingService, we can test it.

We are going to use the MQ REST API.

You can use any API test tool; [RESTER](https://addons.mozilla.org/en-GB/firefox/addon/rester/) is good if you use Firefox.


## Congratulations

Well done! You have successfully completed this tutorial. You have

* Configured a Kubernetes cluster for Platform Engineering including CICD and GitOps.
* Built and deployed a queue manager, deployed it, and interacted with it.

